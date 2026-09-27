# Recipe Schema Reference

Complete field reference for the vendor `Recipe.json` files that drive the build tools in this repository. Every vendor under `vendors/<Vendor>/` has a `Recipe.json` that declares what to fetch, from where, and how to pack it.

The schema is defined in [`schema/vendor-recipe.schema.json`](../schema/vendor-recipe.schema.json). This document is the human-readable companion to that schema.

---

## Table of Contents

- [Overview](#overview)
- [Design Principles](#design-principles)
- [Top-Level Structure](#top-level-structure)
- [The `fetch` Array](#the-fetch-array)
  - [Common Fields](#common-fields)
  - [`winget` Source](#winget-source)
  - [`direct-url` Source](#direct-url-source)
  - [`repo-asset` Source](#repo-asset-source)
  - [`manual` Source](#manual-source)
- [The `assets` Array](#the-assets-array)
- [Bundling Nested Archives](#bundling-nested-archives)
- [Framework Manifest Alignment](#framework-manifest-alignment)
- [Recipe Versioning](#recipe-versioning)
- [Validation Rules](#validation-rules)
- [Complete Example](#complete-example)
- [Field Reference Tables](#field-reference-tables)

---

## Overview

A recipe is a JSON document with two responsibilities:

1. **Tell the build tool what to fetch.** For every app the deployment framework expects, the recipe says where that app comes from — `winget`, a direct vendor URL, a committed repository asset, or manual supply.
2. **Tell the build tool what to pack.** Committed customization assets (wallpapers, registry files, `Customizations.ps1`, and so on) are declared in the recipe with their target paths inside the archive.

The recipe does **not** describe install behaviour. That belongs to the framework manifest in the deployment repository. See [Framework Manifest Alignment](#framework-manifest-alignment) below.

---

## Design Principles

Three principles shape the schema:

1. **Single source of truth for install behaviour.** The deployment repository's `Manifests/<Vendor>.json` declares how each app is installed — `AppName`, `InstallerPath`, `InstallerArgs`, eligibility, pinning. The recipe does **not** duplicate any of that. It adds only the build-time information the framework manifest cannot know: where to fetch each app from, and which committed assets to bundle.

2. **Explicit is better than inferred.** Every app declares exactly one source type. No `auto`. No guessing. The build tool fails loudly when it cannot resolve a source.

3. **Small surface, extensible.** The schema is intentionally minimal. New source types and new optional fields can be added without breaking existing recipes.

---

## Top-Level Structure

```json
{
  "recipeVersion": "1.0",
  "name": "Dell",
  "frameworkManifest": "Dell.json",
  "architecture": "x64",
  "fetch": [ /* app entries */ ],
  "assets": [ /* asset entries */ ]
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `recipeVersion` | string | yes | Schema version the recipe targets. See [Recipe Versioning](#recipe-versioning). |
| `name` | string | yes | Vendor token. Must match the folder name and the `name` field in the framework manifest. |
| `frameworkManifest` | string | yes | Filename of the framework manifest this recipe pairs with. Usually `<name>.json`. |
| `architecture` | string | yes | `"x64"` or `"x86"`. Selects which framework manifest the tool validates against. |
| `fetch` | array | yes | One entry per app. See [The `fetch` Array](#the-fetch-array). |
| `assets` | array | no | Committed customization assets. See [The `assets` Array](#the-assets-array). |

---

## The `fetch` Array

Every entry in `fetch` corresponds to exactly one `AppName` in the framework manifest. The build tool validates this and fails if the two are out of sync.

### Common Fields

These fields appear in every `fetch` entry, regardless of source type.

| Field | Type | Required | Notes |
|---|---|---|---|
| `AppName` | string | yes | Must match the `AppName` in the framework manifest exactly. |
| `Source` | string | yes | One of `"winget"`, `"direct-url"`, `"repo-asset"`, `"manual"`. |

The remaining fields depend on the `Source` value.

### `winget` Source

Use `winget` when the app is available in the winget package repository as an MSI or EXE.

```json
{
  "AppName": "Dell Command | Update for Windows Universal",
  "Source": "winget",
  "WingetId": "Dell.CommandUpdate.Universal",
  "WingetArchitecture": "x64",
  "WingetLocale": "en-US"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `WingetId` | string | yes | The winget package ID. Verify with `winget show --id <id>`. |
| `WingetArchitecture` | string | no | Forces `--architecture`. Only for multi-arch packages where x64 must be explicit. |
| `WingetLocale` | string | no | Forces `--locale`. Only for locale-specific installers. |

The build tool runs:

```powershell
winget download `
    --id $WingetId `
    --download-directory $StagingDir `
    --accept-source-agreements `
    --accept-package-agreements `
    --disable-interactivity
```

The downloaded file is staged at the path the framework manifest declares for this `AppName`. The recipe does not declare the target path.

**Limitations:**

- `winget download` does not work for MSStore-only packages (UWP, MSIX) without a work or school account. Those must use `manual`.
- The downloaded filename changes with the package version. The build tool does not depend on a fixed name — it picks the single new file from the download directory.

See [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md) for the full `winget download` behaviour.

### `direct-url` Source

Use `direct-url` when the vendor publishes the installer at a stable HTTPS URL.

```json
{
  "AppName": "Dell SupportAssist",
  "Source": "direct-url",
  "Url": "https://dl.dell.com/FOLDER06731690M/1/Dell-SupportAssist.exe",
  "Sha256": "a1b2c3d4e5f6..."
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `Url` | string | yes | Must resolve without authentication. |
| `Sha256` | string | recommended | 64-character lowercase hex SHA-256 of the file. Strongly recommended. |
| `Version` | string | no | Informational. Recorded for changelog and diff review. |

If `Sha256` is provided, the tool verifies the downloaded file against it and fails on mismatch. If `Sha256` is omitted, the tool computes the hash, emits a warning, and records the observed hash in its log so the recipe can be updated with a follow-up PR.

The downloaded file is staged at the path the framework manifest declares for this `AppName`.

### `repo-asset` Source

Use `repo-asset` when the file ships in the repository itself, under `vendors/<Vendor>/Assets/`. This is only appropriate for small customization assets, not vendor installers.

```json
{
  "AppName": "Dell Customizations",
  "Source": "repo-asset",
  "AssetPath": "Assets\\Customizations.ps1"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `AssetPath` | string | yes | Relative path under `vendors/<Vendor>/`. |

The build tool copies the file from `vendors/<Vendor>/<AssetPath>` to the path the framework manifest declares for this `AppName`.

**File size limit:** No file under `Assets/` should exceed 10 MB unless it is a wallpaper or theme asset. If you believe a larger file belongs in `Assets/`, open an issue first and explain why.

### `manual` Source

Use `manual` when neither `winget` nor a stable direct URL exists. Every `manual` entry shifts work onto every user of the recipe, so use it as a last resort.

```json
{
  "AppName": "DellInc.DellSupportAssistforPCs",
  "Source": "manual",
  "Notes": "MSIX bundle from the Microsoft Store. Download on a machine with a work or school account, then place the .msix at C:\\Recovery\\OEM\\Apps\\SupportAssist\\UWP\\SupportAssist_x64.msix before building.",
  "ExpectedFilename": "SupportAssist_x64.msix"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `Notes` | string | yes | Instructions the user follows to obtain the file. Printed in the build summary and log. |
| `ExpectedFilename` | string | yes | The filename the user must supply. Used to verify the file is present at the expected path. |

The build tool does not attempt a download. It checks whether the expected file is already present at the framework manifest's `InstallerPath`. If it is present, the build proceeds. If not, the app is recorded in the build summary as pending, and the archive is built without it. The user is told exactly what is missing and where to place it.

When you use `manual`, describe the situation in a comment in the recipe (JSON does not support comments, so put it in a sibling `README.md` under `vendors/<Vendor>/`). Future maintainers will thank you.

---

## The `assets` Array

The `assets` array declares committed customization files that must ship inside the archive. Every entry has a `Source` (a path under `vendors/<Vendor>/Assets/`) and a `TargetPath` (a path relative to the archive root).

```json
{
  "assets": [
    {
      "Source": "Assets\\Customizations.ps1",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\csup.txt",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\gpsFix.reg",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\OEMinfo.reg",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\unattend.xml",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\OEM.7z",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\Customizations\\Dell.7z",
      "TargetPath": "Customizations"
    },
    {
      "Source": "Assets\\Customizations\\G-series.7z",
      "TargetPath": "Customizations"
    }
  ]
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `Source` | string | yes | Relative path under `vendors/<Vendor>/`. Must exist on disk. |
| `TargetPath` | string | yes | Relative path inside the archive. `"."` places the file at the archive root. |

A `TargetPath` of `"."` places the file at the archive root. Any other value places it under a folder of that name. Nested paths use backslashes (`"Customizations\\Wallpapers"`).

When the archive is extracted on the target, the top-level structure is:

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

Everything declared in `assets` lands at the top level or under the folders named in `TargetPath`. Everything declared in `fetch` lands under `Apps\` at the path the framework manifest declares.

---

## Bundling Nested Archives

Some vendor payloads contain nested `.7z` files that are extracted during deployment, not during the build. For example, the Dell payload ships:

- `OEM.7z` — infrastructure extracted to `C:\OEM` by `Customizations.ps1`
- `Customizations\Dell.7z` — wallpapers and themes for the default family
- `Customizations\G-series.7z` — additional assets for the G-series family

These nested archives are **content**, not build output. The recipe treats them as `assets` and copies them into the archive unchanged. The build tool does not extract, rebuild, or inspect them.

The rationale: the nested archives are extracted by `Customizations.ps1` at deployment time with logic that lives in the deployment repository. The build tool has no reason to know what is inside them. It just ensures they land at the right path inside the parent archive.

If you need to build a nested archive from sources, do that manually and commit the resulting `.7z` to `Assets/`. The tool does not build nested archives.

---

## Framework Manifest Alignment

The recipe and the framework manifest must agree on the set of apps. The build tool validates this and fails on any mismatch:

```
[ERROR] Manifest and recipe are out of sync
[ERROR]   In manifest but not in recipe: Dell Command | Update for Windows Universal
[ERROR]   In recipe but not in manifest: DellCmdUpdate
```

**Every `AppName` in the framework manifest must have exactly one `fetch` entry in the recipe.** Conversely, every `fetch` entry must reference an `AppName` that exists in the framework manifest.

The build tool does **not** require field-level duplication. The framework manifest declares the install behaviour (`InstallerPath`, `InstallerArgs`, eligibility, pinning). The recipe declares only the fetch behaviour (`Source`, `WingetId` or `Url`, and so on). There is no overlap.

**Where the framework manifest lives:**

```
\\SERVER\DeploymentShare$\<arch>\$OEM$\$1\Recovery\OEM\Apps\Manifests\<Vendor>.json
```

**Where the recipe lives:**

```
vendors\<Vendor>\Recipe.json
```

**How they are paired:** the recipe's `frameworkManifest` field names the manifest filename. The tool loads the manifest from the deployment share path (passed via `-ManifestPath`) and matches apps by `AppName`.

---

## Recipe Versioning

The `recipeVersion` field declares which schema version the recipe targets. This lets the build tool refuse to load a recipe written for a newer schema, and lets future schema changes be staged without breaking existing recipes.

Current version: **`"1.0"`**

When the schema changes in a backward-compatible way (new optional field, new source type), the version stays at `1.0`. When the schema changes in a breaking way (required field added, field renamed, field removed), the version bumps to `1.1`, `2.0`, and so on, and the build tool refuses to load older recipes until they are updated.

No migrations are automated. When a breaking change occurs, users update their recipes by hand.

---

## Validation Rules

The build tool enforces the following rules at load time. A recipe that fails any rule is rejected before any download or packing work begins.

| Rule | What It Checks |
|---|---|
| Recipe file exists | `vendors/<Vendor>/Recipe.json` must exist. |
| JSON is valid | The file must parse as JSON. |
| Schema is valid | The file must match `schema/vendor-recipe.schema.json`. |
| Recipe version supported | `recipeVersion` must be `"1.0"` (or a version the tool understands). |
| Vendor name matches folder | `name` must equal the folder name under `vendors/`. |
| Architecture is known | `architecture` must be `"x64"` or `"x86"`. |
| Framework manifest exists | The manifest named in `frameworkManifest` must exist at the configured manifest path. |
| App names match exactly | Every `AppName` in the manifest must have a matching entry in `fetch`, and vice versa. |
| Every source is explicit | Every `fetch` entry must have a `Source` field with one of the four valid values. |
| Source-specific fields present | `WingetId` for `winget`, `Url` for `direct-url`, `AssetPath` for `repo-asset`, `Notes` and `ExpectedFilename` for `manual`. |
| Asset files exist | Every `assets` entry's `Source` must exist on disk. |
| Asset paths are relative | No absolute paths in `Source` or `TargetPath`. |
| Asset sizes are reasonable | No file under `Assets/` exceeds 10 MB unless it is a wallpaper or theme asset. |

Failing any rule produces a clear error and a non-zero exit code. The tool never silently skips a bad entry.

---

## Complete Example

A complete Dell recipe, abbreviated for illustration:

```json
{
  "recipeVersion": "1.0",
  "name": "Dell",
  "frameworkManifest": "Dell.json",
  "architecture": "x64",

  "fetch": [
    {
      "AppName": "Microsoft.WindowsAppRuntime.1.8",
      "Source": "winget",
      "WingetId": "Microsoft.WindowsAppRuntime.1.8"
    },
    {
      "AppName": "Microsoft .NET Windows Desktop Runtime 8",
      "Source": "winget",
      "WingetId": "Microsoft.DotNet.DesktopRuntime.8"
    },
    {
      "AppName": "Dell SupportAssist",
      "Source": "direct-url",
      "Url": "https://dl.dell.com/FOLDER06731690M/1/Dell-SupportAssist.exe",
      "Sha256": "a1b2c3d4e5f6..."
    },
    {
      "AppName": "Dell Command | Update for Windows Universal",
      "Source": "winget",
      "WingetId": "Dell.CommandUpdate.Universal"
    },
    {
      "AppName": "Dell Optimizer",
      "Source": "winget",
      "WingetId": "Dell.Optimizer"
    },
    {
      "AppName": "Alienware Command Center (v6)",
      "Source": "direct-url",
      "Url": "https://dl.dell.com/FOLDER.../Alienware-Command-Center-Application-Full-Installer.exe",
      "Sha256": "b2c3d4e5f6a1..."
    },
    {
      "AppName": "Alienware Command Center (v5)",
      "Source": "direct-url",
      "Url": "https://dl.dell.com/FOLDER.../Alienware-Command-Center-5-x-Full-Installer.exe",
      "Sha256": "c3d4e5f6a1b2..."
    },
    {
      "AppName": "DellInc.DellSupportAssistforPCs",
      "Source": "manual",
      "Notes": "MSIX bundle from the Microsoft Store. Download on a machine with a work or school account, then place the .msix at C:\\Recovery\\OEM\\Apps\\SupportAssist\\UWP\\SupportAssist_x64.msix before building.",
      "ExpectedFilename": "SupportAssist_x64.msix"
    }
  ],

  "assets": [
    {
      "Source": "Assets\\Customizations.ps1",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\csup.txt",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\gpsFix.reg",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\OEMinfo.reg",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\unattend.xml",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\OEM.7z",
      "TargetPath": "."
    },
    {
      "Source": "Assets\\Customizations\\Dell.7z",
      "TargetPath": "Customizations"
    },
    {
      "Source": "Assets\\Customizations\\G-series.7z",
      "TargetPath": "Customizations"
    }
  ]
}
```

---

## Field Reference Tables

### Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `recipeVersion` | string | yes | Schema version. Currently `"1.0"`. |
| `name` | string | yes | Vendor token. Must match folder name. |
| `frameworkManifest` | string | yes | Filename of the framework manifest. |
| `architecture` | string | yes | `"x64"` or `"x86"`. |
| `fetch` | array | yes | App fetch entries. |
| `assets` | array | no | Committed asset entries. |

### `fetch` Entry Fields

| Field | Type | Required For | Description |
|---|---|---|---|
| `AppName` | string | all sources | Matches the framework manifest entry exactly. |
| `Source` | string | all sources | `"winget"`, `"direct-url"`, `"repo-asset"`, or `"manual"`. |
| `WingetId` | string | `winget` | Winget package ID. |
| `WingetArchitecture` | string | `winget` (optional) | Forces `--architecture`. |
| `WingetLocale` | string | `winget` (optional) | Forces `--locale`. |
| `Url` | string | `direct-url` | Direct download URL. |
| `Sha256` | string | `direct-url` (recommended) | SHA-256 of the downloaded file. |
| `Version` | string | `direct-url` (optional) | Informational version string. |
| `AssetPath` | string | `repo-asset` | Relative path under `vendors/<Vendor>/`. |
| `Notes` | string | `manual` | Instructions for obtaining the file. |
| `ExpectedFilename` | string | `manual` | Filename the user must supply. |

### `assets` Entry Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `Source` | string | yes | Relative path under `vendors/<Vendor>/`. |
| `TargetPath` | string | yes | Relative path inside the archive. `"."` for archive root. |

---

*See [docs/ADDING-A-VENDOR.md](ADDING-A-VENDOR.md) for a worked example of authoring a recipe, [docs/BUILDING-PACKS.md](BUILDING-PACKS.md) for the build workflow, and [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md) for `winget download` behaviour.*
