<div align="center">

# upgrade-all-windows-apps

**One command to upgrade every installed application on Windows — silently, with zero prompts.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Windows_10_1809%2B%20%7C%20Windows_11-blue)](https://learn.microsoft.com/en-us/windows/package-manager/winget/)

</div>

---

## Overview

`winget upgrade --all` is the closest thing Windows has to `apt upgrade`. However, out of the box it still halts on:

- Source terms-of-service prompts (first run per source)
- Per-package license / EULA prompts
- Installer progress windows
- Packages with undetectable versions (silently skipped)
- Pinned packages (silently skipped)

This repository documents a single, copy-paste command that eliminates every one of these, enabling a true **fire-and-forget** bulk upgrade.

## Requirements

| Requirement | Notes |
|:---|:---|
| Windows 10 1809+ or Windows 11 | `winget` is preinstalled on Windows 11; on Windows 10 install via [Microsoft Store](https://apps.microsoft.com/detail/9PGJGP5LSPG9) or the [App Installer](https://learn.microsoft.com/en-us/windows/package-manager/winget/) package |
| Administrator privileges | Required for machine-scope installs and to write to `Program Files` |
| Internet connection | winget downloads installers from package sources |

## Usage

### Preview (recommended first step)

Before running the full upgrade, see what *would* be updated:

```cmd
winget upgrade --all
```

This lists all available upgrades without installing anything.

### Full unattended upgrade

Open **Command Prompt** or **PowerShell** as **Administrator** and run:

```cmd
winget upgrade --all --include-unknown --include-pinned --silent --accept-package-agreements --accept-source-agreements --disable-interactivity
```

The command runs to completion with zero user input. Typical duration: **2–10 minutes** depending on the number of upgradable packages and network speed.

## Flag Reference

| Flag | Description |
|:---|:---|
| `--all` | Upgrade every installed package that has an available update |
| `--include-unknown` | Include packages whose installed version cannot be determined (e.g. some portable apps, manually installed software) |
| `--include-pinned` | Include packages that have a **non-blocking** pin. Blocking pins still prevent upgrade |
| `--silent` | Suppress all installer UI windows (MSI/EXE progress dialogs) |
| `--accept-package-agreements` | Auto-accept per-package license terms (EULAs) |
| `--accept-source-agreements` | Auto-accept source terms of service (e.g. the `winget` source) |
| `--disable-interactivity` | Disable all remaining interactive prompts at the winget level |

> **Note:** `--silent` controls the *installer* UI. `--disable-interactivity` controls *winget's own* prompts. Both are needed for a fully unattended run.

## What This Does NOT Do

- **Does not update Windows itself.** Use `start ms-settings:windowsupdate` or Windows Update for OS updates.
- **Does not bypass blocking pins.** If a package has a blocking pin (`winget pin add --blocking`), it will still be skipped even with `--include-pinned`.
- **Does not reinstall.** If a package is already at the latest version, it is skipped (exit code `UPDATE_NOT_APPLICABLE`).
- **Does not manage dependencies automatically** unless the package manifest declares them. Use `--skip-dependencies` to explicitly skip dependency resolution.

## Exit Codes

winget returns specific exit codes. The most common ones you'll encounter:

| Exit Code (hex) | Symbol | Meaning |
|:---|:---|:---|
| `0x00000000` | `SUCCESS` | All upgrades completed successfully |
| `0x8A15002B` | `UPDATE_NOT_APPLICABLE` | No applicable update found (all packages already up to date) |
| `0x8A15002C` | `UPDATE_ALL_HAS_FAILURE` | `upgrade --all` completed, but **one or more** packages failed |
| `0x8A150109` | `INSTALL_REBOOT_REQUIRED_TO_FINISH` | A reboot is needed to complete one or more installations |
| `0x8A15010D` | `INSTALL_ALREADY_INSTALLED` | The latest version is already installed |
| `0x8A150014` | `NO_APPLICATIONS_FOUND` | No packages matched the criteria |

> A non-zero exit code does **not** always mean failure. `UPDATE_NOT_APPLICABLE` is the expected result when everything is already current.

To decode any exit code:

```cmd
winget error 0x8A15002C
```

## Scheduling (Task Scheduler)

For automated periodic upgrades, wrap the command in a batch file:

**`upgrade-all-windows-apps.bat`**

```bat
@echo off
winget upgrade --all --include-unknown --include-pinned --silent --accept-package-agreements --accept-source-agreements --disable-interactivity
```

Then create a scheduled task:

1. Open **Task Scheduler** → **Create Task**
2. Set **Run with highest privileges**
3. Trigger: e.g. Weekly, Sunday 02:00
4. Action: **Start a program** → browse to your `.bat` file
5. Under **Conditions**, uncheck "Start the task only if the computer is on AC power" if needed

Alternatively, via PowerShell:

```powershell
$action    = New-ScheduledTaskAction -Execute "upgrade-all-windows-apps.bat"
$trigger   = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 2am
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -Action $action -Trigger $trigger -Principal $principal -TaskName "Winget Weekly Upgrade"
```

## Logging

Add `--log <path>` to write a detailed log file:

```cmd
winget upgrade --all --include-unknown --include-pinned --silent --accept-package-agreements --accept-source-agreements --disable-interactivity --log "$env:TEMP\winget-upgrade.log"
```

For verbose logging:

```cmd
winget upgrade --all --include-unknown --include-pinned --silent --accept-package-agreements --accept-source-agreements --disable-interactivity --verbose-logs
```

Default log location: `%LOCALAPPDATA%\Packages\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\LocalState\DiagOutputDir`

## PowerShell Wrapper (with exit-code handling)

```powershell
winget upgrade --all --include-unknown --include-pinned --silent --accept-package-agreements --accept-source-agreements --disable-interactivity
$code = $LASTEXITCODE

# Treat "nothing to do" as success
$successCodes = @(0, -1978335189, -1978334963)  # SUCCESS, UPDATE_NOT_APPLICABLE, INSTALL_ALREADY_INSTALLED

if ($successCodes -contains $code) {
    Write-Host "All packages up to date." -ForegroundColor Green
} elseif ($code -eq -1978335188) {  # UPDATE_ALL_HAS_FAILURE
    Write-Warning "Upgrade completed with some failures. Check output above."
} else {
    Write-Error "winget exited with code $code (0x{0:X8})" -f $code
}
```

## Troubleshooting

| Symptom | Cause | Fix |
|:---|:---|:---|
| `0x8a15003f: The source data is corrupted or tampered` | Corrupted source index | `winget source reset --force` then retry |
| `0x8a15000f: Installer still running` | A previous install is stuck | Close the stuck installer, or `winget source reset --force` |
| Package hangs indefinitely | Pending update from a prior interrupted install | `winget source update` → retry; or uninstall & reinstall the specific package |
| `--include-pinned` doesn't upgrade a package | The pin is **blocking** | `winget pin remove <package-id>` |
| `0x8a150046: Source agreements were not agreed to` | `--disable-interactivity` without `--accept-source-agreements` | Ensure both flags are present (they are in this command) |
| MSI error `1603` | Generic installer failure (permissions, antivirus, running app) | Close the app, retry, or check the per-package log in `DiagOutputDir` |
| MSI error `1618` | Another installation is in progress | Wait for the other installer to finish, then retry |

## Limitations

- **EXE-based installers** have non-standard exit codes that winget may not map. A "failure" may actually be a successful install.
- **Portable apps** are upgraded by replacing the entire directory. `--purge` can be added to clean up old files first.
- **MSIX / Microsoft Store apps** are handled natively by winget and generally upgrade without issue.
- winget does **not** support wildcard queries. `--all` is the only way to bulk-upgrade.

## References

- [winget upgrade documentation](https://learn.microsoft.com/en-us/windows/package-manager/winget/upgrade)
- [winget troubleshooting](https://learn.microsoft.com/en-us/windows/package-manager/winget/troubleshooting)
- [winget return codes](https://github.com/microsoft/winget-cli/blob/master/doc/windows/package-manager/winget/returnCodes.md)

## License

[MIT](./LICENSE)   
