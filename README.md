<div align="center">

# MDT OEM Extensibility

**Build OEM-specific deployment payloads from official vendor sources — applications, drivers, wallpapers, and customizations — packaged for MDT Zero-Touch Deployment.**

[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%20%7C%207-5391FE?logo=powershell&logoColor=white)](https://github.com/PowerShell/PowerShell)
[![winget](https://img.shields.io/badge/winget-supported-0078D4)](https://learn.microsoft.com/en-us/windows/package-manager/winget/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE.md)
[![Sponsor](https://img.shields.io/badge/Sponsor-ArthurJDurand-ea4aaa?logo=github-sponsors&logoColor=white)](https://github.com/sponsors/ArthurJDurand)

[What This Is](#what-this-is) · [How It Fits](#how-it-fits-into-the-project) · [Repository Structure](#repository-structure) · [Quick Start](#quick-start) · [Documentation](#documentation) · [Support](#support-this-project) · [Contributing](#contributing)

</div>

---

## What This Is

A build system for the OEM payload archives that [`MDT-Zero-Touch-Deployment`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment) consumes at deployment time. It fetches installers and drivers from official vendor sources, stages committed customization assets, and packs everything into the `.7z` archives the deployment framework expects.

Three things this repository does not do:

- **Does not host vendor binaries.** Installers are downloaded from Dell, HP, Lenovo, or Microsoft's winget at build time. The repository ships only the tooling, the recipes that describe what to fetch, and the small customization assets (wallpapers, `.reg` files, scripts).
- **Does not run during deployment.** It runs once on the build host, before the deployment share is populated. Its output goes to `\\SERVER\Shared\OEM\<arch>\<Vendor>.7z`.
- **Does not manage the framework itself.** The OEM Apps framework, its modules, and its manifests live in the main deployment repository. This repository produces the content the framework extracts and installs.

### What Makes It Useful

- **Official sources only.** Every installer is fetched from the vendor's own CDN or Microsoft's winget. No third-party mirrors, no repackaged binaries, no ambiguous provenance.
- **Reproducible.** Every recipe records the source URL, version, and where possible a SHA-256 hash. Anyone can rebuild the same archive you shipped.
- **Incremental.** Downloads are cached by URL and hash. Updating one app in a ten-app pack does not re-download the other nine.
- **Deterministic output.** The `.7z` structure matches what the deployment framework expects, byte-for-byte, every time.
- **Vendor-agnostic.** Eleven OEMs are supported out of the box, with a documented path for adding more.

### Who This Is For

- Technicians running [`MDT-Zero-Touch-Deployment`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment) who need to populate the OEM payload archives
- Contributors adding support for a new OEM or updating an existing vendor's pack
- Anyone who wants to rebuild a vendor archive from scratch without downloading from unofficial sources

This repository assumes working knowledge of PowerShell, 7-Zip, the `winget` command-line tool, and the deployment repository's OEM content model. If any of those are unfamiliar, read the main repository's [`docs/OEM.md`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment/blob/main/docs/OEM.md) first.

---

## How It Fits into the Project

This is the third repository in a three-part project. Each part has a distinct lifecycle.

| Repository | Purpose | Changes When |
|---|---|---|
| [`MDT-Windows-Image-Builder`](https://github.com/ArthurJDurand/MDT-Windows-Image-Builder) | Builds the Windows `install.wim` used by MDT | Microsoft ships a new Windows build |
| [`MDT-Zero-Touch-Deployment`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment) | The MDT deployment share — task sequences, scripts, framework, OEM content structure | The deployment logic changes |
| **`MDT-OEM-Extensibility`** (this repo) | Builds the OEM payload archives the deployment consumes | A vendor ships a new version of an app or driver |

The flow:

```
MDT-Windows-Image-Builder
        │
        │ produces install.wim
        ▼
MDT-Zero-Touch-Deployment  ◄──── MDT-OEM-Extensibility
        │                               │
        │ expects                       │ produces
        │ \\SERVER\Shared\OEM\x64\*.7z  │ <Vendor>.7z archives
        │                               │
        └───────────────┬───────────────┘
                        ▼
                  PXE deployment to target machines
```

You do not need this repository to deploy. A deployment share populated with hand-built archives works fine. This repository exists to make archive construction repeatable, verifiable, and cheap.

---

## What the Payload Archives Contain

Each `<Vendor>.7z` is a self-contained payload that `ExtractOEMAppsx64.ps1` (or the x86 variant) extracts into `C:\Recovery\OEM` on the target machine. From there, `pre.ps1`, `Customizations.ps1`, and the OEM Apps framework consume it.

A Dell payload, for example, contains:

```
Dell.7z
├── Customizations.ps1                Vendor-specific pre-install script
├── csup.txt                          Vendor metadata consumed by SetupComplete
├── gpsFix.reg                        Registry tweaks
├── OEMinfo.reg                       OEM branding registry
├── unattend.xml                      OEM-attend overlay for PBR
├── OEM.7z                            Infrastructure extracted to C:\OEM
├── Customizations\
│   ├── Dell.7z                       Wallpapers and themes (default family)
│   └── G-series.7z                   Additional assets for the G-series family
└── Apps\
    ├── CommandCenter\
    │   ├── v5\Alienware-Command-Center-5-x-Full-Installer.exe
    │   └── v6\Alienware-Command-Center-Application-Full-Installer.exe
    ├── CommandUpdate\
    │   ├── Dell-Command-Update-Windows-Universal-Application.exe
    │   ├── PreinstallKit\windowsdesktop-runtime-10.0.11-win-x64.exe
    │   └── UWP\DellCommandUpdate.appxbundle
    ├── FusionService\
    ├── MyAlienware\
    ├── Optimizer\
    ├── PowerManagerService\
    ├── PrecisionOptimizer\
    └── SupportAssist\
```

The structure mirrors the `InstallerPath` values in the framework's `Manifests\<Vendor>.json`, so the framework finds each app exactly where it expects.

Every vendor follows the same pattern. What changes is the set of apps and the family-specific subfolders. See [`docs/BUILDING-PACKS.md`](docs/BUILDING-PACKS.md) for per-vendor details.

---

## Repository Structure

```
MDT-OEM-Extensibility/
├── README.md
├── LICENSE.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── .gitignore
├── .github/
│   ├── FUNDING.yml
│   ├── release.yml
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml
│       └── feature_request.yml
├── tools/
│   ├── Build-OEMPack.ps1              Main entry point — build one vendor's archive
│   ├── New-OEMAppPack.ps1             Pack a staging folder into <Vendor>.7z
│   ├── Test-OEMAppPack.ps1            Validate a built archive against the framework manifest
│   ├── Resolve-WingetPackage.ps1      Wrap `winget download`
│   ├── Resolve-OEMUrl.ps1             Direct download with hash verification and retry
│   ├── Build-DellWinPEMap.ps1         Build the Dell WinPE driver map
│   ├── Build-HPWinPEMap.ps1           Build the HP WinPE driver map
│   ├── Build-LenovoWinPEMap.ps1       Build the Lenovo WinPE driver map
│   └── README.md
├── vendors/
│   ├── Dell/
│   │   ├── Recipe.json                What to fetch, from where, at what version
│   │   ├── Assets/                    Committed customization assets
│   │   │   ├── Customizations.ps1
│   │   │   ├── csup.txt
│   │   │   ├── gpsFix.reg
│   │   │   ├── OEMinfo.reg
│   │   │   ├── unattend.xml
│   │   │   └── Wallpapers\
│   │   └── README.md
│   ├── HP/
│   ├── Lenovo/
│   ├── ASUS/
│   ├── Acer/
│   ├── MSI/
│   ├── Gigabyte/
│   ├── Dynabook/
│   ├── Huawei/
│   ├── Microsoft/
│   └── Proline/
├── cache/                             URL + hash keyed (gitignored)
├── schema/
│   ├── vendor-recipe.schema.json
│   └── winpe-map.schema.json
└── docs/
    ├── BUILDING-PACKS.md
    ├── ADDING-A-VENDOR.md
    ├── RECIPE-SCHEMA.md
    ├── WINGET-DOWNLOAD.md
    └── TROUBLESHOOTING.md
```

The `tools/`, `vendors/`, `schema/`, and `docs/` folders will fill in as the build system is developed. The README is the current reference for the intended workflow.

---

## Quick Start

> **Prerequisites:** Windows 10 or Windows 11 build host, PowerShell 5.1 or 7, 7-Zip installed at `C:\Program Files\7-Zip\7z.exe`, `winget` available on PATH, and network access to vendor CDNs.

### 1. Clone the repository

```bash
git clone https://github.com/ArthurJDurand/MDT-OEM-Extensibility.git C:\Source\MDT-OEM-Extensibility
```

### 2. Ensure `winget` is available

```powershell
winget --version
```

If this returns nothing, install **App Installer** from the Microsoft Store or update it via Windows Update.

### 3. Build a single vendor pack

```powershell
cd C:\Source\MDT-OEM-Extensibility
.\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath \\SERVER\Shared\OEM\x64
```

The tool reads `vendors\Dell\Recipe.json`, resolves each app, downloads anything missing into `cache\`, stages the assets, and packs everything into `Dell.7z`.

### 4. Build all vendors

```powershell
.\tools\Build-OEMPack.ps1 -All -OutputPath \\SERVER\Shared\OEM\x64
```

Every vendor with a complete recipe is built in sequence. Vendors with missing or manual-only apps are reported at the end.

### 5. Validate the output

```powershell
.\tools\Test-OEMAppPack.ps1 -Archive \\SERVER\Shared\OEM\x64\Dell.7z `
                           -Manifest \\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json
```

The validator checks that the archive contains every app the framework manifest expects, at the paths the framework manifest declares.

---

## Requirements

### Build host

- Windows 10 or Windows 11 (physical or virtual)
- PowerShell 5.1 (built into Windows) or PowerShell 7
- 7-Zip installed at the default location
- `winget` (App Installer) available on PATH
- At least 10 GB free disk space for the cache
- Network access to:
  - `downloads.dell.com`, `dl.dell.com` (Dell)
  - `ftp.ext.hp.com`, `ftp.hp.com` (HP)
  - `download.lenovo.com`, `support.lenovo.com` (Lenovo)
  - `download.microsoft.com`, `winget.microsoft.com` (Microsoft and winget)
  - Vendor CDNs for the remaining OEMs

### Admin rights

Not required to build packs. Required only if you point the output path at a network share that demands elevation.

### Licensing

Every installer is fetched from the vendor's own distribution channel or from Microsoft's winget. Nothing in this repository redistributes vendor binaries. The recipes record where each app comes from, so provenance is always traceable.

If you fork this repository and add vendors, keep that property. Do not commit vendor installers.

---

## Documentation

| Document | Description | Status |
|---|---|---|
| [docs/BUILDING-PACKS.md](docs/BUILDING-PACKS.md) | Build a vendor pack end-to-end, with per-vendor notes | Planned |
| [docs/ADDING-A-VENDOR.md](docs/ADDING-A-VENDOR.md) | Add a new OEM, with Dell as the worked example | Planned |
| [docs/RECIPE-SCHEMA.md](docs/RECIPE-SCHEMA.md) | Reference for `Recipe.json` — every field, every source type | Planned |
| [docs/WINGET-DOWNLOAD.md](docs/WINGET-DOWNLOAD.md) | When `winget download` works, when it does not, and how the tool falls back | Planned |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Common errors and fixes | Planned |
| [CHANGELOG.md](CHANGELOG.md) | Version history | Planned |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute | Planned |

The repository is in early development. The README describes the intended workflow; the tools and per-vendor recipes will follow as they are validated.

---

## Related Projects

| Project | Purpose |
|---|---|
| [`MDT-Zero-Touch-Deployment`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment) | The MDT deployment share. Consumes the `.7z` archives this repository produces. Contains the OEM Apps framework, task sequences, offline media, and full deployment documentation. |
| [`MDT-Windows-Image-Builder`](https://github.com/ArthurJDurand/MDT-Windows-Image-Builder) | Builds the Windows `install.wim` used by the deployment share. Optional if you have your own Windows image. |

---

## Contributing

Contributions are welcome. Areas where help is especially valuable:

- **New vendor recipes** — if you have working download URLs for a vendor not yet covered, open a PR or Discussion
- **Vendor updates** — vendors ship new versions regularly; keeping recipes current is the highest-value contribution
- **`winget download` coverage** — some apps work through winget, some do not; documenting which is which saves everyone time
- **Alternative sources** — for apps winget does not cover and vendors do not publish stable URLs for, if you find a documented official source, share it
- **Documentation** — real-world build logs and screenshots
- **Testing on different hardware and networks** — vendor CDNs behave differently by region; if you find a URL that works in one region but not another, that is worth reporting

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full guidelines (planned).

---

## Support This Project

This project is maintained in spare time and provided free of charge. If it saved you or your organization time, consider [sponsoring ongoing maintenance](https://github.com/sponsors/ArthurJDurand). Sponsorship funds vendor recipe upkeep, testing across hardware, and issue triage.

[![Sponsor](https://img.shields.io/badge/Sponsor-ArthurJDurand-ea4aaa?logo=github-sponsors&logoColor=white)](https://github.com/sponsors/ArthurJDurand)

---

## License

MIT License — see [LICENSE.md](LICENSE.md).

---

## Author

**Arthur Durand**

- GitHub: [@ArthurJDurand](https://github.com/ArthurJDurand)
- Repository: [MDT-OEM-Extensibility](https://github.com/ArthurJDurand/MDT-OEM-Extensibility)
- Sponsor: [github.com/sponsors/ArthurJDurand](https://github.com/sponsors/ArthurJDurand)

---

## Acknowledgments

- **Dell, HP, Lenovo, ASUS, Acer, MSI, Gigabyte, Dynabook, Huawei, Microsoft, and Proline** — for publishing their installers and drivers at stable, documented URLs
- **Microsoft** — for the `winget` package manager and its `download` subcommand
- The MDT and Windows deployment communities for ongoing knowledge sharing

---

<div align="center">

**If this project helped you build OEM payloads, consider giving it a ⭐**

</div>
