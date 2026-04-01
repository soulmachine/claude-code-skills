---
name: mac-mini-as-headless-server
description: Use when setting up a Mac (especially Mac Mini) for unattended 24/7 server operation, headless use, or remote-only access. Covers sleep prevention, screen saver, Wake-on-LAN, auto-restart, App Nap, and SSH enablement.
---

# Configure macOS for Server Use

## Overview

Configures macOS for unattended 24/7 server operation by disabling sleep, screen saver, screen lock, and App Nap, while enabling Wake-on-LAN, auto-restart after power failure, and SSH remote login.

**Requires `sudo`.** Target: Mac Mini or any Mac used as a headless server.

## When to Use

- Setting up a Mac as a home server, CI runner, or always-on machine
- Preparing a Mac Mini for headless/remote-only operation
- Troubleshooting a Mac that keeps sleeping, locking, or becoming unreachable

## Quick Reference

| Setting | Command | Effect |
|---------|---------|--------|
| Screen saver off | `defaults -currentHost write com.apple.screensaver idleTime -int 0` | Disables screen saver activation |
| Screen lock off | `defaults write com.apple.screensaver askForPassword -int 0` | No password on wake |
| Lock delay off | `defaults write com.apple.screensaver askForPasswordDelay -int 0` | Immediate effect |
| System sleep off | `pmset -c sleep 0` | Never sleep |
| Display sleep off | `pmset -c displaysleep 0` | Never turn off display |
| Disk sleep off | `pmset -c disksleep 0` | Never spin down disks |
| Dim before sleep off | `pmset -c halfdim 0` | No pre-sleep dimming |
| Wake-on-LAN | `pmset -c womp 1` | Wake on network access |
| Auto-restart | `pmset -c autorestart 1` | Restart after power loss |
| App Nap off | `defaults write NSGlobalDomain NSAppSleepDisabled -bool YES` | Prevents app throttling |
| SSH on | `systemsetup -setremotelogin on` | Enables Remote Login |

## Implementation

Run as root. `REAL_USER` is resolved from `$SUDO_USER` to apply per-user defaults correctly.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Require root
if [[ $EUID -ne 0 ]]; then
  echo "Error: run with sudo."
  exit 1
fi

REAL_USER="${SUDO_USER:-$USER}"

# 1. Disable screen saver
sudo -u "$REAL_USER" defaults -currentHost write com.apple.screensaver idleTime -int 0

# 2. Disable screen lock password
sudo -u "$REAL_USER" defaults write com.apple.screensaver askForPassword -int 0
sudo -u "$REAL_USER" defaults write com.apple.screensaver askForPasswordDelay -int 0

# 3. Prevent all sleep (charger profile — only profile on Mac Mini)
pmset -c sleep 0
pmset -c displaysleep 0
pmset -c disksleep 0
pmset -c halfdim 0

# 4. Wake-on-LAN
pmset -c womp 1

# 5. Auto-restart after power failure
pmset -c autorestart 1

# 6. Disable App Nap globally
sudo -u "$REAL_USER" defaults write NSGlobalDomain NSAppSleepDisabled -bool YES

# 7. Enable SSH
systemsetup -setremotelogin on 2>/dev/null || {
  launchctl load -w /System/Library/LaunchDaemons/ssh.plist 2>/dev/null || true
}

# Verify
pmset -g
```

## Common Mistakes

- **Running without sudo** — `pmset` and `systemsetup` require root; `defaults` commands must run as the real user via `sudo -u`
- **Forgetting `-c` flag** — Without `-c` (charger/AC), settings apply to battery profile, which doesn't exist on Mac Mini
- **Expecting immediate effect** — Screen saver and password changes may require logout/restart
- **FileVault blocking auto-login** — Auto-login (System Settings > Users & Groups > Login Options) requires FileVault off

## Reverting

To restore display sleep later:
```bash
sudo pmset -c displaysleep 10  # 10 minutes
```
