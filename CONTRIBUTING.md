# Contributing to MDT OEM Extensibility

First off — **thank you** for considering a contribution. This project exists because the deployment community shares knowledge, and every improvement (a new vendor recipe, a fixed download URL, a validated `winget download` path) makes it more useful for everyone.

This document explains how to contribute effectively and what standards your contributions should meet.

---

## Table of Contents

- [Ways to Contribute](#ways-to-contribute)
- [Before You Start](#before-you-start)
- [Development Setup](#development-setup)
- [Script Standards](#script-standards)
- [Recipe Standards](#recipe-standards)
- [Asset Standards](#asset-standards)
- [Commit Message Convention](#commit-message-convention)
- [Branch Naming](#branch-naming)
- [Pull Request Process](#pull-request-process)
- [What Reviewers Look For](#what-reviewers-look-for)
- [Areas Where Help Is Needed](#areas-where-help-is-needed)
- [Reporting Bugs](#reporting-bugs)
- [License of Contributions](#license-of-contributions)
- [Code of Conduct](#code-of-conduct)
- [Questions](#questions)

---

## Ways to Contribute

You don't have to write code to contribute. All of the following are valuable:

| Contribution Type | Examples |
|---|---|
| **New vendor recipes** | A `Recipe.json` for a vendor not yet covered, with working download sources for every app |
| **Recipe updates** | Correcting a broken URL, updating a version number, adding a SHA-256 for verification |
| **`winget` coverage** | Confirming that an app downloads via `winget download`, or documenting that it does not |
| **Customization assets** | Wallpapers, `.reg` files, vendor-branded scripts, theme files |
| **Tooling** | Improvements to `Build-OEMPack.ps1`, `Resolve-WingetPackage.ps1`, or the map builders |
| **Documentation** | Clarifying a step, adding per-vendor build notes, adding a troubleshooting entry |
| **Testing** | Confirming a recipe works on your build host and reporting the result |
| **Ideas** | Feature requests, workflow suggestions, architectural feedback |

If you're unsure whether an idea is in scope, **open a [Discussion](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/discussions) first** before investing time in a PR.

---

## Before You Start

### Check existing issues and PRs

Someone may already be working on the same vendor or the same tool. Search [open issues](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/issues) and [open PRs](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/pulls) before starting.

### Open an issue first for large changes

For anything beyond a broken URL fix — especially new vendors, new recipe fields, or changes to the recipe schema — **open an issue to discuss the approach first**. This avoids wasted effort if the change does not fit the project's direction.

### Small, focused PRs are preferred

One logical change per PR. A PR that fixes a typo *and* adds a recipe *and* refactors a tool is hard to review and hard to revert if something goes wrong.

### Vendors are one-PR-at-a-time

Add one vendor per PR. Adding five vendors in a single PR makes review impossible. Start with the one that is most valuable to you, get it merged, then add the next.

---

## Development Setup

To test your changes, you need a build host that can actually fetch content.

### Minimum requirements

- **Windows 10 or Windows 11** build host (physical or virtual)
- **PowerShell 5.1 or 7** on PATH
- **7-Zip** installed at `C:\Program Files\7-Zip\7z.exe`
- **`winget` (App Installer)** available on PATH
- Network access to the vendor's CDN and Microsoft's winget endpoint
- At least **10 GB free disk space** for the download cache

### Recommended workflow

1. Fork the repository
2. Clone your fork:
   ```bash
   git clone https://github.com/<your-username>/MDT-OEM-Extensibility.git
   ```
3. Make your changes
4. Test end-to-end:
   - Add or modify a vendor recipe
   - Run the appropriate tool (`Build-OEMPack.ps1`, `Resolve-WingetPackage.ps1`, or a map builder)
   - Verify the output archive contains what the framework manifest expects
   - If possible, run a deployment with the rebuilt archive
5. Commit and push to your fork
6. Open a PR against the `main` branch

### Testing requirements by change type

| Change Type | Testing Required |
|---|---|
| Documentation only | None, but proofread carefully |
| Typo fix in a script comment | None, but ensure nothing else changed |
| Tool bug fix | Reproduce the bug, apply the fix, verify it is resolved |
| New vendor recipe | Build the full archive end-to-end and validate with `Test-OEMAppPack.ps1` |
| URL update in an existing recipe | Re-download that app and confirm the file matches the expected hash |
| New customization asset | Verify the asset lands at the correct path in the built archive |
| Schema change | Update the schema, all existing recipes, and the tools that consume them |
| Map builder change | Run the builder, compare the output against the current map, verify no regressions |

**State what you tested on in the PR description.** "Built `Dell.7z` on Windows 11 24H2 with PowerShell 7.4, validated against `Manifests/Dell.json`" is far more useful than "tested and works."

---

## Script Standards

PowerShell scripts live in `tools/`. They must run on a stock Windows host without PowerShell 7 unless explicitly documented.

### 1. Header documentation

Every script must begin with a comment block containing at minimum:

```powershell
<#
.SYNOPSIS
    One-line description of what the script does.

.DESCRIPTION
    Longer explanation including when this script runs in the build workflow
    and what preconditions it expects.

.PARAMETER <Name>
    Description of each parameter.

.EXAMPLE
    .\Build-OEMPack.ps1 -Vendor Dell -OutputPath \\SERVER\Shared\OEM\x64

.NOTES
    - PowerShell 5.1 compatible: yes/no
    - External dependencies (7-Zip, winget, network access)
    - Known limitations
#>
```

### 2. Parameter blocks

Use `[CmdletBinding()]` and explicit parameter definitions. Avoid positional-only arguments for anything non-obvious.

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $true)]
    [ValidateSet('Dell','HP','Lenovo','ASUS','Acer','MSI','Gigabyte','Dynabook','Huawei','Microsoft','Proline')]
    [string]$Vendor,

    [Parameter(Mandatory = $false)]
    [string]$OutputPath = (Get-Location).Path,

    [switch]$Force
)
```

### 3. No hardcoded user paths

Accept paths as parameters with sensible defaults. Never hardcode a specific user's profile or drive.

```powershell
# BAD
$cacheDir = "F:\Downloads\Cache"

# GOOD
$cacheDir = if ($CachePath) { $CachePath } else { Join-Path $PSScriptRoot '..\cache' }
```

### 4. Retry logic for downloads

Vendor CDNs fail intermittently. Wrap every download in a retry loop with exponential backoff.

```powershell
$maxAttempts = 3
$delay = 5
for ($attempt = 1; $attempt -le $maxAttempts; $attempt++) {
    try {
        Invoke-WebRequest -Uri $url -OutFile $dest -UseBasicParsing -ErrorAction Stop
        break
    } catch {
        if ($attempt -eq $maxAttempts) { throw }
        Start-Sleep -Seconds $delay
        $delay = [math]::Min($delay * 1.5, 30)
    }
}
```

### 5. Hash verification

Every downloaded file must be verified against a recorded hash. If the recipe does not record a hash, the tool must emit a warning and record the observed hash so the next run can enforce it.

```powershell
$observed = (Get-FileHash -Path $dest -Algorithm SHA256).Hash
if ($expected -and $observed -ne $expected) {
    throw "Hash mismatch for $name. Expected $expected, got $observed."
}
```

### 6. Preserve exit codes

Never mask a failure with a silent `try/catch` that swallows the error. If a script fails, the caller should know.

```powershell
# BAD — swallows the error
try { & winget download --id $id } catch { }

# GOOD — check the exit code explicitly
& winget download --id $id --download-directory $staging --accept-source-agreements --accept-package-agreements
if ($LASTEXITCODE -ne 0) {
    throw "winget download failed with exit code $LASTEXITCODE for $id"
}
```

### 7. Cleanup on failure

Any script that creates staging folders, mounts archives, or writes partial output must clean up on failure using `try/finally`.

```powershell
try {
    New-Item -Path $staging -ItemType Directory -Force | Out-Null
    # ... stage files ...
}
finally {
    if (Test-Path $staging -and -not $KeepStaging) {
        Remove-Item -Path $staging -Recurse -Force -ErrorAction SilentlyContinue
    }
}
```

### 8. Progress output

Long-running operations should report progress. Use `Write-Progress` for steps that take more than a few seconds, or `Write-Host` for stage announcements.

```powershell
Write-Progress -Activity "Building $Vendor pack" -Status "Downloading $AppName" -PercentComplete $percent
```

### 9. PowerShell 5.1 compatibility

Scripts must run under the Windows version of PowerShell, which is **5.1**. Do not use syntax or cmdlets exclusive to PowerShell 7 (ternary operator `? :`, `??`, `-Parallel`) unless the script explicitly requires PS7 and documents that in its header.

### 10. No `exit` in library scripts

Scripts meant to be dot-sourced or called from another script should **return** rather than `exit`. Standalone scripts may `exit` with a code.

### 11. Idempotence

Scripts that can be re-run must be idempotent. A second run of `Build-OEMPack.ps1` on a fully cached vendor must skip downloads and produce identical output.

---

## Recipe Standards

`Recipe.json` files live under `vendors/<Vendor>/`. They are the source of truth for what the build tool fetches.

### 1. Every recipe must match the framework manifest

The recipe must contain an entry for every app the deployment framework's `Manifests/<Vendor>.json` declares. The build tool validates this and fails if the two are out of sync.

### 2. Source types must be explicit

Every app must declare exactly one source type:

| Source | Used When |
|---|---|
| `winget` | The app is available via `winget download` as an MSI or EXE |
| `direct-url` | The vendor publishes a stable download URL |
| `repo-asset` | The file ships in `vendors/<Vendor>/Assets/` |
| `manual` | The app cannot be fetched automatically and the user must supply it |

Do not use `"Source": "auto"` or similar. Explicit is better.

### 3. Record hashes wherever possible

For `direct-url` sources, always record a SHA-256 in the recipe. If you cannot verify the hash on the first commit, record it after the first successful download and open a follow-up PR.

### 4. Version pins are optional

If the vendor URL is version-independent (e.g. `https://downloads.dell.com/.../DellSupportAssist.exe` always serves the latest), leave the version field out. If the URL is version-specific, pin the version so the recipe is self-documenting.

### 5. Use `manual` sparingly

`manual` is a last resort. It forces every user of the recipe to supply the file themselves. Before using it, verify that `winget download` does not work, that the vendor does not publish a stable URL, and that no enterprise catalog (Dell Command Deploy, HP CMSL, Lenovo System Update) exposes the file.

If you do use `manual`, include a `Notes` field describing exactly where the user can obtain the file and where it must be placed.

### 6. Keep entries ordered to match the framework manifest

This makes diff review easier. The build tool does not require the order to match, but reviewers do.

---

## Asset Standards

Assets live under `vendors/<Vendor>/Assets/`. They are committed to the repository.

### 1. What belongs in Assets

- Customization scripts (`Customizations.ps1` and similar)
- Registry files (`.reg`)
- Configuration files (`.xml`, `.json`, `.txt`, `.ini`)
- Wallpapers and theme assets (`.jpg`, `.png`, `.bmp`, `.theme`)
- Vendor-specific `unattend.xml` overlays
- Any other file that must ship inside the vendor `.7z` archive but is not downloaded from the vendor

### 2. What does NOT belong in Assets

- Vendor installers (`.exe`, `.msi`, `.msix`, `.appx`)
- Driver packages
- Firmware updates
- Any file over 10 MB unless it is a wallpaper or theme

If you think a binary belongs in Assets, open an issue first and explain why the recipe approach does not work.

### 3. File size discipline

Wallpapers are the only routinely large asset. Keep each wallpaper under 10 MB. Prefer JPG over PNG for photographic content. If you need to ship a wallpaper above 10 MB, explain why in the PR.

### 4. Encoding and line endings

- Text files use UTF-8 without BOM (except `.reg` files, which must use UTF-16 LE)
- Line endings follow the repository's `.gitattributes` convention
- No trailing whitespace

### 5. No personal or machine-specific values

Never commit an asset that contains:

- Product keys
- Credentials
- Machine-specific names or paths
- Personal organization names

If a template needs a placeholder, use an obvious one like `COMPANY-NAME` and document it.

---

## Commit Message Convention

This project uses [Conventional Commits](https://www.conventionalcommits.org/). This makes the changelog easier to generate and clarifies what each commit does.

### Format

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

### Types

| Type | Use For |
|---|---|
| `feat` | A new feature, recipe, asset, or tool |
| `fix` | A bug fix (broken URL, wrong hash, wrong path) |
| `docs` | Documentation changes only |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `chore` | Maintenance (dependency bumps, formatting, build changes) |
| `revert` | Reverting a previous commit |

### Scopes (optional but recommended)

- Vendor names in lowercase: `dell`, `hp`, `lenovo`, `asus`, `acer`, `msi`, `gigabyte`, `dynabook`, `huawei`, `microsoft`, `proline`
- Tool names: `build-oempack`, `winget-resolve`, `winpe-map`
- `schema`, `docs`, `readme`, `changelog`

### Examples

```
feat(dell): add SupportAssist and Command Update recipes
fix(hp): update WinPE map URL for HP Client Driver Packs
docs(schema): document the manual source type
refactor(build-oempack): extract download logic into Resolve-OEMUrl.ps1
feat(lenovo): add ThinkPad T14 Gen 4 recipe
```

### Breaking changes

If a change requires users to rebuild their archives or update existing recipes, add `!` after the type/scope and include a `BREAKING CHANGE:` footer:

```
feat(schema)!: require Sha256 for all direct-url sources

BREAKING CHANGE: Recipes without a Sha256 field now fail validation.
Every existing direct-url recipe must be updated with the hash of the
current download before merging this change.
```

---

## Branch Naming

Use a prefix that matches the commit type:

| Prefix | Use For |
|---|---|
| `feature/` | New features, recipes, or tools |
| `fix/` | Bug fixes |
| `docs/` | Documentation changes |
| `refactor/` | Code refactoring |
| `chore/` | Maintenance |

Examples:

- `feature/dell-supportassist-recipe`
- `fix/hp-winpe-map-url`
- `docs/recipe-schema-manual-source`
- `feature/lenovo-thinkpad-t14-gen4`

---

## Pull Request Process

1. **Fork** the repository
2. **Create a branch** from `main` using the naming convention above
3. **Make your changes** following the script, recipe, and asset standards
4. **Update `CHANGELOG.md`** — add your change under `[Unreleased]` in the appropriate section
5. **Test end-to-end** — build the archive, validate it against the framework manifest
6. **Push** to your fork
7. **Open a PR** against `main` with:
   - A clear title matching the Conventional Commits format
   - A description of **what** changed and **why**
   - The **Windows build** and **PowerShell version** you tested on
   - A **link to the related issue** if one exists
   - A **hash of the built archive** so reviewers can verify reproducibility

### PR title format

Match your commit message convention:

```
feat(dell): add SupportAssist and Command Update recipes
```

### What makes a good PR description

```markdown
## Summary
Adds a complete recipe for Dell that covers all apps the framework's
`Manifests/Dell.json` declares.

## Changes
- Added `vendors/Dell/Recipe.json`
- Added `vendors/Dell/Assets/Customizations.ps1`
- Added `vendors/Dell/Assets/wallpapers/` with three family variants
- Documented the recipe in `docs/ADDING-A-VENDOR.md`

## Testing
- Built `Dell.7z` on Windows 11 24H2 with PowerShell 7.4
- Validated against `Manifests/Dell.json` with `Test-OEMAppPack.ps1`
- All 14 apps present at the expected paths
- Two apps use `winget download`, three use `direct-url` with SHA-256 verification,
  one uses `manual` (DellSupportAssistforPCs UWP — no winget path)
- Archive SHA-256: `a1b2c3...`

## Related Issue
Closes #7

## Checklist
- [x] Follows recipe standards
- [x] CHANGELOG.md updated under [Unreleased]
- [x] Tested end-to-end
- [x] No vendor binaries committed
- [x] No personal or machine-specific values in assets
```

---

## What Reviewers Look For

When reviewing a PR, the maintainer checks:

| Item | Why |
|---|---|
| **Recipe matches framework manifest** | Missing or extra apps will break deployment |
| **Source types are explicit** | `auto` and similar are not allowed |
| **Hashes recorded for `direct-url`** | Without a hash, we cannot detect CDN tampering or version changes |
| **`manual` used only when necessary** | It shifts work onto every user |
| **No vendor binaries committed** | Licensing and repository size |
| **Assets under 10 MB per file** | Repository size |
| **Script header documentation** | SYNOPSIS, DESCRIPTION, PARAMETER, EXAMPLE, NOTES |
| **No hardcoded user paths** | Portability |
| **Retry logic on downloads** | Vendor CDNs are flaky |
| **Hash verification on downloads** | Detect tampering and version changes |
| **Exit code preservation** | Failures are visible to the caller |
| **Cleanup on failure** | No orphaned staging folders |
| **PowerShell 5.1 compatible** | Scripts must run on stock Windows |
| **CHANGELOG entry** | Added under `[Unreleased]` |
| **Small, focused scope** | One vendor per PR |

---

## Areas Where Help Is Needed

Some specific things the maintainer would love help with:

### 1. Complete vendor recipes

Dell is the reference. The following vendors need recipes built from scratch:

- **HP** — Support Assistant, Command Update, CMSL, HP Wolf Security
- **Lenovo** — Vantage, System Update, Commercial Vantage
- **ASUS** — MyASUS, Armoury Crate, ASUS Business Manager
- **Acer** — Acer Care Center, Quick Access
- **MSI** — Center, Dragon Center, Creator Center
- **Gigabyte** — Control Center, Smart Update
- **Dynabook** — Service Station
- **Huawei** — PC Manager
- **Microsoft** — Surface-specific utilities (Surface app, UEFI updates)
- **Proline** — the vendor's own utilities

If you have working download URLs for any of these, open a PR.

### 2. `winget download` coverage report

For every app in every vendor's manifest, confirm:
- Does `winget download --id <id>` work?
- If yes, what does the downloaded file look like?
- If no, is the reason a UWP/MSStore restriction, an authentication requirement, or a missing manifest?

This is a straightforward task that saves everyone time. Open a Discussion with your findings for one vendor.

### 3. Enterprise catalog integration

Dell Command Deploy, HP CMSL, and Lenovo System Update all expose catalog APIs that list driver and app download URLs. A tool that consumes these catalogs would eliminate URL scraping for the majority of driver packs. If you have experience with any of these catalogs, open a Discussion.

### 4. Map builder improvements

The three existing map builders (`Build-DellWinPEMap.ps1`, `Build-HPWinPEMap.ps1`, `Build-LenovoWinPEMap.ps1`) work but have limitations. Improvements welcome:

- Better error handling when a catalog is malformed
- Support for additional vendors with similar catalogs
- Schema validation of the emitted JSON
- Incremental updates (only refetch when the catalog changes)

### 5. Alternative sources for blocked apps

Some apps cannot be fetched automatically today. If you find an official, documented source for any of them, share it:

- Dell SupportAssist UWP (MSIX)
- Dell Command Update UWP (AppX)
- HP Support Assistant MSIX
- Lenovo Vantage (MSStore)
- Any other MSStore-only package

### 6. Documentation

- Per-vendor build notes as recipes are validated
- Screenshots of the build process
- A "known working network conditions" table (some CDNs behave differently by region)

If any of these interest you, **open a Discussion first** so we can scope it together.

---

## Reporting Bugs

Found a bug? [Open an issue](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/issues/new) with:

- **Description** — what you expected vs. what happened
- **Steps to reproduce** — exact command, exact vendor, exact app
- **Windows build and PowerShell version** — `winver` output and `$PSVersionTable.PSVersion`
- **Vendor and app** — which recipe entry failed
- **Relevant log file** — tool output, winget output, HTTP errors
- **Screenshots** if applicable

**Please don't paste full logs inline** — attach them as files or link to a Gist.

---

## License of Contributions

By submitting a pull request to this project, you agree that your contribution is licensed under the same [MIT License](LICENSE) that governs the project.

You confirm that:

- You have the right to submit the contribution
- The contribution is your original work, or you have obtained permission to submit it under the MIT License
- Any third-party code, scripts, or assets included in your contribution are compatible with the MIT License and clearly attributed
- Your contribution does not include vendor installers, drivers, or other binaries that you do not have the right to redistribute
- Your contribution does not include product keys, credentials, or other secrets that are not your own to share

---

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).

In short: be respectful, be patient, assume good faith, and focus on the technical problem. Harassment, personal attacks, and dismissive behavior are not tolerated. Violations can be reported to the maintainer via a [private GitHub security advisory](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/security/advisories/new).

---

## Questions?

- **General questions or ideas:** [GitHub Discussions](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/discussions)
- **Bug reports:** [GitHub Issues](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/issues)
- **Security vulnerabilities:** [Private security advisory](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/security/advisories/new)
- **Direct contact:** See the [author's GitHub profile](https://github.com/ArthurJDurand)

---

## Thank You

Whether you add a new vendor, fix a broken URL, or just confirm that a `winget download` path works — **your contribution matters**. This project is built on community knowledge, and every improvement helps someone build a vendor pack with less friction.

Thank you for being part of it.

---

<div align="center">

**Happy building!** 🛠️

</div>
