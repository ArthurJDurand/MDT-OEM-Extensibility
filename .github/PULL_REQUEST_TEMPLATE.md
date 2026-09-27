<!--
  Thanks for submitting a pull request! Before you continue, please:

  1. Read CONTRIBUTING.md — especially the Script, Recipe, and Asset Standards.
  2. Ensure your branch is based on the latest `main`.
  3. Make sure all applicable checkboxes below are ticked.

  For large or architectural changes — new vendors beyond what is already
  scaffolded, recipe schema changes, or new source types — please open an
  issue or a Discussion first so we can agree on the approach before you
  invest significant time.
-->

## Summary

<!--
  One or two sentences describing WHAT this PR changes and WHY.
  Link the related issue if one exists.
-->

Closes #

## Type of Change

<!--
  Tick all that apply. This informs the reviewer and the auto-generated
  release notes (see .github/release.yml).
-->

- [ ] 🐛 Bug fix (non-breaking change that fixes an issue)
- [ ] 🚀 New tool or capability
- [ ] ⚠️ Breaking change (fix or feature that changes the recipe schema, tool interface, or archive layout; may require users to update existing recipes and rebuild archives)
- [ ] 📦 New vendor recipe
- [ ] 🔧 Vendor recipe update (URL, hash, version)
- [ ] 🎨 Vendor asset (wallpaper, customization script, registry file)
- [ ] 🗺️ WinPE map builder change
- [ ] 📝 Documentation only (no tool or recipe changes)
- [ ] 🧹 Refactor or maintenance (no functional change)
- [ ] 🔒 Security fix

## Affected Area

<!--
  Which parts of the project does this PR touch? Tick all that apply.
  This helps reviewers focus and helps users know what to rebuild or re-test.
-->

- [ ] `tools/` — build tooling (PowerShell scripts)
- [ ] `vendors/<Vendor>/Recipe.json` — a vendor recipe
- [ ] `vendors/<Vendor>/Assets/` — committed customization assets
- [ ] `vendors/<Vendor>/Overrides/` — manual overrides for a vendor
- [ ] `schema/` — JSON schema for recipes or WinPE maps
- [ ] `docs/` — documentation
- [ ] `README.md` — project README
- [ ] Repository infrastructure (`.github/`, `LICENSE`, `CHANGELOG.md`, `.gitignore`)

## Vendor

<!--
  If this PR touches a specific vendor, name it here.
  If it touches multiple vendors or is vendor-agnostic, say so.
-->

- Vendor(s): <!-- e.g. Dell, or "multiple", or "not applicable" -->

## Changes Made

<!--
  A bullet list of the specific changes. Keep it scannable.
  Reference specific files, recipes, or tool functions where relevant.
-->

-
-
-

## Testing

<!--
  Describe exactly how you tested this. "It works" is not enough.
  The more detail, the easier it is to review with confidence.
-->

**Build host:**

<!--
  Example: Windows 11 24H2 (build 26100.1742), PowerShell 7.4.5, winget v1.9.25200, 7-Zip 23.01
-->

**Vendors built and validated:**

<!--
  Example:
  - Dell: built end-to-end, validated with Test-OEMAppPack.ps1 against Manifests/Dell.json
  - HP: partial build, failed on HP Support Assistant (see Known Issues below)
-->

**Testing steps performed:**

<!--
  Example:
  1. Ran .\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath C:\Test
  2. All 14 apps downloaded and staged successfully
  3. Ran Test-OEMAppPack.ps1 -Archive C:\Test\Dell.7z -Manifest ...\Manifests\Dell.json
  4. Validator reported all 14 apps present at the expected paths
  5. Inspected the archive contents with 7z l and confirmed the folder structure
  6. (If applicable) Deployed the rebuilt archive to a test machine and confirmed
     the framework extracted and installed correctly
-->

1.
2.
3.

**Result:**

<!--
  Example: Build completed in 6 minutes 12 seconds on a cached run. Archive
  SHA-256: a1b2c3d4e5f6... Reproducible across three consecutive runs.
-->

**Archive hash (if a vendor pack was rebuilt):**

<!--
  Providing the SHA-256 of the built archive lets reviewers verify
  reproducibility. Skip this if the PR does not produce a rebuilt archive.
-->

- `SHA-256`: <!-- e.g. a1b2c3d4e5f6... -->

## Download Source Verification

<!--
  For PRs that add or update recipe entries, confirm that each source was
  verified against the vendor's official channel. This is the single most
  important check for recipe PRs.
-->

- [ ] Every new or updated `winget` entry was tested with `winget download --id <id>` and produced a valid installer
- [ ] Every new or updated `direct-url` entry was verified to download successfully and the SHA-256 in the recipe matches the downloaded file
- [ ] Every `manual` entry has a `Notes` field explaining where the user must obtain the file
- [ ] No vendor installers, drivers, or firmware were committed to the repository
- [ ] No personal or machine-specific values (product keys, credentials, local paths) are present in any committed file

## Known Issues / Limitations

<!--
  Anything that doesn't work, needs follow-up, or is known to be a
  partial implementation. Be honest — reviewers would rather know now.
-->

- None.

## Screenshots or Logs

<!--
  If this PR adds a new tool or changes tool output, include a terminal
  screenshot or an excerpt of the new output. For recipe PRs, include the
  relevant `winget download` or HTTP response output if the source was
  recently verified. Drag and drop images directly into this box. For logs,
  attach as a file or upload to a Gist — do not paste long logs inline.
-->

## Checklist

<!--
  Every box must be ticked before a reviewer will look at the PR.
  If a box doesn't apply, explain why in the Notes section below.
-->

### Script Standards (see CONTRIBUTING.md)

- [ ] Every script includes a `.SYNOPSIS` / `.DESCRIPTION` / `.PARAMETER` / `.EXAMPLE` / `.NOTES` header block
- [ ] Scripts use `[CmdletBinding()]` with explicit parameter definitions
- [ ] No hardcoded user paths — paths are parameters with sensible defaults
- [ ] Downloads include retry logic with exponential backoff
- [ ] Downloads include SHA-256 hash verification (or warn and record the observed hash when the recipe does not yet pin one)
- [ ] Scripts preserve `$LASTEXITCODE` and surface failures to the caller — no silent `try/catch` swallowing
- [ ] Scripts clean up staging folders on failure via `try/finally`
- [ ] Long-running operations report progress via `Write-Progress` or stage announcements
- [ ] Scripts are PowerShell 5.1 compatible unless the header explicitly documents a PS7 requirement
- [ ] No `exit` in scripts intended to be dot-sourced or called from another script
- [ ] Scripts are idempotent — a second run on a fully cached vendor skips downloads and produces identical output

### Recipe Standards (see CONTRIBUTING.md)

- [ ] The recipe contains an entry for every app the framework manifest declares
- [ ] Every app declares exactly one source type (`winget`, `direct-url`, `repo-asset`, or `manual`)
- [ ] Every `direct-url` entry records a SHA-256 (or the PR description explains why not yet)
- [ ] `manual` is used only when no automatic fetch path exists, with a `Notes` field describing where to obtain the file
- [ ] Recipe entries are ordered to match the framework manifest for easier diff review

### Asset Standards (see CONTRIBUTING.md)

- [ ] No vendor installers, drivers, firmware, or other binaries are committed
- [ ] Wallpapers and theme assets are under 10 MB per file
- [ ] Text files are UTF-8 without BOM (except `.reg` files, which are UTF-16 LE)
- [ ] No personal or machine-specific values (product keys, credentials, local paths) are present
- [ ] File names are descriptive and follow the existing convention

### Repository Hygiene

- [ ] `CHANGELOG.md` updated under `[Unreleased]` in the appropriate section (`Added`, `Changed`, `Fixed`, `Removed`, `Security`)
- [ ] No downloaded caches, staging folders, or built `.7z` archives are committed
- [ ] No credentials, product keys, or secrets committed
- [ ] No unrelated files, formatting changes, or drive-by refactors included
- [ ] Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/) (`feat(vendor): …`, `fix(tool): …`, etc.)
- [ ] Branch is based on the latest `main` and rebased if necessary
- [ ] PR title matches the Conventional Commits format

### Documentation

- [ ] User-facing documentation updated (`README.md` or `docs/*`) if the workflow changed
- [ ] If a new vendor was added, `vendors/<Vendor>/README.md` describes the vendor's build notes
- [ ] If a new tool was added, it is listed in the README's repository structure and referenced from the relevant doc
- [ ] If a recipe schema field was added, `docs/RECIPE-SCHEMA.md` documents it (or the PR notes that the doc update is a follow-up)
- [ ] If a source type was added, `docs/WINGET-DOWNLOAD.md` (or the equivalent doc) describes when to use it

### Testing

- [ ] Built at least one vendor archive end-to-end on a real Windows host
- [ ] Ran `Test-OEMAppPack.ps1` against the framework manifest and confirmed all apps are present at the expected paths
- [ ] No regressions observed in adjacent vendors or tools
- [ ] If the change affects a network-dependent path, tested from a network that reaches the vendor's CDN

## Notes for Reviewers

<!--
  Anything else the reviewer should know. Call out tricky parts, areas
  you're unsure about, or questions you'd like feedback on.
-->

- None.

---

<!--
  By submitting this pull request, you agree that your contribution is
  licensed under the MIT License that governs this project. See
  CONTRIBUTING.md → "License of Contributions" for details.
-->
