# Building OEM Packs

This document walks through the end-to-end process of building a vendor payload archive — from cloning the repository to a validated `<Vendor>.7z` ready to drop into a deployment share.

The workflow is the same for every vendor. What changes is the set of apps, the customization assets, and the sources each app is fetched from. Dell is the reference vendor with the most complete recipe.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Before Your First Build](#before-your-first-build)
- [Building One Vendor](#building-one-vendor)
- [Building Multiple Vendors](#building-multiple-vendors)
- [Building All Vendors](#building-all-vendors)
- [What Happens During a Build](#what-happens-during-a-build)
- [Validating the Archive](#validating-the-archive)
- [Placing the Archive in Your Deployment Share](#placing-the-archive-in-your-deployment-share)
- [Rebuilding After Changes](#rebuilding-after-changes)
- [Manual Sources](#manual-sources)
- [Cache Management](#cache-management)
- [Advanced Options](#advanced-options)
- [What Good Looks Like](#what-good-looks-like)
- [Troubleshooting](#troubleshooting)

---

## Overview

A build does four things:

1. **Reads the recipe.** `vendors\<Vendor>\Recipe.json` declares what to fetch, from where, at what version.
2. **Fetches missing content.** Downloads installers into `cache\` (or reuses cached copies). Sources are `winget`, `direct-url`, `repo-asset`, or `manual`.
3. **Stages and packs.** Assembles the extracted content into a staging folder that mirrors the archive's internal structure, then packs it into `<Vendor>.7z`.
4. **Validates.** Checks the archive against the framework manifest in your deployment share to ensure every app the framework expects is present at the expected path.

The output is a single `<Vendor>.7z` file. Place it at `\\SERVER\Shared\OEM\x64\<Vendor>.7z` (and `\x86\` if you support 32-bit hardware). The deployment framework picks it up automatically.

---

## Prerequisites

Before your first build, ensure the following are installed and available on your build host:

| Requirement | Minimum | How to verify |
|---|---|---|
| Windows | Windows 10 1809+ or Windows 11 | `winver` |
| PowerShell | 5.1 or 7 | `$PSVersionTable.PSVersion` |
| 7-Zip | Any recent version | `Test-Path "C:\Program Files\7-Zip\7z.exe"` |
| winget (App Installer) | Latest from Microsoft Store | `winget --version` |
| Disk space | 10 GB free for cache, plus room for staging | `Get-PSDrive C` |
| Network access | Vendor CDNs and Microsoft winget endpoints | `Test-NetConnection download.microsoft.com -Port 443` |

You do **not** need administrator rights on the build host. The tool writes only to the repository folder, the cache folder, and the output path you specify.

You **do** need write access to the output path. If you are building directly into a deployment share over SMB, verify you can create and delete files there:

```powershell
New-Item "\\SERVER\Shared\OEM\x64\_write-test.txt" -Force
Remove-Item "\\SERVER\Shared\OEM\x64\_write-test.txt" -Force
```

---

## Before Your First Build

### 1. Clone the repository

```bash
git clone https://github.com/ArthurJDurand/MDT-OEM-Extensibility.git C:\Source\MDT-OEM-Extensibility
cd C:\Source\MDT-OEM-Extensibility
```

### 2. Confirm the framework manifest is reachable

The tool validates the built archive against the framework's manifest, which lives in your deployment share:

```
\\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\<Vendor>.json
```

If this path is different in your environment, pass `-ManifestPath` on every build. The tool cannot guess.

```powershell
Test-Path "\\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json"
```

If this returns `False`, either your deployment share is not populated, or the manifest lives elsewhere. See the main repository's [`docs/APPS-FRAMEWORK.md`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment/blob/main/docs/APPS-FRAMEWORK.md) for the expected layout.

### 3. Confirm the vendor has a recipe

```powershell
Get-ChildItem vendors -Directory | Select-Object Name
```

Eleven vendors should appear. Not all will have a complete `Recipe.json` today. Dell is complete; the rest are scaffolds. If you are building a scaffold vendor, the build will report missing sources.

### 4. Confirm `winget` works

```powershell
winget source list
```

You should see `winget` and `msstore` as configured sources. If `winget` is missing or broken, the build will skip all `winget` sources and fail on those apps.

### 5. Do a dry run on Dell

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -DryRun
```

A dry run resolves every source, reports what would be downloaded, and exits without downloading anything. This is the fastest way to verify your environment before committing to a real build.

---

## Building One Vendor

The standard invocation:

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath \\SERVER\Shared\OEM\x64
```

What this does:

1. Loads `vendors\Dell\Recipe.json`
2. Loads the framework manifest from `\\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json`
3. Validates that every app in the manifest has a matching recipe entry
4. For each app:
   - If already in `cache\` and hash matches, skip the download
   - If `winget` source, invoke `Resolve-WingetPackage.ps1`
   - If `direct-url` source, invoke `Resolve-OEMUrl.ps1`
   - If `repo-asset` source, copy from `vendors\Dell\Assets\`
   - If `manual` source, skip and record in the summary
5. Assemble a staging folder that mirrors the archive structure
6. Pack the staging folder into `Dell.7z` at the output path
7. Validate the archive against the framework manifest

Expect **10 minutes to 2 hours** for a first build of Dell, depending on cache state, network speed, and how many apps are manual.

### Console output

The tool prints progress as it works:

```
[2026-01-15 14:32:18] [INFO]    Starting build for vendor: Dell
[2026-01-15 14:32:18] [INFO]    Loading recipe: vendors\Dell\Recipe.json
[2026-01-15 14:32:18] [INFO]    Loading manifest: \\SERVER\DeploymentShare$\...\Dell.json
[2026-01-15 14:32:19] [SUCCESS] Manifest and recipe are in sync (14 apps)
[2026-01-15 14:32:19] [INFO]    Cache: C:\Source\MDT-OEM-Extensibility\cache
[2026-01-15 14:32:19] [INFO]    Staging: C:\Users\me\AppData\Local\Temp\MDT-OEM-Dell-abc123
[2026-01-15 14:32:20] [INFO]    [ 1/14] Microsoft.WindowsAppRuntime.1.8 (winget)
[2026-01-15 14:32:21] [INFO]      Cache hit: a1b2c3d4e5f6...
[2026-01-15 14:32:21] [SUCCESS] [ 1/14] Staged
[2026-01-15 14:32:21] [INFO]    [ 2/14] Microsoft .NET Windows Desktop Runtime 8 (winget)
[2026-01-15 14:32:22] [INFO]      Cache miss — downloading
...
```

### Console summary

At the end:

```
═══════════════════════════════════════════════════════════════════
 BUILD SUMMARY: Dell
═══════════════════════════════════════════════════════════════════
  Total apps        : 14
  Downloaded        : 3
  Cached (skipped)  : 10
  Manual (pending)  : 1
  Failed            : 0
  Staging folder    : (deleted)
  Archive           : \\SERVER\Shared\OEM\x64\Dell.7z
  Size              : 3.42 GB
  Duration          : 8m 12s
  SHA-256           : a1b2c3d4e5f6...
═══════════════════════════════════════════════════════════════════
```

If any app failed, the summary lists it with a reason. If any app is `manual`, the summary prints the instructions the recipe provides for obtaining it.

---

## Building Multiple Vendors

Pass a list of vendors:

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell,HP,Lenovo -OutputPath \\SERVER\Shared\OEM\x64
```

Each vendor is built independently. If one fails, the others continue. The tool prints a combined summary at the end.

This is useful for a maintenance window: rebuild the three vendors you actually deploy, ignore the rest.

---

## Building All Vendors

```powershell
.\tools\Build-OEMPack.ps1 -All -OutputPath \\SERVER\Shared\OEM\x64
```

Every vendor with a complete recipe is built in sequence. Vendors with missing apps or manual-only coverage are reported at the end. Expect a long runtime — the full set of eleven vendors, once complete, will fetch on the order of 40–60 GB.

Do not run `-All` on the first build. Build Dell first, verify it works end-to-end, then add vendors.

---

## What Happens During a Build

A step-by-step trace for the curious. Skip this section unless something is failing.

### 1. Recipe load and validation

The tool reads `vendors\<Vendor>\Recipe.json`, validates it against `schema\vendor-recipe.schema.json`, and loads the framework manifest from the deployment share.

If the schema validation fails, the build stops with a list of missing or malformed fields. See [`docs/RECIPE-SCHEMA.md`](RECIPE-SCHEMA.md) for the field reference.

### 2. Manifest/recipe sync check

Every `AppName` in the framework manifest must have a matching entry in the recipe. If a manifest entry is missing, the build stops with:

```
[ERROR] Manifest and recipe are out of sync
[ERROR]   In manifest but not in recipe: <AppName>
```

Add the missing recipe entry (see [`docs/ADDING-A-VENDOR.md`](ADDING-A-VENDOR.md)) and rerun.

### 3. Per-app fetch

For each recipe entry, the tool determines the source type and calls the appropriate resolver:

| Source | Resolver | Behavior |
|---|---|---|
| `winget` | `Resolve-WingetPackage.ps1` | Runs `winget download` into a temp folder, picks the single new file, moves it to `cache\` |
| `direct-url` | `Resolve-OEMUrl.ps1` | Downloads the URL with retry and SHA-256 verification, moves it to `cache\` |
| `repo-asset` | (built-in) | Copies from `vendors\<Vendor>\Assets\` |
| `manual` | (skipped) | Records the app as pending; continues |

Cache lookup happens first. If `cache\<key>` exists and its hash matches the recipe, the resolver is not called.

### 4. Staging

Once every resolvable app is in `cache\`, the tool builds a staging folder that mirrors the archive structure:

```
<staging>\
├── Customizations.ps1
├── csup.txt
├── gpsFix.reg
├── OEMinfo.reg
├── unattend.xml
├── OEM.7z
├── Customizations\
│   ├── Dell.7z
│   └── G-series.7z
└── Apps\
    ├── CommandCenter\
    ├── CommandUpdate\
    └── ...
```

The layout mirrors the `InstallerPath` values in the framework manifest. Every file the deployment framework expects to find at a specific path ends up at that path inside the archive.

### 5. Packing

The tool invokes `New-OEMAppPack.ps1`, which calls 7-Zip with the correct compression settings:

```powershell
& "C:\Program Files\7-Zip\7z.exe" a -t7z -mx=5 -mmt=on -bsp1 "<output>.7z" "<staging>\*"
```

- `-t7z` — 7z format
- `-mx=5` — normal compression (a good balance of size and speed for installers, which are already compressed)
- `-mmt=on` — multi-threaded
- `-bsp1` — progress to stdout

For large archives, the tool splits into `.7z.001`, `.7z.002`, and so on if `-Volume` is passed.

### 6. Validation

The tool invokes `Test-OEMAppPack.ps1` to verify that the built archive contains every app the framework manifest expects, at the expected path. If validation fails, the archive is left in place but the build reports a failure.

### 7. Cleanup

The staging folder is deleted. The cache is left intact for the next build.

---

## Validating the Archive

Validation is automatic at the end of every build. To validate independently:

```powershell
.\tools\Test-OEMAppPack.ps1 `
    -Archive \\SERVER\Shared\OEM\x64\Dell.7z `
    -Manifest \\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json
```

The validator:

1. Lists the archive contents with 7-Zip.
2. Parses the framework manifest for every `InstallerPath`.
3. Confirms each path exists inside the archive.
4. Reports missing paths and unexpected extras.

Exit code `0` means the archive is complete. Any other exit code indicates a mismatch.

To skip validation during a fast rebuild (not recommended for production):

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath \\SERVER\Shared\OEM\x64 -SkipValidation
```

---

## Placing the Archive in Your Deployment Share

If you built directly into the deployment share's OEM folder (as in the examples above), you are done. The framework picks up the archive at the next deployment.

If you built to a staging location, copy the archive manually:

```powershell
Copy-Item "C:\Output\Dell.7z" "\\SERVER\Shared\OEM\x64\Dell.7z" -Force
```

If you support 32-bit deployments, also build and copy the x86 variant:

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -Architecture x86 -OutputPath \\SERVER\Shared\OEM\x86
```

The x86 build is a **work in progress**. Most x86 recipes are not yet complete. See the main repository's [Known Limitations](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment#known-limitations) for the current state.

### Verifying the deployment share has the archive

```powershell
Get-Item "\\SERVER\Shared\OEM\x64\Dell.7z" | Select-Object Name, Length, LastWriteTime
```

The `Length` should match the archive size reported in the build summary. The `LastWriteTime` should be recent.

### Offline media

If you build a DEPLOY USB for offline deployment, copy the archive to the USB after the build. See the main repository's [`docs/OFFLINE-MEDIA.md`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment/blob/main/docs/OFFLINE-MEDIA.md) for the USB workflow.

---

## Rebuilding After Changes

### Nothing changed

If no app versions have changed and the recipe has not been modified, a rebuild uses the cache for everything and skips downloads. It still repacks the archive, which takes a few minutes.

If you want to verify that nothing changed without repacking:

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -DryRun
```

### One app changed

The tool detects the change via the recipe's version or hash field, downloads only the changed app, and repacks. This is the common case — one vendor ships an update, you refresh just that app.

### The recipe changed

Any change to `vendors\<Vendor>\Recipe.json` triggers a full re-resolution of the affected entries. Unaffected entries still use the cache.

### Force a full re-download

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath \\SERVER\Shared\OEM\x64 -Force
```

`-Force` ignores the cache, re-downloads every app, and repacks. Use this when you suspect the cache is stale or corrupt.

### Purge the cache

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -PurgeCache
```

Deletes cached files that are not referenced by any current recipe. Useful when disk space is tight.

---

## Manual Sources

Some apps cannot be fetched automatically. The recipe marks them as `manual`. When a build encounters one, it records the app in the summary and continues — but the resulting archive is **incomplete**.

### What the build tells you

```
[WARN]  Manual source: DellInc.DellSupportAssistforPCs
[WARN]    Notes: MSIX bundle from the Microsoft Store. Download on a machine
[WARN]           with a work or school account, then place the .msix at
[WARN]           C:\Recovery\OEM\Apps\SupportAssist\UWP\SupportAssist_x64.msix
[WARN]           before building.
```

### How to satisfy a manual source

1. Obtain the file using the instructions in the `Notes` field.
2. Place the file at the target path inside the staging folder. The tool creates the staging folder at a temporary location and prints its path in the log.
3. Rerun the build with `-KeepStaging`:
   ```powershell
   .\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath \\SERVER\Shared\OEM\x64 -KeepStaging
   ```
4. After the build, remove the staging folder manually.

**Alternatively**, stage the file in the cache and mark it in the recipe as `repo-asset`. This is not recommended for large files or files that change frequently, because it commits a vendor binary to the repository.

**Long-term goal:** reduce the number of `manual` sources to zero by finding official download URLs. If you know of a reliable source for any of the current `manual` apps, open a PR.

---

## Cache Management

The cache lives at `cache\` under the repository root by default. It is gitignored.

### Cache layout

```
cache\
├── winget\
│   ├── <WingetId>\
│   │   └── <version>\
│   │       └── <installer file>
├── direct-url\
│   └── <sha256 prefix>\
│       └── <installer file>
└── repo-asset\
    └── <vendor>\
        └── <asset file>
```

The cache is content-addressed where possible. A `winget` entry is keyed by winget ID and version. A `direct-url` entry is keyed by SHA-256 prefix. A `repo-asset` entry is keyed by vendor and filename.

### Sizing the cache

Rough estimates for a complete eleven-vendor build:

| Vendor | Cache size (approximate) |
|---|---|
| Dell | 4 GB |
| HP | 2 GB |
| Lenovo | 2 GB |
| ASUS | 1.5 GB |
| Acer | 1 GB |
| MSI | 1.5 GB |
| Gigabyte | 500 MB |
| Dynabook | 300 MB |
| Huawei | 500 MB |
| Microsoft | 1 GB |
| Proline | 200 MB |
| **Total** | **14.5 GB** |

Add 30–50% for version history as vendors update. A cache can easily reach 25 GB if you rarely purge.

### Overriding the cache path

```powershell
.\tools\Build-OEMPack.ps1 -Vendor Dell -CachePath D:\Cache
```

Useful when the repository lives on a small SSD and the cache should live on a larger HDD or network share.

**Do not put the cache on a network share** unless you have no alternative. The latency of reading small files over SMB slows builds considerably.

### Clearing the cache

Full clear:

```powershell
Remove-Item cache\* -Recurse -Force
```

Selective clear (only entries not referenced by any recipe):

```powershell
.\tools\Build-OEMPack.ps1 -PurgeCache
```

---

## Advanced Options

| Flag | Purpose |
|---|---|
| `-Vendor <name>` | Vendor or comma-separated list of vendors to build |
| `-All` | Build every vendor with a complete recipe |
| `-OutputPath <path>` | Where to write the `.7z` archive. Default: current directory. |
| `-ManifestPath <path>` | Path to the framework manifest folder. Overrides the default. |
| `-CachePath <path>` | Where to store downloaded content. Default: `cache\` in the repository root. |
| `-SevenZipPath <path>` | Path to `7z.exe`. Default: `C:\Program Files\7-Zip\7z.exe`. |
| `-Architecture <arch>` | `x64` (default) or `x86`. Selects the recipe file to use. |
| `-Volume <size>` | Split the archive into parts of this size, e.g. `3g`. Required for FAT32 targets. |
| `-Force` | Ignore cache; re-download and repack everything. |
| `-DryRun` | Resolve and report without downloading or packing. |
| `-SkipValidation` | Skip the post-build manifest validation. |
| `-KeepStaging` | Do not delete the staging folder after the build. |
| `-PurgeCache` | Delete unreferenced cache entries and exit. |
| `-Verbose` | Print per-file detail. |
| `-LogPath <path>` | Write the log to this path. Default: `C:\ProgramData\MDT-OEM-Extensibility\Logs\<Vendor>.log`. |

For the full parameter list, run:

```powershell
Get-Help .\tools\Build-OEMPack.ps1 -Full
```

---

## What Good Looks Like

A successful build ends with:

- A `<Vendor>.7z` file at the specified output path
- A file size in the expected range (see the table in [Cache Management](#cache-management) for rough per-vendor sizes)
- A SHA-256 printed in the summary
- A `Failed: 0` count in the summary
- A `Manual: 0` count in the summary, or a known list of pending manual apps
- No error lines in the log

To double-check:

```powershell
& "C:\Program Files\7-Zip\7z.exe" t \\SERVER\Shared\OEM\x64\Dell.7z
```

This runs 7-Zip's archive integrity test. If it passes, the archive is readable.

To compare against the framework manifest one more time:

```powershell
.\tools\Test-OEMAppPack.ps1 `
    -Archive \\SERVER\Shared\OEM\x64\Dell.7z `
    -Manifest \\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json
```

Exit code `0` means you are done.

---

## Troubleshooting

The most common issues are covered in [`docs/TROUBLESHOOTING.md`](TROUBLESHOOTING.md). Quick pointers:

| Symptom | Likely Cause | Doc |
|---|---|---|
| Recipe not found | Wrong vendor name or missing file | [Build Tool Errors](TROUBLESHOOTING.md#build-tool-errors) |
| Manifest and recipe out of sync | Manifest was updated, recipe was not | [Build Tool Errors](TROUBLESHOOTING.md#build-tool-errors) |
| `winget download` fails | ID is wrong, or the package is MSStore-only | [`winget download` Issues](TROUBLESHOOTING.md#winget-download-issues) |
| HTTP 403 from a direct URL | Vendor is blocking the request | [Direct URL Download Issues](TROUBLESHOOTING.md#direct-url-download-issues) |
| Hash mismatch | Vendor shipped a new version | [Hash Verification Failures](TROUBLESHOOTING.md#hash-verification-failures) |
| 7-Zip fails to pack | Output path is locked or on a FAT32 volume | [7-Zip Packing Failures](TROUBLESHOOTING.md#7-zip-packing-failures) |
| Validation reports missing apps | Manual apps were not supplied | [Verification and Validation Failures](TROUBLESHOOTING.md#verification-and-validation-failures) |

---

*See [docs/ADDING-A-VENDOR.md](ADDING-A-VENDOR.md) for how to author a recipe, [docs/RECIPE-SCHEMA.md](RECIPE-SCHEMA.md) for the field reference, [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md) for `winget download` specifics, and [docs/TROUBLESHOOTING.md](TROUBLESHOOTING.md) for a symptom-first guide.*
