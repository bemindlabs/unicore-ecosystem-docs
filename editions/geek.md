# UniCore Geek Edition

Updated: 2026-03-22

The Geek Edition is a terminal-first variant of UniCore built for developers, power users, and CLI enthusiasts. There is no web dashboard — all interaction happens through a CLI, TUI (terminal user interface), or REPL console. It includes a game mode and a plugin API for extending the platform.

## Overview

| Attribute | Value |
|-----------|-------|
| **License** | Business Source License 1.1 (BSL 1.1) |
| **Repository** | Private (available with license) |
| **Price** | Free (Community base) / Pro license for extended agents |
| **Hosting** | Self-hosted |
| **Edition flag** | `UNICORE_EDITION: geek` |

Geek Edition runs the same backend services as Community or Pro but replaces the Next.js dashboard entirely with terminal interfaces.

## Core Interfaces

### CLI (`unicore-cli`)

A full-featured command-line interface for all platform operations.

```bash
# Install globally
npm install -g @unicore-geek/cli

# Authenticate
unicore login --host http://localhost:4000 --email <your-admin-email>

# ERP operations
unicore crm list contacts
unicore crm create contact --name "Acme Corp" --email "hello@acme.com"
unicore inventory list products --low-stock
unicore invoice create --contact "Acme Corp" --amount 1500

# Agent interactions
unicore agent chat --agent assistant
unicore agent run --agent erp --prompt "Show me overdue invoices"

# Workflow management
unicore workflow list
unicore workflow trigger --id wf_01HXYZ --input '{"order_id": "123"}'

# RAG / knowledge base
unicore rag upload --file ./docs/manual.pdf --namespace support
unicore rag search --query "how to reset password"
```

Run `unicore --help` or `unicore <command> --help` for full usage.

### TUI (Terminal UI)

An interactive terminal UI built with [Ink](https://github.com/vadimdemedes/ink) (React for CLI).

```bash
# Launch the TUI
unicore tui

# Or launch a specific module
unicore tui --module crm
unicore tui --module agents
unicore tui --module workflows
```

The TUI features:
- Keyboard-driven navigation (vim-style `j/k`, arrow keys, `/` to search)
- Split-pane layout — list on the left, detail/editor on the right
- Live-updating dashboards using polling or Server-Sent Events
- Color-coded status indicators and sparkline charts

### REPL Console

An interactive JavaScript/TypeScript REPL with full SDK access.

```bash
# Launch the REPL
unicore repl

# Inside the REPL:
> const contacts = await crm.listContacts({ limit: 10 })
> contacts.forEach(c => console.log(c.name, c.email))

> const result = await agents.chat('assistant', 'What is my revenue this month?')
> console.log(result.message)

> await workflows.trigger('wf_01HXYZ', { order_id: '123' })
```

The REPL supports:
- Tab completion for all SDK methods
- History persistence across sessions (`~/.unicore/repl_history`)
- `.save` and `.load` commands for script snippets
- TypeScript evaluation via `ts-node`

## Game Mode

Game Mode is a productivity gamification layer that overlays all Geek Edition interfaces.

```bash
# Enable game mode
unicore config set game-mode true

# View stats
unicore game stats
unicore game leaderboard
```

### Features

- **XP System** — Earn experience points for every action (creating contacts, completing workflows, resolving support tickets)
- **Achievements** — Unlock badges for milestones (e.g., "100 invoices processed", "First agent deployed")
- **Streaks** — Daily usage streaks with multiplier bonuses
- **Level System** — Levels 1–100 with cosmetic rewards (custom TUI themes, ASCII art on login)
- **Leaderboard** — Team leaderboard showing top contributors by XP
- **Quest System** — Daily and weekly quests with objectives tied to real work tasks

Game data is stored in the main PostgreSQL database under a dedicated `game` schema.

## Plugin API

Geek Edition exposes a plugin API for extending the CLI, TUI, and REPL.

### Plugin Structure

```
my-plugin/
├── package.json         ← Plugin manifest
├── index.ts             ← Entry point
├── commands/            ← CLI command extensions
├── screens/             ← TUI screen extensions
└── repl/                ← REPL binding extensions
```

### Plugin Manifest (`package.json`)

```json
{
  "name": "@my-org/unicore-plugin-example",
  "version": "1.0.0",
  "unicorePlugin": {
    "name": "example",
    "version": "1.0.0",
    "minGeekVersion": "1.0.0",
    "commands": ["commands/index.ts"],
    "screens": ["screens/index.tsx"],
    "replBindings": ["repl/bindings.ts"],
    "permissions": ["crm:read", "crm:write", "workflows:execute"]
  }
}
```

### Plugin Lifecycle

```typescript
import { GeekPlugin, PluginContext } from '@unicore-geek/plugin-sdk';

export default class ExamplePlugin implements GeekPlugin {
  async onLoad(ctx: PluginContext): Promise<void> {
    ctx.logger.info('Example plugin loaded');
    // Register CLI commands
    ctx.cli.register('example:hello', this.helloCommand.bind(this));
  }

  async onUnload(): Promise<void> {
    // Cleanup
  }

  private async helloCommand(args: string[], ctx: PluginContext) {
    ctx.output.print(`Hello from example plugin! Args: ${args.join(', ')}`);
  }
}
```

### Installing Plugins

```bash
# Install from npm
unicore plugin install @my-org/unicore-plugin-example

# Install from local path
unicore plugin install ./path/to/my-plugin

# List installed plugins
unicore plugin list

# Remove a plugin
unicore plugin remove @my-org/unicore-plugin-example
```

Plugins are stored in `~/.unicore/plugins/` and loaded at startup.

## Configuration

Geek Edition is configured via `~/.unicore/config.yaml`:

```yaml
host: http://localhost:4000
email: <your-admin-email>
token: <jwt-token>  # Set automatically on login

tui:
  theme: dracula        # dracula | solarized | gruvbox | nord | catppuccin
  vim_keys: true
  refresh_interval: 5   # seconds

repl:
  history_file: ~/.unicore/repl_history
  typescript: true

game_mode: false
```

## Differences from Dashboard Editions

| Feature | Community / Pro | Geek |
|---------|----------------|------|
| Web Dashboard | Yes | No |
| CLI | Basic | Full-featured |
| TUI | No | Yes |
| REPL | No | Yes |
| Game Mode | No | Yes |
| Plugin API | No | Yes |
| All backend services | Yes | Yes |
| API Gateway | Yes | Yes |
| ERP, AI, RAG | Yes | Yes |

The backend API, ERP, AI Engine, RAG, and all other services run identically. Geek Edition simply replaces the frontend.

## Installation

```bash
# Clone the Geek Edition repo (requires license — request access at https://unicore.bemind.tech/get-started)
git clone https://github.com/bemindlabs/unicore-geek.git
cd unicore-geek

# Install dependencies
pnpm install

# Start backend services (same as Community)
docker compose --profile apps up -d

# Install the CLI globally
pnpm build
npm install -g ./packages/cli

# Login
unicore login --host http://localhost:4000 \
  --email <your-admin-email> --password <your-admin-password>

# Launch the TUI
unicore tui
```

## Related Documentation

- [Plugin Development Guide](../development/plugin-development.md)
- [Community Edition](./community.md)
- [Pro Edition](./pro.md)
- [Local Setup](../development/local-setup.md)

---

© 2026 BeMind Technology — Licensed under BSL 1.1
