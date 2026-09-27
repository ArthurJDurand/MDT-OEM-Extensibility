# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Versioning Policy

Because this repository ships build tooling and vendor recipes rather than runtime code, version numbers have specific meaning for users:

| Bump | Meaning | Examples |
|---|---|---|
| **MAJOR** | Breaking changes to the recipe schema, the tool interface, or the archive layout that require users to update their recipes or rebuild existing archives. | New required recipe field, renamed tool parameter, changed archive structure |
| **MINOR** | New vendors, new tool capabilities, new asset types, or schema extensions that are backward-compatible. Existing recipes continue to build. | A new vendor recipe, a new source type, a new map builder |
| **PATCH** | Corrections to recipe URLs, hashes, and documentation. Safe to pull at any time. | A fixed download URL, an updated SHA-256, a corrected typo |

**When upgrading across MAJOR versions, read the migration notes in the release description.**

---

## [Unreleased]

### Added

### Changed

### Fixed

### Removed

### Security

---

## [1.0.0] - YYYY-MM-DD

Initial public release.

### Added

#### Build Tooling (`tools/`)

- `Build-OEMPack.ps1` — Main entry point. Reads a vendor's `Recipe.json`, resolves every app from its declared source, stages the assets, and packs everything into `<Vendor>.7z`.
- `New-OEMAppPack.ps1` — Packs a staging folder into a `.7z` archive using 7-Zip with the correct structure and compression settings.
- `Test-OEMAppPack.ps1` — Validates a built archive against the framework manifest to ensure every app the framework expects is present at the expected path.
- `Resolve-WingetPackage.ps1` — Wraps `winget download` for MSI and EXE packages. Handles exit codes, retries, and staging into the correct subfolder.
- `Resolve-OEMUrl.ps1` — Direct download with retry and SHA-256 verification. Used for apps the vendor publishes at stable URLs.
- `Build-DellWinPEMap.ps1` — Parses Dell's `DriverPackCatalog.cab`, filters for WinPE driver packages, and emits a map keyed by WinPE generation (WinPE3 through WinPE11).
- `Build-HPWinPEMap.ps1` — Parses the HP Client Windows PE Driver Packs page and emits a map keyed by WinPE family.
- `Build-LenovoWinPEMap.ps1` — Parses Lenovo's `recipecard.json`, resolves each WinPE pack to its direct download URL, and emits a map keyed by machine type.

#### Vendor Recipes (`vendors/`)

- **Dell** — Recipe covering all applications declared in the framework's `Manifests/Dell.json`:
  - Alienware Command Center (v5 and v6)
  - Dell Command Update (Universal and UWP)
  - Dell Optimizer
  - Dell Precision Optimizer
  - Dell Power Manager Service
  - Dell SupportAssist (desktop and UWP)
  - DellInc.MyAlienware
  - Fusion Service
  - Microsoft Windows App Runtime and .NET Desktop Runtime prerequisites
- **HP**, **Lenovo**, **ASUS**, **Acer**, **MSI**, **Gigabyte**, **Dynabook**, **Huawei**, **Microsoft**, **Proline** — placeholder folders with `Recipe.json` scaffolds. Recipes to be completed as download sources are validated.

#### Vendor Assets (`vendors/<Vendor>/Assets/`)

- **Dell** — customization assets committed to the repository:
  - `Customizations.ps1` — the Dell pre-install customization script
  - `csup.txt`, `gpsFix.reg`, `OEMinfo.reg`, `unattend.xml`
  - Wallpaper and theme assets for the default and G-series families
- Other vendors: assets to be added alongside their recipes.

#### Schema (`schema/`)

- `vendor-recipe.schema.json` — JSON Schema for the vendor recipe format, covering all four source types (`winget`, `direct-url`, `repo-asset`, `manual`).
- `winpe-map.schema.json` — JSON Schema for the WinPE map files emitted by the map builders.

#### Documentation (`docs/`)

- `README.md` — project overview, workflow, structure, quick start.
- `CONTRIBUTING.md` — contribution guidelines, script standards, recipe standards, asset standards.
- `CODE_OF_CONDUCT.md` — Contributor Covenant Code of Conduct.
- `LICENSE` — MIT License.
- `CHANGELOG.md` — this file.
- `docs/BUILDING-PACKS.md` — planned: how to build a vendor pack end to end.
- `docs/ADDING-A-VENDOR.md` — planned: how to add a new OEM, with Dell as the worked example.
- `docs/RECIPE-SCHEMA.md` — planned: reference for every recipe field.
- `docs/WINGET-DOWNLOAD.md` — planned: when `winget download` works and when it does not.
- `docs/TROUBLESHOOTING.md` — planned: common errors and fixes.

#### Repository Infrastructure

- `.gitignore` — ignore rules that prevent vendor installers, downloaded caches, and built archives from being committed.
- `.github/FUNDING.yml` — GitHub Sponsors configuration.
- `.github/release.yml` — auto-categorized release notes from pull requests.
- `.github/ISSUE_TEMPLATE/bug_report.yml` — bug report template.
- `.github/ISSUE_TEMPLATE/feature_request.yml` — feature request template.
- `.github/PULL_REQUEST_TEMPLATE.md` — pull request template.

### Security

- Documented that vendor installers, drivers, and firmware updates are never committed. Every binary is fetched from the vendor's official channel at build time and verified against a recorded hash.
- The `.gitignore` excludes common secret patterns (`*productkey*`, `*serial*`, `*.pfx`, `*.p12`, and so on) to reduce accidental commits of sensitive data.
- Recipes record SHA-256 hashes for `direct-url` sources so tampering or silent version changes are detected.
- The `winget download` wrapper verifies installer hashes where Microsoft exposes them.

### Known Limitations at Release

- Dell is the only vendor with a complete recipe. The remaining ten vendors are scaffolds awaiting download source validation.
- Some Dell apps are declared as `manual` in the recipe because no automatic fetch path exists today (UWP MSIX packages such as `DellInc.DellSupportAssistforPCs`). Users must supply these files manually.
- `winget download` does not work for UWP/MSStore-only packages without a work or school account. Recipes mark these as `manual`.
- The WinPE map builders depend on vendor catalogs that change format without notice. If a catalog is restructured, the corresponding builder will need updating.
- No archive signature verification beyond per-file SHA-256 is implemented today. Future releases may add signed manifest support.

---

[Unreleased]: https://github.com/ArthurJDurand/MDT-OEM-Extensibility/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/ArthurJDurand/MDT-OEM-Extensibility/releases/tag/v1.0.0
