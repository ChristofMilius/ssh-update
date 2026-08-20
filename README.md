# OpenSSH Upgrade: Windows Built-in → GitHub Release

## Overview

| | Before | After |
|---|---|---|
| **OpenSSH** | 9.5p2 (Windows Capability) | 10.0p2 (GitHub/Win32-OpenSSH) |
| **LibreSSL** | 3.8.2 | 4.2.0 |
| **Telemetry** | Enabled (opt-out via Settings) | None |
| **Install path** | `C:\Windows\System32\OpenSSH` | `C:\Program Files\OpenSSH` |

## Prerequisites

- Windows 10/11 with built-in OpenSSH installed
- Administrator access (UAC elevation required for several steps)
- winget package manager installed

## Steps

### 1. Check current version

```powershell
ssh -V
# OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2
```

### 2. Search for available updates

```powershell
winget search OpenSSH
# Found: Microsoft.OpenSSH.Preview — Version 10.0.0.0
```

The "Preview" label is a **support designation only** (no enterprise support contract), not a quality indicator. GitHub releases do not include telemetry.

### 3. Install via winget

```powershell
winget install Microsoft.OpenSSH.Preview --accept-package-agreements --accept-source-agreements
```

### 4. Sanity check — verify new binary exists

```powershell
& "C:\Program Files\OpenSSH\ssh.exe" -V
# OpenSSH_for_Windows_10.0p2 Win32-OpenSSH-GitHub, LibreSSL 4.2.0
```

At this point, running `ssh -V` still shows the old version — `C:\Windows\System32\OpenSSH` takes PATH priority.

### 5. Uninstall old Windows Capability OpenSSH

```powershell
# Requires UAC elevation
Remove-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
Remove-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

### 6. Sanity check — verify old capability is removed

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
# Should show: NotPresent
```

### 7. Clean up stale PATH entry

The old path `C:\Windows\System32\OpenSSH;` remains in the system PATH as a dead entry.

**Manual step** — Open System Properties → Environment Variables → System → Path → Remove `C:\Windows\System32\OpenSSH`. Leave `C:\Program Files\OpenSSH\` in place.

### 8. Sanity check — verify correct PATH and version

Open a **new terminal** (current session has stale PATH cache):

```powershell
ssh -V
# OpenSSH_for_Windows_10.0p2 Win32-OpenSSH-GitHub, LibreSSL 4.2.0

Get-Command ssh.exe | Select-Object Source
# C:\Program Files\OpenSSH\ssh.exe
```

### 9. Fix Git SSH path

Git looks for `ssh.exe` at `C:\Windows\System32\OpenSSH` by default. After moving to `C:\Program Files\OpenSSH`, Git operations like `git push` fail with `cannot spawn ssh.exe`. Set `GIT_SSH_COMMAND` so Git uses the new path:

```powershell
[Environment]::SetEnvironmentVariable("GIT_SSH_COMMAND", '"C:\Program Files\OpenSSH\ssh.exe"', "User")
```

Verify:

```powershell
# In a new terminal (or refresh current: $env:GIT_SSH_COMMAND = [Environment]::GetEnvironmentVariable("GIT_SSH_COMMAND", "User"))
git push  # should succeed (or report "Everything up-to-date")
```

### 10. Test GitHub SSH authentication

```powershell
ssh -T git@github.com
# Hi <username>! You've successfully authenticated...
```

### 11. Re-register ssh-agent service (if previously used)

Uninstalling the Windows Capability version removes the `ssh-agent` Windows service. If you use KeePassXC or other tools that integrate with ssh-agent:

```powershell
# Requires elevated PowerShell (Run as Administrator)
New-Service -Name "ssh-agent" `
  -DisplayName "OpenSSH Authentication Agent" `
  -BinaryPathName "C:\Program Files\OpenSSH\ssh-agent.exe" `
  -StartupType Automatic `
  -Description "OpenSSH Authentication Agent"

Start-Service ssh-agent
```

### 12. Sanity check — verify agent service

```powershell
Get-Service ssh-agent
# Status: Running, StartType: Automatic

ssh-add -l
# Lists keys loaded in agent
```

Restart KeePassXC (or any SSH agent integration) to re-establish the connection.

## Notes

- SSH keys in `~\.ssh\` are **not affected** by the upgrade — they are version-independent.
- The `sshd` service was automatically updated to point to the new path by the winget installer.
- The new version adds support for `mlkem768x25519-sha256` and other modern key exchange algorithms.
- Future updates: `winget upgrade Microsoft.OpenSSH.Preview`
- If your SSH private key is managed by an agent (e.g. KeePassXC) rather than stored as a file on disk, make sure `~/.ssh/config` for `github.com` does **not** contain `IdentityFile` or `IdentitiesOnly` — these force the SSH client to look for a key file and ignore the agent entirely. A minimal config works best:

  ```
  Host github.com
      HostName github.com
      User git
  ```
