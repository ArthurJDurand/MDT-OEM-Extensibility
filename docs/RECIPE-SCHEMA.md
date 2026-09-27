# `winget download` Reference

This document explains how the build tool uses `winget download` to fetch vendor installers, when it works, when it does not, and how to verify a package before adding it to a recipe.

`winget download` is the fastest and most reliable way to fetch Microsoft-signed or vendor-signed installers for MSI and EXE packages. For MSIX and UWP packages from the Microsoft Store, it requires a work or school account and does not work in unattended builds. Those cases fall back to `direct-url` or `manual` sources.

---

## Table of Contents

- [What `winget download` Does](#what-winget-download-does)
- [When It Works](#when-it-works)
- [When It Does Not Work](#when-it-does-not-work)
- [Basic Usage](#basic-usage)
- [Finding the Winget ID for an App](#finding-the-winget-id-for-an-app)
- [Verifying a Package Before Adding It to a Recipe](#verifying-a-package-before-adding-it-to-a-recipe)
- [Deterministic Output Names](#deterministic-output-names)
- [Common Flags](#common-flags)
- [Common Failure Modes](#common-failure-modes)
- [Fallback Strategies](#fallback-strategies)
- [Recipe Source Type: `winget`](#recipe-source-type-winget)
- [Automation Notes](#automation-notes)
- [Related Documentation](#related-documentation)

---

## What `winget download` Does

`winget download` resolves a package's manifest, downloads the installer from the source URL recorded in that manifest, verifies the installer's SHA-256 hash against the manifest's expected hash, and writes the file to a local folder.

It does **not** install anything. It is purely a fetch operation.

Contrast with `winget install`, which resolves the same manifest, downloads the installer, verifies the hash, and then runs the installer silently on the current machine.

For this project, we want `winget download` — the installer is later repacked into a vendor `.7z` archive and shipped to a target machine, where the framework (or `pre.ps1`) runs it during OOBE.

---

## When It Works

`winget download` works when all of the following are true:

1. The package is published in the `winget` source (the community-maintained manifest repository).
2. The manifest declares an `InstallerType` of `msi`, `exe`, `inno`, `nullsoft`, `wix`, `burn`, or any other non-Store type.
3. The installer's `InstallerUrl` in the manifest is a stable, publicly accessible HTTPS URL.
4. The manifest provides a `InstallerSha256` value that matches the file at that URL.

If all four are true, `winget download` succeeds without authentication and without a Microsoft account.

**Common examples that work:**

```
Microsoft.Edge
Google.Chrome
7zip.7zip
RARLab.WinRAR
Notepad++.Notepad++
Dell.CommandUpdate.Universal
Dell.Optimizer
Dell.SupportAssist
HP.SupportAssistant
HP.ImageAssistant
Lenovo.SystemUpdate
Lenovo.Vantage
```

The last four are examples of vendor-published MSI or EXE packages that appear in the `winget` source. They are what makes this workflow possible for OEM apps.

---

## When It Does Not Work

`winget download` fails or requires authentication when **any** of the following is true:

1. The package is published only in the `msstore` source (the Microsoft Store).
2. The package is an MSIX, APPX, or bundle (`.msix`, `.appxbundle`, `.msixbundle`).
3. The package requires a Microsoft work or school account to download.

The most common case in practice is OEM UWP apps and Store-only utilities. Examples:

```
DellInc.DellSupportAssistforPCs       (MSIX, msstore-only)
DellInc.DellCommandUpdate             (AppX, msstore-only)
DellInc.MyAlienware                   (msstore-only)
9PMHP03NJ9QP                          (msstore-only, Alienware Command Center supplemental)
Microsoft.WindowsAppRuntime.1.8       (sometimes works, sometimes not, depending on version)
```

For these apps, the recipe must use `direct-url` or `manual` as the source type.

> **What about a work or school account?**
> It is technically possible to authenticate to the Microsoft Store with a work or school account and download Store-only packages. But the authentication flow is interactive, tokens expire, and unattended builds break the moment a token expires. For a build system that is meant to be reproducible, this is the wrong trade-off. We treat Store-only packages as `manual` and document where the user can obtain them.

---

## Basic Usage

```powershell
winget download --id Microsoft.Edge --download-directory C:\Cache
```

Output:

```
Found Microsoft Edge [Microsoft.Edge] Version 154.0.4258.37
This application is licensed to you by its owner.
Microsoft is not responsible for, nor does it grant any licenses to, third-party packages.
Downloading https://download.microsoft.com/download/f0d34257-.../MicrosoftEdgeEnterpriseX64.msi
  ██████████████████████████████   159 MB /  159 MB
Successfully verified installer hash
Installer downloaded: C:\Cache\Microsoft Edge_154.0.4258.37_Machine_X64_wix_en-US.msi
```

The tool does not print a JSON output, so the build tool wraps it and inspects the file system afterward.

### Minimal unattended invocation

For scripts, always pass both acceptance flags. Without them, `winget` will prompt and hang:

```powershell
winget download `
    --id $WingetId `
    --download-directory $StagingDir `
    --accept-source-agreements `
    --accept-package-agreements `
    --disable-interactivity
```

- `--accept-source-agreements` — accepts the `winget` source terms.
- `--accept-package-agreements` — accepts the package's license terms.
- `--disable-interactivity` — fails fast instead of prompting.

---

## Finding the Winget ID for an App

Three methods, in order of speed:

### 1. Search from the command line

```powershell
winget search "Dell Command Update"
```

Output:

```
Name                                    Id                              Version   Source
--------------------------------------------------------------------------------------
Dell Command | Update                   Dell.CommandUpdate              5.5.0     winget
Dell Command | Update (Universal)       Dell.CommandUpdate.Universal    5.5.0     winget
```

The `Id` column is what goes in the recipe's `WingetId` field.

### 2. Search on `winget.run`

[winget.run](https://winget.run/) indexes the same manifests and provides a searchable web interface. Useful when you are not on a Windows machine.

### 3. Search the manifest repository directly

The manifests live at [github.com/microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs). You can browse or grep them. Each manifest is a YAML file with an `Id` field.

---

## Verifying a Package Before Adding It to a Recipe

Before you add a `winget` source to a recipe, verify three things:

### 1. The package downloads

```powershell
$tmp = "$env:TEMP\winget-test"
New-Item -ItemType Directory -Path $tmp -Force | Out-Null

winget download `
    --id <WingetId> `
    --download-directory $tmp `
    --accept-source-agreements `
    --accept-package-agreements `
    --disable-interactivity

if ($LASTEXITCODE -ne 0) { "FAILED with exit code $LASTEXITCODE" }
```

If this succeeds, the package is fetchable.

### 2. The output is what you expected

```powershell
Get-ChildItem $tmp
```

You should see one file with an extension that matches the app's installer type (`.msi`, `.exe`, and so on). If you see a `.msix` or `.appxbundle`, the app is a Store-only package and `winget download` succeeded only because it is published in both sources — this will fail in a clean environment. Mark the recipe entry as `manual` instead.

### 3. The output name is deterministic

Run the download twice with a clean cache and compare the filenames:

```powershell
Remove-Item $tmp -Recurse -Force
New-Item -ItemType Directory -Path $tmp -Force | Out-Null
winget download --id <WingetId> --download-directory $tmp --accept-source-agreements --accept-package-agreements --disable-interactivity
Get-ChildItem $tmp | Select-Object Name
```

If the name changes between runs (it usually includes the version number), the build tool must glob the output rather than assume a fixed filename. The `Resolve-WingetPackage.ps1` wrapper handles this — it lists the folder after download and picks the single new file.

---

## Deterministic Output Names

`winget download` produces filenames in this pattern:

```
<ProductName>_<Version>_<Scope>_<Architecture>_<InstallerType>_<Locale>.<extension>
```

Examples:

```
Microsoft Edge_154.0.4258.37_Machine_X64_wix_en-US.msi
Google Chrome_154.0.8037.58_Machine_X64_wix_en-US.msi
Dell Command Update_5.5.0_Machine_X64_wix_en-US.exe
```

The name is deterministic for a given version but changes when the version changes. The build tool must not depend on a fixed name.

Instead, the wrapper script:

1. Records the directory contents before the download.
2. Runs `winget download`.
3. Records the directory contents after.
4. Picks the single new file.
5. Verifies its extension matches the expected type.
6. Moves it into the staging folder with the correct filename for the recipe.

The recipe does not need to know the winget-generated filename — it just needs to know the winget ID and the target path inside the archive.

---

## Common Flags

| Flag | Purpose | Required for this project? |
|---|---|---|
| `--id <id>` | The winget package ID. | Yes |
| `--download-directory <path>` | Where to write the installer. | Yes |
| `--accept-source-agreements` | Accept the winget source terms. | Yes — otherwise prompts |
| `--accept-package-agreements` | Accept the package license terms. | Yes — otherwise prompts |
| `--disable-interactivity` | Fail instead of prompting. | Yes — for unattended builds |
| `--version <version>` | Pin a specific package version. | No — recipe pins versions via the manifest, not the winget flag |
| `--architecture <arch>` | Force x64, x86, or arm64. | Sometimes — for multi-arch packages where x64 must be explicit |
| `--locale <locale>` | Force a specific locale. | Sometimes — for packages with locale-specific installers |

For the build tool, the standard invocation is:

```powershell
winget download `
    --id $WingetId `
    --download-directory $StagingDir `
    --accept-source-agreements `
    --accept-package-agreements `
    --disable-interactivity
```

Optional `--architecture` and `--locale` are passed when the recipe specifies them.

---

## Common Failure Modes

| Exit Code | Meaning | Cause | Fix |
|---|---|---|---|
| `0` | Success | — | — |
| `0x8A15002B` | No matching package found | ID is wrong or package was removed from the source | Re-search and update the recipe |
| `0x8A150014` | Installer hash mismatch | The vendor replaced the file at the URL without updating the manifest | Wait 24 hours and retry; if it persists, open an issue with the winget-pkgs repo |
| `0x8A15005E` | Package requires admin approval | Rare, usually for packages that install drivers | Use `direct-url` from the vendor instead |
| `0x8A15002E` | Download failed | Network issue, CDN rate limit, or the URL is dead | Retry; if persistent, check the vendor's status page |
| `0x8A150033` | Source requires authentication | MSStore-only package or work/school account requirement | Mark as `manual` in the recipe |
| `0x8A150013` | Package agreements not accepted | Missing `--accept-package-agreements` | Add the flag |
| `0x8A150010` | Source agreements not accepted | Missing `--accept-source-agreements` | Add the flag |

For a full list, run:

```powershell
winget download --help
```

Or see the [winget error codes documentation](https://learn.microsoft.com/en-us/windows/package-manager/winget/returnCodes).

---

## Fallback Strategies

When `winget download` does not work for an app, the recipe must fall back to one of the other source types.

### `direct-url`

Used when the vendor publishes the installer at a stable, documented HTTPS URL.

Requirements:
- The URL must resolve without authentication.
- The URL must serve the current version (or a version the recipe explicitly pins).
- The recipe must record a SHA-256 so tampering or silent version changes are detected.

Example recipe entry:

```json
{
  "AppName": "Dell SupportAssist",
  "Source": "direct-url",
  "Url": "https://dl.dell.com/FOLDER06731690M/1/Dell-SupportAssist.exe",
  "Sha256": "a1b2c3..."
}
```

### `manual`

Used when neither `winget download` nor a stable direct URL exists.

Requirements:
- The recipe must include a `Notes` field describing exactly where the user obtains the file.
- The recipe must state the filename the user must place at the target path.
- The build tool must fail loudly if the file is missing, with a message that repeats the `Notes` field.

Example recipe entry:

```json
{
  "AppName": "DellInc.DellSupportAssistforPCs",
  "Source": "manual",
  "Notes": "MSIX bundle from the Microsoft Store. Download on a machine with a work or school account, then place the .msix at C:\\Recovery\\OEM\\Apps\\SupportAssist\\UWP\\SupportAssist_x64.msix before building.",
  "ExpectedFilename": "SupportAssist_x64.msix"
}
```

### `repo-asset`

Used when the file ships in the repository itself. This is only appropriate for small customization assets, not vendor installers.

---

## Recipe Source Type: `winget`

In a vendor recipe, a `winget` source entry looks like this:

```json
{
  "AppName": "Dell Command | Update for Windows Universal",
  "Source": "winget",
  "WingetId": "Dell.CommandUpdate.Universal",
  "WingetArchitecture": "x64",
  "WingetLocale": "en-US"
}
```

Fields:

| Field | Required? | Purpose |
|---|---|---|
| `AppName` | Yes | Must match the framework manifest's `AppName` for this app |
| `Source` | Yes | Must be `"winget"` |
| `WingetId` | Yes | The winget package ID |
| `WingetArchitecture` | No | Forces `--architecture`. Use only for multi-arch packages. |
| `WingetLocale` | No | Forces `--locale`. Use only for locale-specific installers. |

The build tool reads this entry, invokes `Resolve-WingetPackage.ps1`, and stages the resulting file at the path the framework manifest declares for `AppName`.

---

## Automation Notes

### Caching

The build tool caches downloaded installers by SHA-256 or URL. If a file with the expected hash already exists in `cache/`, the download is skipped. This means:

- Rebuilding an unchanged vendor is fast.
- Changing one app in a ten-app recipe re-downloads only that one app.
- A network outage mid-build does not require re-downloading what already succeeded.

The cache lives at `cache/` at the repository root and is gitignored. See [CONTRIBUTING.md](../CONTRIBUTING.md) for the cache layout.

### Retry logic

The wrapper script retries downloads with exponential backoff:

- Up to 3 attempts
- Initial delay 5 seconds, multiplied by 1.5 each retry, capped at 30 seconds
- Hash verification on each attempt

If all attempts fail, the script throws and the build stops. The user can re-run and the cache preserves what succeeded.

### Hash verification

`winget download` verifies the installer hash against the manifest it fetched. The build tool does **not** re-verify — winget's verification is sufficient for `winget` sources. For `direct-url` sources, the build tool performs its own SHA-256 check against the hash recorded in the recipe.

### Logging

`winget download` output goes to the console. The build tool captures it and writes it to the build log at `C:\ProgramData\MDT-OEM-Extensibility\Logs\<Vendor>.log` (or the path configured via `-LogPath`).

---

## Related Documentation

- [docs/BUILDING-PACKS.md](BUILDING-PACKS.md) — full build workflow
- [docs/ADDING-A-VENDOR.md](ADDING-A-VENDOR.md) — how to author a vendor recipe
- [docs/RECIPE-SCHEMA.md](RECIPE-SCHEMA.md) — complete recipe field reference
- [docs/TROUBLESHOOTING.md](TROUBLESHOOTING.md) — general troubleshooting
- [winget documentation](https://learn.microsoft.com/en-us/windows/package-manager/winget/) — Microsoft's own reference
- [winget.run](https://winget.run/) — web-based package search
- [github.com/microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs) — manifest repository
