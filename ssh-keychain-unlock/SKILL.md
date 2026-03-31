---
name: ssh-keychain-unlock
description: Use when Claude Code auth fails over SSH on macOS, keychain is locked in headless/remote sessions, or setting up Claude Code on a Mac for remote access
---

# SSH Keychain Unlock for Claude Code on macOS

## Overview

Claude Code stores credentials in the macOS Keychain (`Claude Code-credentials` service). When accessing a Mac via SSH (no GUI session), the login keychain is locked, causing Claude Code to appear unauthenticated.

## When to Use

- Claude Code says it's not logged in when accessed via SSH
- `security show-keychain-info ~/Library/Keychains/login.keychain-db` shows the keychain is locked
- Setting up a Mac Mini or headless Mac for remote Claude Code usage

## Solutions

### Option 1: Interactive Unlock on SSH Login

Add to `~/.zshrc`:

```bash
# Unlock macOS keychain for SSH sessions (needed for Claude Code auth)
if [[ -n "$SSH_CONNECTION" ]]; then
  security unlock-keychain ~/Library/Keychains/login.keychain-db 2>/dev/null
fi
```

Prompts for macOS login password each SSH session. Simple but requires manual input.

### Option 2: Auto-Unlock at Boot (Headless)

For fully headless operation with no password prompt:

**1. Create password file** (`~/.claude/.keychain-password`, permissions `600`):
```bash
echo 'YOUR_MACOS_PASSWORD' > ~/.claude/.keychain-password
chmod 600 ~/.claude/.keychain-password
```

**2. Create unlock script** (`~/.claude/unlock-keychain.sh`, permissions `700`):
```bash
cat > ~/.claude/unlock-keychain.sh << 'SCRIPT'
#!/bin/bash
security unlock-keychain -p "$(cat ~/.claude/.keychain-password)" ~/Library/Keychains/login.keychain-db
SCRIPT
chmod 700 ~/.claude/unlock-keychain.sh
```

**3. Create LaunchAgent** (`~/Library/LaunchAgents/com.claude.unlock-keychain.plist`):
```bash
cat > ~/Library/LaunchAgents/com.claude.unlock-keychain.plist << 'PLIST'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.unlock-keychain</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>__HOME__/.claude/unlock-keychain.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
PLIST
# Fix path
sed -i '' "s|__HOME__|$HOME|g" ~/Library/LaunchAgents/com.claude.unlock-keychain.plist
```

**4. Load the agent:**
```bash
launchctl load ~/Library/LaunchAgents/com.claude.unlock-keychain.plist
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `security show-keychain-info ~/Library/Keychains/login.keychain-db` | Check keychain lock status |
| `security unlock-keychain ~/Library/Keychains/login.keychain-db` | Manually unlock (interactive) |
| `bash ~/.claude/unlock-keychain.sh` | Test auto-unlock script |
| `launchctl load ~/Library/LaunchAgents/com.claude.unlock-keychain.plist` | Load LaunchAgent |
| `launchctl unload ~/Library/LaunchAgents/com.claude.unlock-keychain.plist` | Unload LaunchAgent |

## Common Mistakes

- **Wrong permissions on password file** - Must be `600` (owner-only). Others can read your macOS password otherwise.
- **Forgetting to load the LaunchAgent** - Creating the plist isn't enough; run `launchctl load` to activate it.
- **Password file out of sync** - If you change your macOS password, update `~/.claude/.keychain-password` too.
