# aimux

[![npm version](https://img.shields.io/npm/v/@digital-threads/aimux?color=cb3837&logo=npm)](https://www.npmjs.com/package/@digital-threads/aimux)
[![npm downloads](https://img.shields.io/npm/dw/@digital-threads/aimux?color=cb3837&logo=npm)](https://www.npmjs.com/package/@digital-threads/aimux)
[![license](https://img.shields.io/npm/l/@digital-threads/aimux?color=blue)](./LICENSE)
[![node](https://img.shields.io/node/v/@digital-threads/aimux?color=339933&logo=node.js)](https://nodejs.org)
[![GitHub stars](https://img.shields.io/github/stars/Digital-Threads/aimux?style=social)](https://github.com/Digital-Threads/aimux)

Local AI workspace orchestrator — manage multiple AI CLI subscriptions with shared knowledge and isolated authentication.

## Problem

You have multiple Claude Code subscriptions (personal, work, client) each in separate `~/.claude-*` directories. You maintain symlinks manually, duplicate settings, and juggle bash functions to switch between them. Or you want to use a self-hosted / 3rd-party Claude API endpoint alongside your subscription.

## Solution

**aimux** treats your AI CLI configs like tmux treats terminals: one shared brain, multiple isolated sessions.

- **Shared layer**: agents, skills, commands, rules, memory, plugins, settings — symlinked from a single source of truth
- **Private layer**: credentials, rate limits, session state — isolated per profile
- **Zero duplication**: add a skill once, available everywhere
- **3rd-party API support**: connect any profile to a custom endpoint via a per-profile `.env` file

## Install

```bash
npm install -g @digital-threads/aimux
```

## Getting Started

### You have `~/.claude` + extra directories (`~/.claude-work`, etc.)

```bash
npm install -g aimux
aimux init              # auto-detects all ~/.claude* dirs
aimux status            # verify profiles, auth, symlinks
aimux run w             # launch work profile (prefix matching)
```

### You have only `~/.claude` (one subscription)

```bash
npm install -g aimux
aimux init              # creates config with main profile
aimux profile add work  # add a new profile
aimux auth login work   # OAuth for the new account
aimux profile update w -m claude-opus-4-6
aimux run w
```

### You want to connect a 3rd-party / self-hosted API endpoint

```bash
aimux profile add myapi --api
# Configure API endpoint (leave blank to skip a field):
#   Base URL:        https://api.your-provider.com/v1
#   Auth token:      [hidden]
#   Default model:   claude-sonnet-4-6
# ✓ Credentials saved to ~/.aimux/profiles/myapi/.env
aimux run myapi
```

### Fresh machine (nothing installed)

```bash
# Install Claude CLI first, then:
claude auth login       # creates ~/.claude
npm install -g @digital-threads/aimux
aimux init
aimux profile add work
aimux auth login work
```

### Day-to-day usage

```bash
aimux run               # interactive picker (↑↓ + Enter)
aimux run w             # prefix match → work
aimux run o -m claude-sonnet-4-6  # one-time model override
aimux run w --resume    # flags pass through to Claude CLI
aimux status            # dashboard
aimux usage             # token usage by profile for the last 7 days
aimux usage --all       # all known transcript usage

# Set default model per profile (quote model names with special chars)
aimux profile update w -m claude-opus-4-6
aimux profile update o -m "claude-opus-4-6[1m]"
```

## Commands

| Command | Description |
|---------|-------------|
| `aimux init` | Auto-detect Claude dirs, create config, migrate profiles |
| `aimux init --source <path>` | Initialize with explicit source directory |
| `aimux status` | TUI dashboard — profiles, auth, symlink health |
| `aimux usage` | Show token usage by profile from Claude transcript metadata |
| `aimux usage --profile work --since 24h` | Show usage for one profile over a recent window |
| `aimux run [profile]` | Launch AI CLI with correct env and model |
| `aimux run` | Interactive picker — history pre-selects last used profile |
| `aimux run w` | Prefix matching — launches `work` if unambiguous |
| `aimux run work -m claude-sonnet-4-6` | Launch with model override |
| `aimux agents` | Multi-profile agent view — see and manage claude background sessions across **all** profiles in one TUI |
| `aimux profile add <name>` | Create new profile with symlinks |
| `aimux profile add <name> --api` | Create profile for 3rd-party API endpoint (interactive prompt) |
| `aimux profile update <name>` | Update model/cli settings |
| `aimux profile update <name> -e KEY=VALUE` | Set env var in profile `.env` file |
| `aimux profile update <name> --unset-env KEY` | Remove env var from profile `.env` file |
| `aimux profile list` | List all profiles |
| `aimux profile remove <name>` | Remove profile and clean up |
| `aimux profile clone <src> <name>` | Clone profile with private files |
| `aimux rebuild [profile]` | Sync symlinks and surface local shared-file conflicts |
| `aimux doctor` | Health check — broken symlinks, missing shared entries, conflicts |
| `aimux auth login <profile>` | Launch OAuth flow for a profile |
| `aimux auth status` | Show auth file status per profile |
| `aimux setup-shell` | Auto-install shell completions (bash/zsh/fish) |
| `aimux migrate isolate` | One-time migration: convert per-profile `jobs/`, `daemon/`, `projects/` symlinks into real private dirs so each profile gets its own supervisor and sessions. Safe — no data is deleted. Add `--dry-run` to preview. |

All profile commands support **prefix matching**: `aimux run w` → `work`, `aimux profile update o` → `own`.

## How It Works

```
~/.claude/          ← source of truth (your main profile)
  agents/
  skills/
  commands/
  memory/
  settings.json
  .credentials.json  ← private, stays here

~/.aimux/
  config.yaml        ← aimux config
  profiles/
    work/
      agents/ → ~/.claude/agents      ← symlink (shared)
      skills/ → ~/.claude/skills      ← symlink (shared)
      memory/ → ~/.claude/memory      ← symlink (shared)
      .credentials.json               ← real file (private, OAuth)
      .claude.json                    ← real file (private)
    myapi/
      agents/ → ~/.claude/agents      ← symlink (shared)
      .env                            ← real file (private, API credentials)
```

When you run `aimux run work`, it sets `CLAUDE_CONFIG_DIR=~/.aimux/profiles/work` and launches the CLI. Claude sees a complete config directory — shared content via symlinks, private auth locally.

For API profiles, aimux additionally loads the profile's `.env` file and injects its variables into the environment before launching.

## Config

```yaml
# ~/.aimux/config.yaml
version: 1
shared_source: /home/user/.claude

profiles:
  main:
    cli: claude
    path: /home/user/.claude
    is_source: true
  work:
    cli: claude
    model: claude-opus-4-6
    path: /home/user/.aimux/profiles/work
  myapi:                                    # 3rd-party API profile
    cli: claude
    model: claude-sonnet-4-6
    path: /home/user/.aimux/profiles/myapi  # has .env with API credentials

private:
  - .credentials.json
  - .env                  # API credentials — never symlinked, never committed
  - .claude.json
  - ...
```

For API profiles, credentials live in `~/.aimux/profiles/<name>/.env`:

```bash
# ~/.aimux/profiles/myapi/.env — do not commit
ANTHROPIC_BASE_URL=https://api.your-provider.com/v1
ANTHROPIC_AUTH_TOKEN=sk-your-token...
ANTHROPIC_MODEL=claude-sonnet-4-6
```

These are injected as environment variables when the profile is launched. The `.env` file is always private (never symlinked to the shared source).

## Requirements

- Node.js 22+
- Claude Code CLI installed

## License

MIT
