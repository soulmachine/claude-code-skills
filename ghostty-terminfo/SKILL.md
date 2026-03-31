---
name: ghostty-terminfo
description: Use when SSHing to a remote host from Ghostty terminal and encountering terminfo errors, missing colors, broken key bindings, or "unknown terminal type" warnings
---

# Ghostty Terminfo Installation

## Overview

Ghostty uses `xterm-ghostty` as its `$TERM` value. Remote hosts that lack this terminfo entry will show broken terminal behavior. The fix is to transfer the terminfo from the local machine to the remote host.

## When to Use

- Remote SSH sessions show "unknown terminal type xterm-ghostty"
- Missing colors, broken backspace/arrow keys, or garbled output over SSH
- Setting up a new remote host for use with Ghostty
- Writing SSH scripts that should handle Ghostty terminfo automatically

## Quick Reference

| Task | Command |
|------|---------|
| Check if remote has terminfo | `ssh HOST 'infocmp xterm-ghostty >/dev/null 2>&1'` |
| Install terminfo on remote | `infocmp -x xterm-ghostty \| ssh HOST tic -x -` |
| One-liner check + install | `ssh HOST 'infocmp xterm-ghostty >/dev/null 2>&1' \|\| infocmp -x xterm-ghostty \| ssh HOST tic -x -` |
| Verify after install | `ssh HOST 'infocmp xterm-ghostty >/dev/null 2>&1 && echo OK'` |

## Implementation

### One-Time Install

```bash
infocmp -x xterm-ghostty | ssh user@host tic -x -
```

This exports the local terminfo and compiles it on the remote host. Only needs to run once per remote machine.

### Conditional Install in SSH Scripts

```bash
if [ "$TERM" = "xterm-ghostty" ]; then
  ssh "$HOST" 'infocmp xterm-ghostty >/dev/null 2>&1' || \
    infocmp -x xterm-ghostty | ssh "$HOST" tic -x -
fi
```

Only runs when connecting from Ghostty, skips if already installed.

## Common Mistakes

- **Running from non-Ghostty terminal** - `infocmp xterm-ghostty` fails if the local machine doesn't have the terminfo. Run from Ghostty or a machine with Ghostty installed.
- **Forgetting `-x` flag** - Both `infocmp -x` and `tic -x` need the extended flag to preserve Ghostty's extended capabilities.
- **Workaround instead of fix** - Setting `TERM=xterm-256color` before SSH works but loses Ghostty-specific features. Install the terminfo instead.
