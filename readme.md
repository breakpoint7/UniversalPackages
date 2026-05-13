# Azure DevOps Artifacts Guide

A practical guide for developers who need to store and reuse binary/support files across build pipelines using **Azure DevOps Artifacts**, with a focus on **Universal Packages**.

---

## 1. Package Types in Azure DevOps Artifacts

Azure DevOps Artifacts is a package hosting service that supports several package protocols. Each feed in Azure Artifacts can host one or more of these types.

| Package Type      | Best For                                                                 | Typical Content                          | Client Tooling                       | Versioning Scheme            |
| ----------------- | ------------------------------------------------------------------------ | ---------------------------------------- | ------------------------------------ | --------------------------- |
| **NuGet**         | .NET libraries and tools                                                 | `.nupkg` (DLLs, content, MSBuild props)  | `nuget.exe`, `dotnet`, Visual Studio | SemVer (1.2.3, 1.2.3-beta)  |
| **npm**           | Node.js / JavaScript / TypeScript libraries                              | `.tgz` (JS modules)                      | `npm`, `yarn`, `pnpm`                | SemVer                      |
| **Maven**         | Java / JVM artifacts                                                     | `.jar`, `.war`, `.pom`                   | `mvn`, `gradle`                      | Maven version (1.0.0, -SNAPSHOT) |
| **Python (PyPI)** | Python wheels and source dists                                           | `.whl`, `.tar.gz`                        | `pip`, `twine`                       | PEP 440                     |
| **Cargo**         | Rust crates                                                              | `.crate`                                 | `cargo`                              | SemVer                      |
| **Universal**     | **Anything else** — arbitrary files, binaries, tools, assets, large blobs | Any directory of files (any size/format) | Azure CLI (`az artifacts universal`) | SemVer (X.Y.Z, required)    |

### When to choose Universal Packages

Choose Universal when:

- The content is **not a language-specific library** (e.g., firmware, ISO files, signed binaries, fonts, ML models, test fixtures, SDK zips, native toolchains).
- You want to package **a whole directory** of mixed assets and pull them down as a unit on a build agent.
- The files can be large (Universal supports up to **4 TiB per package**, with deduplication on upload).
- You need a single, simple distribution mechanism that works on Windows, Linux, and macOS agents via the Azure CLI.

If the content *is* a typed library (NuGet, npm, Maven, PyPI, Cargo), prefer the native feed type — you get richer tooling, restore semantics, and IDE integration.

---

## 2. Prerequisites

Before you start, make sure you have:

1. An **Azure DevOps organization** and **project** (e.g., `https://dev.azure.com/<org>/<project>`).
2. An **Azure Artifacts feed** created in that project — see §3 below.
3. The **Azure CLI** installed: <https://learn.microsoft.com/cli/azure/install-azure-cli>.
4. The **azure-devops** CLI extension:
   ```powershell
   az extension add --name azure-devops
   ```
5. Signed in to Azure DevOps:
   ```powershell
   az login
   az devops configure --defaults organization=https://dev.azure.com/<org> project=<project>
   ```
   For non-interactive (pipeline) scenarios, use a PAT:
   ```powershell
   $env:AZURE_DEVOPS_EXT_PAT = "<your-pat>"
   ```
   The PAT needs **Packaging: Read & write** scope.

---

## 3. Create an Azure Artifacts Feed

A **feed** is the container that holds your packages and controls who can read/publish them. You need one before you can publish anything.

### 3.1 Create via the web UI (easiest)

1. Browse to `https://dev.azure.com/<org>/<project>`.
2. Open **Artifacts** in the left nav.
3. Click **+ Create Feed**.
4. Fill in:
   - **Name** — e.g., `BuildTools`. Lowercase, no spaces recommended.
   - **Visibility** — who can *see* the feed:
     - *People in my organization* (typical for internal feeds).
     - *Specific people* (locked-down, you grant access explicitly).
   - **Scope** — **Project** vs **Organization**:
     - **Project-scoped** (recommended default): lives under one project, simpler permissions, supports project-level PATs. Use when the feed serves one team/project. CLI calls must pass `--project <proj> --scope project`.
     - **Organization-scoped**: shared across all projects in the org. Use only when multiple projects genuinely need the same feed. CLI calls omit `--project` and pass `--scope organization`.
   - **Include packages from common public sources** (the upstream sources checkbox) — see §3.3.
5. Click **Create**.

### 3.2 Create via the CLI

```powershell
az artifacts feed create `
  --organization "https://dev.azure.com/<org>" `
  --project "<project>" `
  --name "BuildTools"
```

Omit `--project` to create an **organization-scoped** feed. By default this creates a project-scoped feed with the standard `@Local`, `@Prerelease`, `@Release` views.

### 3.3 Upstream sources — should you enable them?

**Upstream sources** let your feed transparently proxy packages from public registries (nuget.org, npmjs.com, PyPI, Maven Central, etc.). When a client asks the feed for a package it doesn't have, the feed fetches it from the upstream, **caches it in your feed**, and serves it back. Subsequent requests are served from the cache.

**Enable upstreams (recommended for most teams) when:**

- Your projects consume any public NuGet / npm / PyPI / Maven / Cargo packages — the cache gives you **build reproducibility** even if a public package is yanked/deleted (e.g., the classic `left-pad` scenario).
- You want a **single feed URL** for developers and CI — they point at your feed and get both your private packages and public ones.
- You want a **supply-chain checkpoint**: every public package that enters your org gets recorded in the feed, with audit history and the ability to deprecate/remove specific versions.
- You operate behind a restricted network and need a controlled egress point for public packages.

**Leave upstreams off when:**

- The feed will hold **only Universal Packages** or other purely internal artifacts (there is no public Universal Packages registry to proxy, so upstreams add nothing useful for a Universal-only feed).
- Org policy requires that public packages flow through a **separate, dedicated** ingestion feed that you then promote into release feeds — in which case enable upstreams on the ingestion feed only.
- You explicitly want to forbid public packages from being pulled into the feed.

**Practical recommendation:** if your feed will mix Universal Packages with any language-typed packages (NuGet/npm/etc.), **enable upstream sources** and accept the default public sources. You can always add or remove individual upstreams later under Feed Settings → **Upstream sources**. If the feed is *purely* Universal Packages, leave upstreams off — you can enable them later if the feed's purpose expands.

> Once a public package version has been pulled through an upstream and cached, **that exact version stays in your feed even if it's removed upstream** — this is the main reproducibility benefit.

### 3.4 Set feed permissions

After creating, open **Feed Settings → Permissions** and verify the roles:

| Role            | Who should have it                                          |
| --------------- | ----------------------------------------------------------- |
| **Owner**       | Feed admins (small group).                                  |
| **Contributor** | People/services that publish packages, including pipelines. |
| **Collaborator**| Can save packages from upstreams but not publish their own. |
| **Reader**      | Everyone who consumes packages.                             |

For pipelines, add **Project Collection Build Service (<org>)** as **Contributor** (publish) or **Reader** (download-only).

---

## 4. Create a Universal Package from a Directory

### 4.1 Lay out your source directory

Put everything you want to ship into a single folder. The folder's contents (recursively) become the package payload — the top-level folder name itself is **not** included.

Example:

```
C:\src\my-tools\
├── bin\
│   ├── tool.exe
│   └── helper.dll
├── config\
│   └── default.json
└── README.txt
```

### 4.2 Choose a name and version

- **Name**: lowercase, `a–z 0–9 - _ .`, max 255 chars (e.g., `build-tools-win-x64`).
- **Version**: must be **SemVer 2.0** — `MAJOR.MINOR.PATCH` with optional `-prerelease` (e.g., `1.0.0`, `1.2.3-beta.1`).

### 4.3 Publish

```powershell
az artifacts universal publish `
  --organization "https://dev.azure.com/<org>" `
  --project "<project>" `
  --scope project `
  --feed "<feed-name>" `
  --name "build-tools-win-x64" `
  --version "1.0.0" `
  --description "Win x64 build helper binaries" `
  --path "C:\src\my-tools"
```

Notes:

- Use `--scope organization` (and omit `--project`) if your feed is **organization-scoped**.
- Upload is **chunked and deduplicated** — re-publishing similar content is fast.
- A version **cannot be overwritten**. To replace, publish a new version (see §7).

---

## 5. Download / Consume a Universal Package

### 5.1 Manually on any machine (build agent, dev box)

```powershell
az artifacts universal download `
  --organization "https://dev.azure.com/<org>" `
  --project "<project>" `
  --scope project `
  --feed "<feed-name>" `
  --name "build-tools-win-x64" `
  --version "1.0.0" `
  --path "C:\agent\_work\_tools\build-tools"
```

Tips:

- Use `--version "*"` to get the **latest** version, or a SemVer range like `"1.x"` / `"^1.2.0"`.
- The target `--path` is created if it doesn't exist; files land directly inside it (no wrapper folder).

### 5.2 In an Azure Pipeline (YAML)

Use the built-in **UniversalPackages** task — no Azure CLI install needed.

**Download in a pipeline:**

```yaml
steps:
- task: UniversalPackages@0
  displayName: 'Download build-tools universal package'
  inputs:
    command: download
    downloadDirectory: '$(Pipeline.Workspace)/tools'
    feedsToUse: 'internal'
    vstsFeed: '<project>/<feed-name>'      # or just '<feed-name>' for org-scoped
    vstsFeedPackage: 'build-tools-win-x64'
    vstsPackageVersion: '1.0.0'             # or '*' for latest

- script: |
    "$(Pipeline.Workspace)\tools\bin\tool.exe" --version
  displayName: 'Use the tool'
```

**Publish from a pipeline** (e.g., after a build step produces artifacts in `$(Build.ArtifactStagingDirectory)`):

```yaml
- task: UniversalPackages@0
  displayName: 'Publish build-tools universal package'
  inputs:
    command: publish
    publishDirectory: '$(Build.ArtifactStagingDirectory)'
    feedsToUsePublish: 'internal'
    vstsFeedPublish: '<project>/<feed-name>'
    vstsFeedPackagePublish: 'build-tools-win-x64'
    versionOption: 'patch'                  # auto-increment: major | minor | patch | custom
    # versionPublish: '1.2.3'              # only when versionOption: custom
    packagePublishDescription: 'CI build $(Build.BuildNumber)'
```

`versionOption` of `major`/`minor`/`patch` will look at the latest version in the feed and bump it automatically — ideal for CI.

### 5.3 In a Classic pipeline

Use the **Universal packages** task from the task catalog with the same inputs as above.

---

## 6. Authenticating Build Agents (Self-Hosted)

- **Microsoft-hosted agents**: auth is automatic via the pipeline's `System.AccessToken` — nothing to configure.
- **Self-hosted agents**: the agent's service account must have **Reader** (download) or **Contributor** (publish) on the feed. Grant via Feed Settings → **Permissions**.
- **Manual download outside a pipeline**: either `az login` interactively, or set `AZURE_DEVOPS_EXT_PAT` to a PAT with **Packaging (read)** for download, **Packaging (read & write)** for publish.

---

## 7. Updating and Versioning Packages

Universal Package versions are **immutable** — you cannot republish the same `name@version` with different content. To ship a change, publish a **new version**.

### 7.1 Pick a SemVer bump

| Change in package contents                              | Bump   | Example          |
| ------------------------------------------------------- | ------ | ---------------- |
| Bug fix / refreshed binaries, no consumer impact        | PATCH  | 1.0.0 → 1.0.1    |
| New files added, backward compatible                    | MINOR  | 1.0.1 → 1.1.0    |
| Removed/renamed files, breaking changes for consumers   | MAJOR  | 1.1.0 → 2.0.0    |
| Pre-release / testing                                   | suffix | 2.0.0-beta.1     |

### 7.2 Recommended update workflow

1. Update the files in your source directory.
2. Decide the new version (manually, or let the pipeline auto-bump with `versionOption: patch`).
3. Re-run the publish command/task from §4 or §5.2.
4. Update consuming pipelines to reference the new version, or pin to a range (`1.x`, `*`) if you want automatic uptake.

### 7.3 Pinning strategy for consumers

- **Pin exact** (`1.2.3`) for reproducible builds — recommended for release pipelines.
- **Pin range** (`1.x`, `^1.2.0`) for dev/CI pipelines that should pick up patches automatically.
- **Use `*`** only when you genuinely always want latest (rare; risks unexpected breakage).

### 7.4 Promoting through views (optional but recommended)

Each Azure Artifacts feed has **views** (commonly `@Local`, `@Prerelease`, `@Release`). Promote a version to a view to signal quality:

```powershell
# Promote 1.2.0 to the @Release view
az artifacts universal promote `
  --organization "https://dev.azure.com/<org>" `
  --project "<project>" `
  --feed "<feed-name>" `
  --name "build-tools-win-x64" `
  --version "1.2.0" `
  --view "Release"
```

Consumers can then restrict themselves to `@Release` quality versions via the feed view URL.

### 7.5 Deprecating / removing versions

- **Deprecate**: mark a version as deprecated in the Azure DevOps UI (Artifacts → package → version → ⋯) to discourage use without breaking existing consumers.
- **Unlist / Delete**: also available via the UI or `az artifacts universal` — but prefer deprecating over deleting because deletion breaks pinned consumers.

---

## 8. End-to-End Example

Scenario: ship a folder of Windows signing tools as `signing-tools-win` and consume them in a release pipeline.

**1. Initial publish (one-time, from a dev box):**

```powershell
az artifacts universal publish `
  --organization https://dev.azure.com/contoso `
  --project Platform `
  --scope project `
  --feed BuildTools `
  --name signing-tools-win `
  --version 1.0.0 `
  --description "Code signing tool set" `
  --path C:\src\signing-tools
```

**2. Consume in a release pipeline (`azure-pipelines.yml`):**

```yaml
pool:
  vmImage: windows-latest

steps:
- task: UniversalPackages@0
  inputs:
    command: download
    downloadDirectory: '$(Pipeline.Workspace)\signing'
    feedsToUse: internal
    vstsFeed: 'Platform/BuildTools'
    vstsFeedPackage: 'signing-tools-win'
    vstsPackageVersion: '1.x'

- script: '"$(Pipeline.Workspace)\signing\signtool.exe" sign /fd SHA256 $(Build.ArtifactStagingDirectory)\*.exe'
  displayName: 'Sign binaries'
```

**3. Ship an updated tool set:**

```powershell
# After updating files in C:\src\signing-tools
az artifacts universal publish `
  --organization https://dev.azure.com/contoso `
  --project Platform `
  --scope project `
  --feed BuildTools `
  --name signing-tools-win `
  --version 1.1.0 `
  --path C:\src\signing-tools
```

Because the consumer pinned `1.x`, the next pipeline run automatically picks up `1.1.0`.

---

## 9. Troubleshooting Cheatsheet

| Symptom                                              | Likely cause / fix                                                                                  |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `TF400898` / 401 Unauthorized on publish             | PAT missing **Packaging: read & write**, or feed scope mismatch (`--scope project` vs `organization`). |
| "Version already exists"                             | Versions are immutable — bump SemVer and republish.                                                 |
| `az: 'artifacts' is not in the 'az' command group`   | Run `az extension add --name azure-devops`.                                                         |
| Pipeline can't find feed                             | Use full `<project>/<feed>` form for project-scoped feeds.                                          |
| Self-hosted agent can't download                     | Add agent's identity (or **Project Collection Build Service**) as **Reader** on the feed.           |
| Slow first upload, fast re-uploads                   | Expected — Universal Packages deduplicate chunks across versions.                                   |

---

## 10. Quick Reference

**Publish:**
```powershell
az artifacts universal publish --organization <url> --project <proj> --scope project --feed <feed> --name <name> --version <semver> --path <dir>
```

**Download:**
```powershell
az artifacts universal download --organization <url> --project <proj> --scope project --feed <feed> --name <name> --version <semver|range|*> --path <dir>
```

**Promote:**
```powershell
az artifacts universal promote --organization <url> --project <proj> --feed <feed> --name <name> --version <semver> --view <view>
```

**Docs:**
- Universal Packages overview: <https://learn.microsoft.com/azure/devops/artifacts/quickstarts/universal-packages>
- UniversalPackages pipeline task: <https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/universal-packages-v0>
- Azure Artifacts feeds & views: <https://learn.microsoft.com/azure/devops/artifacts/concepts/feeds>
