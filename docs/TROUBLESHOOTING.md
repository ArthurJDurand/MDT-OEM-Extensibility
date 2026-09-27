# Troubleshooting Guide

This document covers common errors you may encounter while building OEM payload archives, their likely causes, and how to resolve them.

If you cannot find your issue here, see [Getting Help](#getting-help) at the bottom.

---

## Table of Contents

- [How to Read a Build Log](#how-to-read-a-build-log)
- [Build Tool Errors](#build-tool-errors)
- [`winget download` Issues](#winget-download-issues)
- [Direct URL Download Issues](#direct-url-download-issues)
- [Hash Verification Failures](#hash-verification-failures)
- [7-Zip Packing Failures](#7-zip-packing-failures)
- [WinPE Map Builder Issues](#winpe-map-builder-issues)
- [Cache Issues](#cache-issues)
- [Network and Proxy Issues](#network-and-proxy-issues)
- [PowerShell and Environment Issues](#powershell-and-environment-issues)
- [File System and Permission Issues](#file-system-and-permission-issues)
- [Verification and Validation Failures](#verification-and-validation-failures)
- [Getting Help](#getting-help)

---

## How to Read a Build Log

Every build tool writes a log to `C:\ProgramData\MDT-OEM-Extensibility\Logs\<Vendor>.log` by default, or to the path passed via `-LogPath`. The log is plain text with timestamped entries:

```
[2026-01-15 14:32:18] [INFO]    Starting build for vendor: Dell
[2026-01-15 14:32:19] [INFO]    Loading recipe: vendors\Dell\Recipe.json
[2026-01-15 14:32:21] [INFO]    Resolving 14 app(s)
[2026-01-15 14:32:22] [INFO]    [1/14] Microsoft.WindowsAppRuntime.1.8 (winget)
[2026-01-15 14:32:30] [SUCCESS] [1/14] Downloaded: Microsoft.WindowsAppRuntime.1.8
[2026-01-15 14:32:31] [INFO]    [2/14] Microsoft .NET Windows Desktop Runtime 8 (winget)
[2026-01-15 14:32:45] [ERROR]   [2/14] winget download failed: 0x8A15002B
```

### Filtering for errors

```powershell
Select-String -Path "C:\ProgramData\MDT-OEM-Extensibility\Logs\Dell.log" -Pattern '\[ERROR\]|\[WARN\]'
```

### Following a log in real time

While a build is running in another window:

```powershell
Get-Content "C:\ProgramData\MDT-OEM-Extensibility\Logs\Dell.log" -Wait -Tail 20
```

### The build summary

At the end of every build the tool prints a summary:

```
═══════════════════════════════════════════════════════════════════
 BUILD SUMMARY: Dell
═══════════════════════════════════════════════════════════════════
  Total apps        : 14
  Downloaded        : 12
  Cached (skipped)  : 2
  Manual (pending)  : 0
  Failed            : 0
  Archive           : C:\Output\Dell.7z
  Size              : 3.42 GB
  SHA-256           : a1b2c3d4e5f6...
═══════════════════════════════════════════════════════════════════
```

If anything failed, the summary lists those apps explicitly with a reason.

---

## Build Tool Errors

### "Recipe not found"

```
[ERROR] Recipe not found: vendors\Dell\Recipe.json
```

**Cause:** The vendor folder does not exist, or the filename is misspelled.

**Fix:**

1. Verify the vendor name matches a folder under `vendors\`:
   ```powershell
   Get-ChildItem vendors -Directory | Select-Object Name
   ```
2. Verify `Recipe.json` exists in that folder:
   ```powershell
   Test-Path "vendors\Dell\Recipe.json"
   ```
3. The vendor name is case-insensitive but must match one of the eleven supported vendors exactly: `Dell`, `HP`, `Lenovo`, `ASUS`, `Acer`, `MSI`, `Gigabyte`, `Dynabook`, `Huawei`, `Microsoft`, `Proline`.

### "Recipe failed schema validation"

```
[ERROR] Recipe failed schema validation: vendors\Dell\Recipe.json
[ERROR]   - Property 'apps' is required but missing
```

**Cause:** The recipe does not match `schema\vendor-recipe.schema.json`.

**Fix:**

1. Open the recipe in an editor with JSON schema awareness (VS Code, for example).
2. Compare against a known-good recipe:
   ```powershell
   Get-Content vendors\Dell\Recipe.json -Raw | ConvertFrom-Json | Out-Null
   ```
   If this fails, the JSON is malformed. Fix the syntax first.
3. Check that every required field is present. See `docs/RECIPE-SCHEMA.md` for the field reference.

### "Framework manifest not found"

```
[ERROR] Framework manifest not found: \\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json
```

**Cause:** The tool cannot find the deployment framework's manifest for this vendor.

**Fix:**

1. Verify the manifest path exists:
   ```powershell
   Test-Path "\\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\Dell.json"
   ```
2. If the deployment share is on a different host or path, pass `-ManifestPath` explicitly:
   ```powershell
   .\tools\Build-OEMPack.ps1 -Vendor Dell -ManifestPath "\\OTHER-SERVER\Share\...\Manifests\Dell.json"
   ```
3. If the manifest is genuinely missing, the framework repo may be incomplete. See the main repo's [`docs/APPS-FRAMEWORK.md`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment/blob/main/docs/APPS-FRAMEWORK.md) for the expected structure.

### "Manifest and recipe are out of sync"

```
[ERROR] Manifest and recipe are out of sync
[ERROR]   In manifest but not in recipe: Dell Command | Update for Windows Universal
[ERROR]   In recipe but not in manifest: DellCmdUpdate
```

**Cause:** The framework manifest lists apps the recipe does not fetch, or vice versa.

**Fix:**

1. Open both files side by side:
   - `\\SERVER\DeploymentShare$\x64\$OEM$\$1\Recovery\OEM\Apps\Manifests\<Vendor>.json`
   - `vendors\<Vendor>\Recipe.json`
2. Every `AppName` in the manifest must have a matching entry in the recipe. The match is on the exact `AppName` string.
3. If the framework team added a new app, add a corresponding recipe entry.
4. If you removed an app from the recipe, remove it from the manifest too, or ask the framework maintainer to remove it.

The tools enforce this. You cannot bypass it without editing the source, which is not supported.

---

## `winget download` Issues

`winget download` is documented separately in [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md). The most common failure modes are summarized here for convenience.

### Exit code 0x8A15002B — "No matching package found"

**Cause:** The winget ID in the recipe does not exist, or the package was removed from the winget source.

**Fix:**

1. Search for the package:
   ```powershell
   winget search "<app name>"
   ```
2. Verify the ID:
   ```powershell
   winget show --id <WingetId>
   ```
3. Update the recipe with the correct ID, or open an issue if the package was removed.

### Exit code 0x8A150014 — "Installer hash mismatch"

**Cause:** The vendor replaced the installer at the URL without updating the winget manifest.

**Fix:**

1. Wait 24 hours and retry. winget maintainers usually catch up.
2. If the problem persists, use `direct-url` as a fallback source in the recipe with the hash of the current file.
3. Open an issue with the [winget-pkgs repository](https://github.com/microsoft/winget-pkgs) if the vendor's file has genuinely changed.

### Exit code 0x8A150033 — "Source requires authentication"

**Cause:** The package is MSStore-only and requires a work or school account.

**Fix:** Mark the recipe entry as `manual`. See [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md#when-it-does-not-work) for details.

### Exit code 0x8A15002E — "Download failed"

**Cause:** Network issue, CDN rate limit, or the vendor's URL is temporarily dead.

**Fix:**

1. Retry the build. The tool retries three times with exponential backoff.
2. If it fails repeatedly, check the vendor's status page.
3. If the URL is genuinely dead, use `direct-url` with a fresh URL from the vendor's site.

### Exit code 0x8A150010 — "Source agreements not accepted"

**Cause:** The tool invoked `winget download` without `--accept-source-agreements`.

**Fix:** This is a bug in the tool. Update to the latest version, or open an issue with the exit code.

### `winget` hangs and never returns

**Cause:** A prompt is waiting for input.

**Fix:**

1. Ensure `--disable-interactivity` is passed. The tool does this by default.
2. If it still hangs, `winget` may be waiting for a source update. Manually accept it once:
   ```powershell
   winget source update
   ```
3. As a last resort, kill the process and retry:
   ```powershell
   Stop-Process -Name winget -Force
   ```

---

## Direct URL Download Issues

### HTTP 403 Forbidden

**Cause:** The vendor is blocking the request. Usually caused by a missing User-Agent, a rate limit, or a geo-restriction.

**Fix:**

1. The tool sends a browser User-Agent by default. If you are seeing 403, the vendor may be blocking the specific agent or the source IP.
2. Try a different network (off VPN, or a different geographic region).
3. If the vendor requires an authenticated session, mark the app as `manual`.

### HTTP 404 Not Found

**Cause:** The URL in the recipe is stale. Vendors rotate URLs when they ship new versions.

**Fix:**

1. Visit the vendor's download page and locate the current installer.
2. Update the recipe's `Url` field.
3. Record the new SHA-256 for the downloaded file.
4. Open a PR with the updated URL and hash.

### HTTP 429 Too Many Requests

**Cause:** Rate limit exceeded. The vendor has capped the number of downloads from your IP in a window.

**Fix:**

1. Wait 10–30 minutes and retry.
2. If the build tool re-downloads the same file repeatedly, investigate the cache (see [Cache Issues](#cache-issues)).
3. For large vendor packs, space out builds rather than running them back to back.

### Timeout after 30 seconds

**Cause:** Slow CDN, network congestion, or a server that is not responding.

**Fix:**

1. The tool retries three times. If all retries time out, the CDN may be having a bad day.
2. Try again later.
3. If a specific vendor's CDN is consistently slow from your region, consider downloading the file manually and placing it in the cache (see [Cache Issues](#cache-issues)).

---

## Hash Verification Failures

### "Hash mismatch for <app>"

```
[ERROR] Hash mismatch for Dell SupportAssist
[ERROR]   Expected: a1b2c3d4e5f6...
[ERROR]   Got:      1234abcd5678...
```

**Cause:** The file at the URL has changed since the hash was recorded.

**Fix:**

1. **Do not ignore this warning.** A hash mismatch means either the vendor shipped a new version or the download was tampered with.
2. Verify the new file by hand:
   ```powershell
   Get-FileHash "C:\Path\To\Downloaded\File.exe" -Algorithm SHA256
   ```
3. Compare against the vendor's published hash if available.
4. If the file is legitimate, update the recipe's `Sha256` field and commit the change:
   ```
   fix(<vendor>): update SHA-256 for <app> after vendor update
   ```
5. If the file is suspicious, delete it from the cache and retry the download from a different network.

### "Hash missing" warning

```
[WARN] No SHA-256 recorded for Dell SupportAssist. Observed: a1b2c3d4e5f6...
```

**Cause:** The recipe entry does not record a hash. The tool computed one but did not enforce it.

**Fix:**

1. Add the observed hash to the recipe's `Sha256` field.
2. Commit the change. From the next build onward, the tool verifies the hash and fails on mismatch.

### "Hash algorithm not supported"

**Cause:** The recipe specifies an algorithm the tool does not support.

**Fix:** Only `SHA256` is supported. If a vendor only publishes MD5, use `SHA256` computed on the downloaded file. The tool computes it for you after the first successful download.

---

## 7-Zip Packing Failures

### "7-Zip not found at C:\Program Files\7-Zip\7z.exe"

**Cause:** 7-Zip is not installed, or is installed at a non-default location.

**Fix:**

1. Install 7-Zip from [7-zip.org](https://www.7-zip.org/).
2. Or pass the path explicitly:
   ```powershell
   .\tools\Build-OEMPack.ps1 -Vendor Dell -SevenZipPath "D:\Tools\7-Zip\7z.exe"
   ```

### "Cannot create archive: access denied"

**Cause:** The output path is locked, read-only, or does not exist.

**Fix:**

1. Verify the output folder exists:
   ```powershell
   Test-Path "C:\Output"
   ```
2. If the output is a network share, verify write access from your account.
3. If a previous build left the archive open (from a validation tool or a hex editor), close it and retry.

### "Archive size exceeds 4 GB"

**Cause:** FAT32's per-file limit, if the output is on a FAT32 volume.

**Fix:**

1. Build to an NTFS volume instead.
2. Or split the archive:
   ```powershell
   .\tools\New-OEMAppPack.ps1 -SourcePath .\staging\Dell -OutputPath C:\Output\Dell.7z -Volume 3g
   ```
   The tool splits into `.7z.001`, `.7z.002`, and so on when `-Volume` is passed.

### "Cannot open file as archive"

**Cause:** The `.7z` file is corrupt, incomplete, or still being written.

**Fix:**

1. Verify the file size is non-zero:
   ```powershell
   Get-Item C:\Output\Dell.7z | Select-Object Length
   ```
2. Test with 7-Zip:
   ```powershell
   & "C:\Program Files\7-Zip\7z.exe" t C:\Output\Dell.7z
   ```
3. If the test fails, delete the archive and rebuild:
   ```powershell
   Remove-Item C:\Output\Dell.7z -Force
   .\tools\Build-OEMPack.ps1 -Vendor Dell -OutputPath C:\Output -Force
   ```

---

## WinPE Map Builder Issues

### "Catalog download failed"

**Cause:** The vendor's catalog is unreachable or has moved.

**Fix:**

1. Test the catalog URL in a browser.
2. For Dell, the catalog is `https://downloads.dell.com/catalog/DriverPackCatalog.cab`.
3. For HP, the catalog page is `https://ftp.ext.hp.com/pub/caps-softpaq/cmit/HP_WinPE_DriverPack.html`.
4. For Lenovo, the recipe card is `https://download.lenovo.com/cdrt/ddrc/recipecard.json`.
5. If a URL is genuinely dead, open an issue. The vendor may have changed their catalog structure.

### "Catalog XML is malformed"

**Cause:** The vendor updated their catalog format without warning.

**Fix:**

1. Open the downloaded catalog and inspect the XML structure.
2. Update the parser in the corresponding `Build-<Vendor>WinPEMap.ps1` script.
3. Open a PR with the fix.

### "No WinPE packages found"

**Cause:** The vendor's catalog does not have entries with the expected type, or the filter is too strict.

**Fix:**

1. Inspect the raw catalog for `type="winpe"` entries (Dell) or the WinPE driver pack table (HP) or the `WinPEPacks` section (Lenovo).
2. If the vendor renamed the type, update the filter in the parser.
3. If the vendor genuinely has no WinPE packages, the vendor may not publish them anymore.

### "DS ID could not be resolved"

**Cause:** Lenovo's support page did not expose the direct download URL.

**Fix:**

1. The Lenovo builder resolves DS IDs via support page scraping. If a support page changes its layout, resolution fails.
2. Update the parser in `Build-LenovoWinPEMap.ps1`.
3. As a workaround, mark that DS ID as `manual` in the output map and resolve it by hand.

### Lenovo builder requires `curl-impersonate`

**Cause:** Lenovo's support site rejects non-browser User-Agents, including standard curl. The builder uses `curl-impersonate` to mimic Chrome.

**Fix:**

1. The builder downloads `curl-impersonate` automatically on first run.
2. If the download fails, the archive is on GitHub at [`lexiforest/curl-impersonate`](https://github.com/lexiforest/curl-impersonate/releases).
3. Place the extracted binary at `C:\Temp\LenovoMapBuild\curl-archive\curl-impersonate.exe` and rerun.

---

## Cache Issues

### Where the cache lives

By default, downloaded files are cached at `cache\` under the repository root. The location can be overridden with `-CachePath`.

The cache is gitignored. It never appears in commits.

### Cache is full or corrupt

**Symptoms:** Builds fail with "cannot write to cache" or "cached file failed hash verification."

**Fix:**

1. Inspect the cache size:
   ```powershell
   (Get-ChildItem cache -Recurse | Measure-Object Length -Sum).Sum / 1GB
   ```
2. If the cache is over ~20 GB, prune old entries:
   ```powershell
   .\tools\Build-OEMPack.ps1 -Vendor Dell -PurgeCache
   ```
   This deletes cached files not referenced by any current recipe.
3. For a corrupt entry, delete it manually:
   ```powershell
   Remove-Item "cache\<entry>" -Force -Recurse
   ```
   The next build re-downloads it.

### Cache is not being used

**Symptom:** Every build re-downloads everything, even when nothing changed.

**Cause:** The cache path is being overridden somewhere, or the tool is not finding the existing cache.

**Fix:**

1. Verify the cache path:
   ```powershell
   Get-ChildItem cache
   ```
2. Verify the tool is using the default path by inspecting the log:
   ```
   [INFO] Cache path: C:\Source\MDT-OEM-Extensibility\cache
   ```
3. If the log shows a different path, check for a `-CachePath` argument in your invocation, or an environment variable overriding it.

---

## Network and Proxy Issues

### "Could not establish trust relationship for the SSL/TLS secure channel"

**Cause:** The build host does not trust the vendor's certificate. Usually caused by a corporate proxy that intercepts TLS, or an outdated root certificate store.

**Fix:**

1. Update the root certificate store:
   ```powershell
   certutil -generateSSTFromWU "C:\Temp\roots.sst"
   Import-Certificate -FilePath "C:\Temp\roots.sst" -CertStoreLocation Cert:\LocalMachine\Root
   ```
2. If behind a corporate proxy, import the proxy's CA certificate into the Windows Root store.
3. As a last resort (not recommended), use `-SkipCertificateCheck` if the tool supports it. Verify the file hash manually afterward.

### "The remote name could not be resolved"

**Cause:** DNS resolution failed. The build host cannot reach the vendor's domain.

**Fix:**

1. Verify DNS:
   ```powershell
   Resolve-DnsName downloads.dell.com
   ```
2. If DNS fails, check the host's DNS configuration.
3. If DNS succeeds but the connection still fails, the vendor may be blocked by a firewall or a corporate proxy.

### "The operation has timed out"

**Cause:** Slow network, VPN, or a proxy that is dropping the connection.

**Fix:**

1. Retry. The tool retries three times.
2. If you are on a VPN, try disconnecting. Some CDNs block VPN ranges.
3. If behind a corporate proxy, verify the proxy is configured for the build host:
   ```powershell
   netsh winhttp show proxy
   ```

### Behind a corporate proxy

The tool respects the system proxy by default. To set it explicitly:

```powershell
[System.Net.WebRequest]::DefaultWebProxy = New-Object System.Net.WebProxy("http://proxy.company.local:8080", $true)
```

Or set the environment variables before invoking:

```powershell
$env:HTTP_PROXY  = "http://proxy.company.local:8080"
$env:HTTPS_PROXY = "http://proxy.company.local:8080"
```

---

## PowerShell and Environment Issues

### "The term 'Build-OEMPack.ps1' is not recognized"

**Cause:** The working directory is wrong, or the script path was not specified.

**Fix:**

1. Navigate to the repository root:
   ```powershell
   Set-Location C:\Source\MDT-OEM-Extensibility
   ```
2. Or invoke with an explicit path:
   ```powershell
   & "C:\Source\MDT-OEM-Extensibility\tools\Build-OEMPack.ps1" -Vendor Dell
   ```

### "Running scripts is disabled on this system"

**Cause:** PowerShell execution policy is too restrictive.

**Fix:**

1. Check the current policy:
   ```powershell
   Get-ExecutionPolicy
   ```
2. For the current session only:
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```
3. For the current user permanently:
   ```powershell
   Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
   ```

### "This script requires PowerShell 7"

**Cause:** A script requires PowerShell 7 but you are running 5.1. Most scripts in this repository run on 5.1; a few (mostly the map builders that use newer features) require 7.

**Fix:**

1. Check your version:
   ```powershell
   $PSVersionTable.PSVersion
   ```
2. Install PowerShell 7 from [learn.microsoft.com](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows).
3. Invoke with `pwsh` instead of `powershell`:
   ```powershell
   pwsh -File .\tools\Build-DellWinPEMap.ps1
   ```

### `winget` is not on PATH

**Cause:** App Installer is not installed, or is an older version that did not add itself to PATH.

**Fix:**

1. Verify:
   ```powershell
   Get-Command winget
   ```
2. Install **App Installer** from the Microsoft Store.
3. If already installed, update it:
   ```powershell
   winget upgrade Microsoft.AppInstaller
   ```

### `7z.exe` is not on PATH

**Cause:** 7-Zip is installed at a non-default location, or was not added to PATH.

**Fix:**

1. Check the default location:
   ```powershell
   Test-Path "C:\Program Files\7-Zip\7z.exe"
   ```
2. If not there, install 7-Zip from [7-zip.org](https://www.7-zip.org/).
3. If installed elsewhere, pass `-SevenZipPath` explicitly, or add the folder to PATH:
   ```powershell
   $env:Path += ";D:\Tools\7-Zip"
   ```

---

## File System and Permission Issues

### "Access to the path is denied"

**Cause:** Insufficient permissions on the output path, staging folder, or cache.

**Fix:**

1. If the output is a network share, verify write access:
   ```powershell
   New-Item "\\SERVER\Shared\OEM\x64\test.txt" -Force
   Remove-Item "\\SERVER\Shared\OEM\x64\test.txt" -Force
   ```
2. If the output is a local path, verify the current user has write access.
3. Run PowerShell as administrator if the tool needs to write to a protected path.

### "The process cannot access the file because it is being used by another process"

**Cause:** The file is open in another program — an archive viewer, a hex editor, or a previous build that did not exit cleanly.

**Fix:**

1. Identify the process holding the file:
   ```powershell
   Get-Process | Where-Object { $_.Modules.FileName -like "*<filename>*" }
   ```
2. Close the process, or reboot if necessary.
3. Delete and rebuild.

### "The specified path is too long"

**Cause:** A path inside the archive exceeds the Windows 260-character limit.

**Fix:**

1. Enable long path support in Windows:
   ```powershell
   Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
                    -Name "LongPathsEnabled" -Value 1 -Type DWord
   ```
2. Reboot.
3. Retry the build.

### File is locked after a failed build

**Cause:** A staging folder or partial download was not cleaned up.

**Fix:**

1. Locate leftover staging folders:
   ```powershell
   Get-ChildItem $env:TEMP -Filter "MDT-OEM-*" -Directory
   ```
2. Delete them manually:
   ```powershell
   Remove-Item "$env:TEMP\MDT-OEM-*" -Recurse -Force
   ```

---

## Verification and Validation Failures

### `Test-OEMAppPack.ps1` reports missing apps

```
[FAIL] Missing in archive: Dell SupportAssist UWP
[FAIL]   Expected at: C:\Recovery\OEM\Apps\SupportAssist\UWP\SupportAssist_x64.msix
```

**Cause:** The archive does not contain an app the framework manifest expects.

**Fix:**

1. Determine whether the app was skipped because it is `manual`:
   ```powershell
   Get-Content vendors\Dell\Recipe.json -Raw | ConvertFrom-Json |
       Select-Object -ExpandProperty apps |
       Where-Object { $_.AppName -eq "Dell SupportAssist UWP" }
   ```
2. If the recipe marks it as `manual`, place the file in the staging folder before building, or use the `-SkipManual` flag on the validator to acknowledge.
3. If the recipe marks it as `winget` or `direct-url`, the fetch failed. Check the build log for that app.

### Archive hash does not match the expected value

**Cause:** The archive was rebuilt from different source content.

**Fix:**

1. Confirm the recipe, assets, and staging folder have not changed since the hash was recorded.
2. If they have, that is expected — update the hash in your documentation.
3. If they have not, the archive is not reproducible. Open an issue.

### Validator passes but the deployment framework cannot find an app

**Cause:** The framework manifest's `InstallerPath` does not match the archive's internal structure.

**Fix:**

1. Compare the framework manifest path with the archive contents:
   ```powershell
   & "C:\Program Files\7-Zip\7z.exe" l C:\Output\Dell.7z | Select-String "<AppName>"
   ```
2. If the path differs, the recipe's `TargetPath` (or the staging layout) is wrong. Fix and rebuild.
3. If the paths match, the issue is in the framework side. See the main repo's [`docs/TROUBLESHOOTING.md`](https://github.com/ArthurJDurand/MDT-Zero-Touch-Deployment/blob/main/docs/TROUBLESHOOTING.md).

---

## Getting Help

If your issue is not covered here:

1. **Search existing issues** — [github.com/ArthurJDurand/MDT-OEM-Extensibility/issues](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/issues)
2. **Search existing discussions** — [github.com/ArthurJDurand/MDT-OEM-Extensibility/discussions](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/discussions)
3. **Open a new discussion** for general questions — [github.com/ArthurJDurand/MDT-OEM-Extensibility/discussions/new](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/discussions/new)
4. **Open a bug report** for reproducible issues — use the [bug report template](https://github.com/ArthurJDurand/MDT-OEM-Extensibility/issues/new?template=bug_report.yml)

### What to include when asking for help

- **What you expected to happen** — one sentence
- **What actually happened** — one sentence, with the exact error message
- **Which vendor and app** — if the failure is vendor-specific
- **Build host details:**
  - Windows build (`winver`)
  - PowerShell version (`$PSVersionTable.PSVersion`)
  - winget version (`winget --version`)
  - 7-Zip version
- **Network context:**
  - Geographic region (approximate)
  - VPN or corporate proxy
- **Relevant log excerpt** — 20–30 lines around the error, not the whole log
- **What you already tried** — so suggestions are not repeated

**Do not paste entire logs inline.** Attach them as files or upload to a [GitHub Gist](https://gist.github.com/) and link the URL.

### Response times

This is a hobby project maintained in spare time. Response times vary from hours to days. If your issue is blocking production, consider it a signal to invest in vendor-supported deployment tooling for production, and use this project for lab or small-batch deployments.

---

*See [docs/WINGET-DOWNLOAD.md](WINGET-DOWNLOAD.md) for `winget download` specifics, [docs/BUILDING-PACKS.md](BUILDING-PACKS.md) for the full build workflow, and the [README](../README.md) for the project overview.*
