# Adding a Vendor

This document walks through adding a new OEM vendor to the build system. It uses Dell as the reference: Dell is the most complete vendor in the repository, and its recipe, assets, and build flow illustrate every pattern you will need.

If you are looking to add a new app to an existing vendor, see the "Extending an Existing Vendor" section at the end.

---

## Table of Contents

- [Before You Start](#before-you-start)
- [Overview of the Vendor Structure](#overview-of-the-vendor-structure)
- [Step 1 — Create the Vendor Folder](#step-1--create-the-vendor-folder)
- [Step 2 — Gather the Framework Manifest](#step-2--gather-the-framework-manifest)
- [Step 3 — Inventory the Apps](#step-3--inventory-the-apps)
- [Step 4 — Classify Each App's Source](#step-4--classify-each-apps-source)
- [Step 5 — Build the Recipe Skeleton](#step-5--build-the-recipe-skeleton)
- [Step 6 — Populate the `fetch` Array](#step-6--populate-the-fetch-array)
- [Step 7 — Add Committed Assets](#step-7--add-committed-assets)
- [Step 8 — Populate the `assets` Array](#step-8--populate-the-assets-array)
- [Step 9 — Validate the Recipe](#step-9--validate-the-recipe)
- [Step 10 — Build the Archive](#step-10--build-the-archive)
- [Step 11 — Verify the Archive](#step-11--verify-the-archive)
- [Step 12 — Commit and Open a PR](#step-12--commit-and-open-a-pr)
- [Extending an Existing Vendor](#extending-an-existing-vendor)
- [The Dell Reference](#the-dell-reference)
- [Common Pitfalls](#common-pitfalls)

---

## Before You Start

Adding a vendor is a substantial contribution. Before you begin, confirm the following:

1. **The vendor is in the supported list.** The build tool currently recognizes these eleven vendors:

   `Dell`, `HP`, `Lenovo`, `ASUS`, `Acer`, `MSI`, `Gigabyte`, `Dynabook`, `Huawei`, `Microsoft`, `Proline`

   If you want to add a vendor that is not in the list, open a Discussion first. Adding a vendor requires updating the framework's vendor detection and manifest in the deployment repository, which is a coordinated change across two repositories.

2. **You have access to a machine from that vendor.** You cannot test a vendor recipe without hardware to validate it against. Emulators and VMs do not exhibit the manufacturer and model strings that the deployment framework reads.

3. **You have write access to the deployment share.** The recipe is validated against the framework manifest, which lives in your deployment share at:

   ```
   \\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\<Vendor>.json
   ```

4. **You have cloned this repository and the deployment repository.** The recipe lives here; the manifest lives in the deployment share.

5. **You have a working copy of 7-Zip, `winget`, and PowerShell 5.1 or later.** See [docs/BUILDING-PACKS.md](BUILDING-PACKS.md) for the full prerequisites list.

---

## Overview of the Vendor Structure

Every vendor folder has the same shape:

```
vendors/<Vendor>/
├── Recipe.json                The build recipe (this document describes how to write it)
├── README.md                  Vendor-specific build notes (optional but recommended)
├── Assets/                    Committed customization files
│   ├── Customizations.ps1
│   ├── csup.txt
│   ├── gpsFix.reg
│   ├── OEMinfo.reg
│   ├── unattend.xml
│   ├── OEM.7z
│   └── Customizations/
│       ├── <Vendor>.7z
│       └── <Family>.7z
└── Overrides/                 Optional manual URL overrides (advanced; usually empty)
```

Not every vendor needs every asset. The Dell structure is one example. Microsoft (Surface) has a much smaller set; Gigabyte and Proline currently have almost no assets at all.

The **only required file** is `Recipe.json`. Everything else is optional, but a recipe without committed assets will only fetch installers and won't apply any vendor customizations.

---

## Step 1 — Create the Vendor Folder

If the vendor folder does not exist yet, create it:

```powershell
New-Item -Path "vendors\<Vendor>" -ItemType Directory -Force
New-Item -Path "vendors\<Vendor>\Assets" -ItemType Directory -Force
```

The `vendors/<Vendor>/` folder is the entry point. The `Recipe.json` file, any vendor-specific `README.md`, and the `Assets/` tree live inside it.

The vendor name must match the list of supported vendors exactly (case-sensitive). `Dell`, not `DELL`. `HP`, not `Hp`.

---

## Step 2 — Gather the Framework Manifest

Before writing a recipe, you need to know which apps the deployment framework expects for this vendor. The manifest lives in the deployment share:

```
\\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\<Vendor>.json
```

Read it. Every `AppName` in the manifest must have a matching `fetch` entry in your recipe. If the manifest is empty or does not exist, you need to add the manifest to the deployment repository first. That is a separate change; open an issue to coordinate.

Example for Dell (abbreviated):

```json
{
  "name": "Dell",
  "apps": [
    {
      "AppName": "Microsoft.WindowsAppRuntime.1.8",
      "InstallerPath": "C:\\Recovery\\OEM\\Apps\\SupportAssist\\PreinstallKit",
      "InstallerFilter": "WindowsAppRuntimeInstall*",
      "InstallerArgs": "--quiet"
    },
    {
      "AppName": "Dell SupportAssist",
      "InstallerPath": "C:\\Recovery\\OEM\\Apps\\SupportAssist",
      "InstallerFilter": "Dell SupportAssist*",
      "InstallerArgs": ""
    },
    {
      "AppName": "Dell Command | Update for Windows Universal",
      "InstallerPath": "C:\\Recovery\\OEM\\Apps\\CommandUpdate",
      "InstallerFilter": "Dell-Command-Update*",
      "InstallerArgs": "/s /l=C:\\ProgramData\\OEM\\Logs\\DellCommandUpdateUniversal_Install.log"
    }
  ]
}
```

Note the `InstallerPath` values. Those are the paths the framework expects each file to be staged at, both on the target (after extraction) and inside the archive. The build tool uses those paths as the target for each fetched installer. Your recipe does not need to restate them.

**Key point:** The framework manifest is authoritative for install behaviour. The recipe is authoritative for fetch behaviour. Do not duplicate anything from the manifest into the recipe except the `AppName`.

---

## Step 3 — Inventory the Apps

List every app the manifest declares. For each one, gather:

- The `AppName` (exact string from the manifest)
- The `InstallerPath` and `InstallerFilter` (to know what filename the tool needs to produce)
- The vendor's download page for that app (to find the source URL or winget ID)
- The expected installer file type (`.exe`, `.msi`, `.msix`, and so on)

Write this down in a scratch file. You will refer to it while writing the recipe.

Example inventory for Dell:

| AppName | InstallerPath | Expected File Type | Source? |
|---|---|---|---|
| `Microsoft.WindowsAppRuntime.1.8` | `Apps\SupportAssist\PreinstallKit` | `.exe` | winget |
| `Microsoft .NET Windows Desktop Runtime 8` | `Apps\SupportAssist\PreinstallKit` | `.exe` | winget |
| `Dell SupportAssist` | `Apps\SupportAssist` | `.exe` | direct URL |
| `Dell Command | Update for Windows Universal` | `Apps\CommandUpdate` | `.exe` | winget |
| `Alienware Command Center (v6)` | `Apps\CommandCenter\v6` | `.exe` | direct URL |
| `DellInc.DellSupportAssistforPCs` | `Apps\SupportAssist\UWP` | `.msix` | manual (MSStore only) |

The inventory is your roadmap. Every row becomes one entry in the recipe's `fetch` array.

---

## Step 4 — Classify Each App's Source

For each app, decide which of the four source types applies. Use this decision tree:

```
Is the app published in the winget repository as an MSI or EXE?
├── Yes → Source: winget
└── No → Continue.

Does the vendor publish the installer at a stable HTTPS URL?
├── Yes → Source: direct-url
└── No → Continue.

Does the file ship in the repository under vendors/<Vendor>/Assets/?
├── Yes → Source: repo-asset
└── No → Source: manual
```

The order matters. Prefer `winget` over `direct-url`, because `winget` handles version pinning and hash verification automatically. Prefer `direct-url` over `manual`, because it keeps the build unattended. Use `manual` only when nothing else works.

### Testing a `winget` candidate

Before you commit to a `winget` source, verify the package downloads:

```powershell
$tmp = "$env:TEMP\winget-test"
New-Item -ItemType Directory -Path $tmp -Force | Out-Null

winget download `
    --id <WingetId> `
    --download-directory $tmp `
    --accept-source-agreements `
    --accept-package-agreements `
    --disable-interactivity

if ($LASTEXITCODE -eq 0) {
    Get-ChildItem $tmp
} else {
    "Failed with exit code $LASTEXITCODE"
}
```

If the download succeeds and produces an `.msi` or `.exe`, the app is a valid `winget` source. If it fails, or produces a `.msix` or `.appxbundle`, use `direct-url` or `manual` instead.

### Testing a `direct-url` candidate

For a direct URL, download it and record the hash:

```powershell
$url = "https://dl.dell.com/FOLDER.../Dell-SupportAssist.exe"
$dest = "$env:TEMP\Dell-SupportAssist.exe"

Invoke-WebRequest -Uri $url -OutFile $dest -UseBasicParsing
(Get-FileHash -Path $dest -Algorithm SHA256).Hash.ToLower()
```

Copy the hash into the recipe. The build tool will verify every download against it.

---

## Step 5 — Build the Recipe Skeleton

Create `vendors/<Vendor>/Recipe.json` with the top-level fields:

```json
{
  "recipeVersion": "1.0",
  "name": "Dell",
  "frameworkManifest": "Dell.json",
  "architecture": "x64",
  "fetch": [],
  "assets": []
}
```

Fill in:

- `recipeVersion` — always `"1.0"` for the current schema
- `name` — must match the folder name exactly
- `frameworkManifest` — the filename of the framework manifest (usually `<name>.json`)
- `architecture` — `"x64"` or `"x86"`

Leave `fetch` and `assets` empty for now. You will populate them in the next two steps.

If you are also supporting x86 for this vendor, create a parallel `vendors/<Vendor>-x86/` folder with its own recipe. The two recipes are independent.

---

## Step 6 — Populate the `fetch` Array

For each row in your app inventory, add one entry to the `fetch` array. The fields depend on the source type.

### `winget` entry

```json
{
  "AppName": "Dell Command | Update for Windows Universal",
  "Source": "winget",
  "WingetId": "Dell.CommandUpdate.Universal"
}
```

Add `WingetArchitecture` and `WingetLocale` only if the package needs them (multi-arch packages or locale-specific installers).

### `direct-url` entry

```json
{
  "AppName": "Dell SupportAssist",
  "Source": "direct-url",
  "Url": "https://dl.dell.com/FOLDER06731690M/1/Dell-SupportAssist.exe",
  "Sha256": "a1b2c3d4e5f6..."
}
```

Always include `Sha256`. Omitting it produces a warning at build time and defeats the purpose of verification.

### `repo-asset` entry

```json
{
  "AppName": "Dell Customizations",
  "Source": "repo-asset",
  "AssetPath": "Assets\\Customizations.ps1"
}
```

The `AssetPath` is relative to `vendors/<Vendor>/`.

### `manual` entry

```json
{
  "AppName": "DellInc.DellSupportAssistforPCs",
  "Source": "manual",
  "Notes": "MSIX bundle from the Microsoft Store. Download on a machine with a work or school account, then place the .msix at C:\\Recovery\\OEM\\Apps\\SupportAssist\\UWP\\SupportAssist_x64.msix before building.",
  "ExpectedFilename": "SupportAssist_x64.msix"
}
```

The `Notes` field is what the user sees when the build reports the app as pending. Make it actionable: where to obtain the file, and where to place it.

### Ordering

Order the `fetch` entries to match the framework manifest's `apps` array. This makes diff review easier. The build tool does not require the order to match, but reviewers do.

---

## Step 7 — Add Committed Assets

The `Assets/` folder contains everything that ships inside the vendor archive but is not downloaded from the vendor. For Dell, this includes:

- `Customizations.ps1` — the vendor's pre-install customization script
- `csup.txt` — vendor metadata consumed by `SetupComplete.cmd`
- `gpsFix.reg` — registry tweaks
- `OEMinfo.reg` — OEM branding registry
- `unattend.xml` — OEM-attend overlay for PBR
- `OEM.7z` — infrastructure extracted to `C:\OEM` by `Customizations.ps1`
- `Customizations/Dell.7z` — wallpapers and themes for the default family
- `Customizations/G-series.7z` — additional assets for the G-series family

For your vendor, gather whatever customizations exist. If you are starting from scratch, the minimum useful set is:

- A `Customizations.ps1` (or equivalent) that applies branding
- A wallpaper or theme pack, packaged as a nested `.7z`
- Any `.reg` files the customization script depends on

### File size limits

- Wallpapers and theme assets: up to 10 MB per file, 50 MB per vendor total
- Everything else: up to a few MB per file

If you need to ship something larger, open an issue first. Large binaries should be fetched from an official source at build time, not committed.

### Naming and encoding

- Use descriptive filenames. `Dell-G15-wallpaper.jpg`, not `img001.jpg`.
- Text files use UTF-8 without BOM.
- `.reg` files use UTF-16 LE with BOM.
- Line endings follow the repository's `.gitattributes` rules.

### Where nested archives go

Nested `.7z` files (like `OEM.7z` and `Customizations/Dell.7z`) go under `Assets/` in a folder structure that mirrors what they contain. The `assets` array in the recipe names the target path inside the parent archive.

---

## Step 8 — Populate the `assets` Array

For every file under `Assets/` that must ship inside the archive, add one entry to the `assets` array:

```json
{
  "Source": "Assets\\Customizations.ps1",
  "TargetPath": "."
}
```

The `Source` is the path under `vendors/<Vendor>/`. The `TargetPath` is the path inside the archive. `"."` places the file at the archive root.

For nested assets:

```json
{
  "Source": "Assets\\Customizations\\Dell.7z",
  "TargetPath": "Customizations"
}
```

This places `Dell.7z` at `Customizations\Dell.7z` inside the archive.

Every file under `Assets/` that you want in the archive must have a matching `assets` entry. Files without an entry are not copied. This is intentional — it lets you keep helper files (a `.gitkeep`, a scratch note) in `Assets/` without shipping them.

---

## Step 9 — Validate the Recipe

Before running a full build, validate the recipe against the schema and the framework manifest:

```powershell
.\tools\Build-OEMPack.ps1 -Vendor <Vendor> -DryRun
```

A dry run:

1. Loads the recipe and validates it against `schema/vendor-recipe.schema.json`.
2. Loads the framework manifest from the deployment share.
3. Checks that every `AppName` matches between the two.
4. Checks that every `assets` entry's `Source` file exists on disk.
5. Reports what would be fetched, what would be manually supplied, and what would be packed.

A clean dry run looks like:

```
[INFO]    Loaded recipe: vendors\Dell\Recipe.json
[INFO]    Loaded manifest: \\SERVER\DeploymentShare$\...\Manifests\Dell.json
[SUCCESS] Manifest and recipe are in sync (14 apps)
[INFO]    12 apps will be fetched from remote sources
[INFO]     8 via winget
[INFO]     4 via direct-url
[INFO]    1 app is manual (SupportAssist UWP)
[INFO]    8 assets will be staged
[SUCCESS] Dry run completed successfully
```

If the dry run reports errors, fix them before proceeding. Common errors:

| Error | Cause | Fix |
|---|---|---|
| "Manifest and recipe are out of sync" | Missing or extra `AppName` | Add or remove `fetch` entries to match the manifest |
| "Asset not found" | `assets[].Source` points at a missing file | Add the file under `vendors/<Vendor>/Assets/` |
| "Recipe version not supported" | `recipeVersion` is not `"1.0"` | Set it to `"1.0"` |
| "Vendor name mismatch" | `name` does not match the folder name | Rename one to match the other |

---

## Step 10 — Build the Archive

Once the dry run passes, run a real build:

```powershell
.\tools\Build-OEMPack.ps1 -Vendor <Vendor> -OutputPath C:\Output
```

The build tool:

1. Downloads each `winget` app via `Resolve-WingetPackage.ps1`
2. Downloads each `direct-url` app via `Resolve-OEMUrl.ps1` and verifies its hash
3. Copies each `repo-asset` file into the staging folder
4. Skips `manual` apps and records them in the summary
5. Copies each asset into the staging folder at its `TargetPath`
6. Packs the staging folder into `<Vendor>.7z` at the output path
7. Runs the validator against the built archive

Expect the first build to take 10 minutes to 2 hours depending on the number and size of apps, your network speed, and how many apps are `manual`.

The build prints a summary at the end:

```
═══════════════════════════════════════════════════════════════════
 BUILD SUMMARY: Dell
═══════════════════════════════════════════════════════════════════
  Total apps        : 14
  Downloaded        : 13
  Cached (skipped)  : 0
  Manual (pending)  : 1
  Failed            : 0
  Archive           : C:\Output\Dell.7z
  Size              : 3.42 GB
  SHA-256           : a1b2c3d4e5f6...
═══════════════════════════════════════════════════════════════════

  MANUAL APPS STILL PENDING:
    - DellInc.DellSupportAssistforPCs
      Notes: MSIX bundle from the Microsoft Store. Download on a machine
             with a work or school account, then place the .msix at
             C:\Recovery\OEM\Apps\SupportAssist\UWP\SupportAssist_x64.msix
             before building.
```

If the build reports a failure, see [docs/TROUBLESHOOTING.md](TROUBLESHOOTING.md).

---

## Step 11 — Verify the Archive

The build tool automatically runs the validator at the end. To validate independently:

```powershell
.\tools\Test-OEMAppPack.ps1 `
    -Archive C:\Output\Dell.7z `
    -Manifest \\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json
```

A clean validation:

```
[INFO]    Loading archive: C:\Output\Dell.7z
[INFO]    Loading manifest: \\SERVER\DeploymentShare$\...\Manifests\Dell.json
[INFO]    Listing archive contents...
[SUCCESS] All 14 apps present at the expected paths
[SUCCESS] All 8 assets present at the expected paths
[SUCCESS] Validation passed
```

Then manually inspect the archive:

```powershell
& "C:\Program Files\7-Zip\7z.exe" l C:\Output\Dell.7z
```

Compare the listing against what the framework expects. Pay attention to:

- File paths match the framework manifest's `InstallerPath` + `InstallerFilter`
- Asset paths match the recipe's `TargetPath`
- No unexpected files
- No missing files

If anything is wrong, fix the recipe and rebuild.

### Testing end-to-end (strongly recommended)

For a new vendor, verification against the manifest is not sufficient. You should also:

1. Copy the built archive to your test deployment share at `\\SERVER\Shared\OEM\x64\<Vendor>.7z`
2. PXE boot a machine from that vendor
3. Run a full deployment through OOBE
4. Verify that the OEM apps install and the framework converges to `USER_DONE`

This is the only way to confirm that the archive works end-to-end. A recipe that validates perfectly but fails on real hardware is worse than no recipe at all.

---

## Step 12 — Commit and Open a PR

Once the build succeeds and the archive is validated:

1. Commit your changes:

   ```
   git add vendors/<Vendor>/
   git commit -m "feat(<vendor>): add vendor recipe"
   ```

   Use the conventional commit format. Scope is the vendor name in lowercase.

2. Update `CHANGELOG.md` under `[Unreleased]` → `### Added`:

   ```
   - **<Vendor>** — Recipe covering all applications declared in the
     framework's `Manifests/<Vendor>.json`
   ```

3. Push to your fork and open a PR against `main`. Include in the PR description:

   - The vendor name
   - The Windows build you tested on
   - The hardware model you validated against (if applicable)
   - The SHA-256 of the built archive
   - Any apps marked `manual` and why
   - Any assets you added and what they do

4. Wait for review. Reviews focus on:

   - Recipe structure and correctness
   - Hash verification coverage
   - `manual` entries — are they justified?
   - Asset content — is anything committed that should not be?
   - No personal or machine-specific values in committed files

---

## Extending an Existing Vendor

Adding a new app to an existing vendor is simpler than adding a whole vendor. The workflow:

1. **Add the app to the framework manifest first.** The manifest lives in the deployment repository, not here. If the app you want to add is not in the manifest, it will not be installed even if the recipe fetches it. The manifest change is a separate PR against `MDT-Zero-Touch-Deployment`.

2. **Add the corresponding `fetch` entry to the recipe.** Match the `AppName` exactly.

3. **If the app is `manual`, add a `Notes` field** explaining how the user obtains the file.

4. **Run a dry run** to confirm the manifest and recipe agree.

5. **Run a real build** and validate the archive.

6. **Open a PR.** Include the framework manifest change and the recipe change as one coordinated PR, or two linked PRs.

### Removing an app

Removing an app from a vendor is the reverse: remove the entry from the framework manifest first, then remove the `fetch` entry from the recipe. A recipe with an orphan `fetch` entry fails validation. A manifest with an orphan `AppName` fails the sync check.

---

## The Dell Reference

Dell is the most complete vendor in the repository. When you are unsure how to classify or format something, look at `vendors/Dell/Recipe.json` and its `Assets/` folder.

### App sources in the Dell recipe

| AppName | Source | Why |
|---|---|---|
| `Microsoft.WindowsAppRuntime.1.8` | `winget` | Available as an MSI in winget |
| `Microsoft .NET Windows Desktop Runtime 8` | `winget` | Available as an MSI in winget |
| `Microsoft .NET Windows Desktop Runtime 10` | `winget` | Available as an MSI in winget |
| `Dell SupportAssist` | `direct-url` | Winget only has the Store version; Dell publishes a standalone installer |
| `DellInc.DellSupportAssistforPCs` | `manual` | UWP MSIX, MSStore-only, no unattended download path |
| `Alienware Command Center (v6)` | `direct-url` | Winget only has the Store version |
| `Alienware Command Center (v5)` | `direct-url` | Older version, kept for legacy hardware |
| `DellInc.MyAlienware` | `winget` (`msstore`) | Store-only, user installs after first logon |
| `Fusion Service` | `direct-url` | Dell-published installer |
| `Dell Optimizer` | `winget` | Available as an MSI in winget |
| `Dell Precision Optimizer` | `direct-url` | Precision-specific, not in winget |
| `Dell Power Manager Service` | `direct-url` | Dell-published installer |
| `Dell Command | Update for Windows Universal` | `winget` | Available as an MSI in winget |
| `DellInc.DellCommandUpdate` | `direct-url` | UWP companion, Dell publishes an appxbundle |

Note that several apps are `direct-url` even though a Store version exists in winget. The standalone Dell installers are used because they can be silently installed without Store authentication. Store-only paths are used only when the app is genuinely a UWP that cannot be distributed otherwise.

### Assets in the Dell recipe

Every file under `vendors/Dell/Assets/` is declared in the `assets` array. The `TargetPath` for most files is `"."` (archive root). The two nested `Customizations/*.7z` files go under `Customizations/`.

---

## Common Pitfalls

### Committing vendor installers

**Never** commit a vendor installer to `Assets/`. The repository's `.gitignore` blocks most binary extensions, but you can force-add them. Do not. Vendor installers belong in the cache, not in Git.

If you believe a specific installer must be committed (for example, because the vendor does not publish a stable URL), open an issue first. The answer is almost always to use `manual` instead.

### Using `manual` too readily

`manual` shifts work onto every user of the recipe. Before you use it, exhaust the other three options. A `manual` entry is a last resort, not a convenience.

### Forgetting the framework manifest

The recipe is validated against the framework manifest at build time. If you add apps to the recipe that are not in the manifest, or vice versa, the build fails. Always update the manifest in `MDT-Zero-Touch-Deployment` before or alongside the recipe.

### Not testing on real hardware

A recipe that validates cleanly against the manifest can still fail on real hardware. Model strings may not match, installers may require interactive dialogs, or the framework may not recognize the vendor. Test on real hardware before opening a PR.

### Committing personal values

Never commit a recipe or asset that contains:

- A product key
- A credential
- A machine-specific name
- A personal organization name

These are the most common reasons a PR is rejected.

### Skipping the `Sha256` field

A `direct-url` entry without a `Sha256` produces a warning and defeats the purpose of hash verification. Always include it. If you do not know the hash yet, run the build once, copy the observed hash from the log, and add it in a follow-up commit.

### Using the wrong asset encoding

`.reg` files must be UTF-16 LE with BOM. Text files must be UTF-8 without BOM. Getting this wrong causes silent failures at deployment time.

### Adding a vendor that is not in the supported list

The supported vendor list is defined in the deployment repository's `ExtractOEMAppsx64.ps1` and in the framework's vendor detection logic. Adding a vendor outside that list requires coordinated changes in the deployment repository. Open a Discussion first.

---

*See [docs/RECIPE-SCHEMA.md](RECIPE-SCHEMA.md) for the field reference, [docs/BUILDING-PACKS.md](BUILDING-PACKS.md) for the build workflow, and [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md) for `winget download` behaviour.*
