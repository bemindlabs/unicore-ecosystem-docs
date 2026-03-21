# Plugin Development

Build custom plugins for UniCore using the `@unicore-geek/plugins` SDK. Plugins extend the platform with custom CLI commands, event hooks, integrations, and automation.

## Overview

The UniCore plugin system provides a stable, typed API for extending the platform. Plugins are distributed as npm packages and installed via the CLI or the Plugin Registry.

**SDK Package:** `@unicore-geek/plugins` (v0.0.2+)

## Prerequisites

- Node.js 20+
- TypeScript 5.5+
- pnpm 10.30+ or npm
- A running UniCore instance for testing

## Quick Start

### 1. Create a plugin project

```bash
mkdir unicore-my-plugin && cd unicore-my-plugin
npm init -y
npm install typescript @types/node --save-dev
```

Add plugin metadata to `package.json`:

```json
{
  "name": "unicore-my-plugin",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "unicoreGeek": {
    "name": "my-plugin",
    "description": "My custom UniCore plugin",
    "author": "Your Name",
    "keywords": ["custom"],
    "engineVersion": ">=0.0.2"
  },
  "scripts": {
    "build": "tsc -p tsconfig.json"
  },
  "peerDependencies": {
    "@unicore-geek/plugins": ">=0.0.2"
  },
  "devDependencies": {
    "@unicore-geek/plugins": ">=0.0.2",
    "typescript": "^5.5.0"
  }
}
```

### 2. Write the plugin

```typescript
// src/index.ts
import type {
  IUnicorePlugin,
  PluginManifest,
  PluginContext,
} from '@unicore-geek/plugins';

export class MyPlugin implements IUnicorePlugin {
  readonly manifest: PluginManifest = {
    name: 'my-plugin',
    version: '1.0.0',
    description: 'My custom UniCore plugin',
    author: 'Your Name',
    keywords: ['custom'],
  };

  activate(context: PluginContext): void {
    // Register a CLI command
    context.registerCommand({
      name: 'hello',
      description: 'Say hello',
      options: [
        { flags: '-n, --name <name>', description: 'Name to greet', defaultValue: 'World' },
      ],
      handler: async (_args, options) => {
        console.log(`Hello, ${options['name']}!`);
      },
    });

    // Register an event hook
    context.registerHook({
      event: 'config:changed',
      priority: 100,
      handler: (payload) => {
        context.log('info', 'Config changed', payload as Record<string, unknown>);
      },
    });

    context.log('info', 'my-plugin activated');
  }

  deactivate(): void {
    // Cleanup resources (timers, connections, etc.)
  }
}

export default new MyPlugin();
```

### 3. Build and install

```bash
npm run build
unicore plugins install ./path/to/unicore-my-plugin
```

## Plugin Interface

Every plugin must implement `IUnicorePlugin`:

```typescript
interface IUnicorePlugin {
  readonly manifest: PluginManifest;
  activate(context: PluginContext): Promise<void> | void;
  deactivate(): Promise<void> | void;
  commands?: CommandDefinition[];
  hooks?: HookDefinition[];
}
```

### Plugin Manifest

```typescript
interface PluginManifest {
  name: string;           // Unique plugin identifier (e.g., "crm-exporter")
  version: string;        // Semver version (e.g., "1.0.0")
  description: string;    // Short description
  author: string | { name: string; email?: string; url?: string };
  engineVersion?: string; // Required SDK version (semver range)
  dependencies?: string[]; // Other plugin names this plugin depends on
  keywords?: string[];    // For marketplace search
  main?: string;          // Entry point (default: dist/index.js)
}
```

The manifest can also be specified via the `unicoreGeek` field in `package.json`. If both are present, the class-level manifest takes precedence.

### Plugin Lifecycle

```
uninstalled → installed → loaded → active ⇄ inactive
                                      ↓
                                    error
```

| State | Description |
|-------|-------------|
| `uninstalled` | Not on disk |
| `installed` | On disk, not loaded |
| `loaded` | Module resolved, not yet activated |
| `active` | `activate()` called, commands and hooks registered |
| `inactive` | `deactivate()` called, cleaned up |
| `error` | Activation failed (check `errorMessage` on the PluginRecord) |

Plugin state is persisted to `~/.unicore/plugins/.plugin-state.json`. Transient states (`active`, `loaded`) reset to `installed` on process restart.

## Plugin Context API

The `PluginContext` is passed to `activate()` and provides the full plugin API surface:

```typescript
interface PluginContext {
  registerCommand(definition: CommandDefinition): void;
  registerHook(definition: HookDefinition): void;
  getConfig(key: string): unknown;
  setConfig(key: string, value: unknown): void;
  callApi(endpoint: string, options?: ApiCallOptions): Promise<unknown>;
  emit(event: string, payload?: unknown): Promise<void>;
  log(level: LogLevel, message: string, meta?: Record<string, unknown>): void;
  dataDir: string;       // ~/.unicore/plugins/{plugin-name}/data
  pluginName: string;
}
```

### Registering Commands

```typescript
context.registerCommand({
  name: 'export',                          // Auto-namespaced: "my-plugin:export"
  description: 'Export data to CSV',
  aliases: ['exp'],
  options: [
    { flags: '-o, --output <path>', description: 'Output file path', defaultValue: './export.csv' },
    { flags: '-f, --format <type>', description: 'Output format (csv|json)' },
  ],
  arguments: [
    { name: 'entity', description: 'Entity to export (contacts|orders)', required: true },
  ],
  handler: async (args, options) => {
    const entity = args['entity'] as string;
    const output = options['output'] as string;
    // ... implementation
  },
});
```

Commands are automatically namespaced with the plugin name. A command named `export` in plugin `crm-exporter` becomes `crm-exporter:export` in the CLI:

```bash
unicore crm-exporter:export contacts --output ./contacts.csv
```

### Registering Event Hooks

```typescript
context.registerHook({
  event: 'config:changed',
  handler: (payload) => {
    const { key, value } = payload as { key: string; value: unknown };
    context.log('info', `Config key "${key}" changed`);
  },
  priority: 50,  // Lower = runs first (default: 100)
});
```

**Built-in events:**

| Event | Payload | Description |
|-------|---------|-------------|
| `config:changed` | `{ key: string, value: unknown }` | A config key was updated |
| `session:start` | `{}` | CLI session started |
| `plugin:activated` | `{ pluginName: string }` | Another plugin was activated |

**Custom events:**

```typescript
// Emit from your plugin
await context.emit('my-plugin:data-synced', { count: 42, duration: 1200 });

// Another plugin can hook into it
context.registerHook({
  event: 'my-plugin:data-synced',
  handler: (payload) => { /* ... */ },
});
```

Hooks execute sequentially in priority order. Errors in one hook are logged but don't block other hooks.

### Configuration

Read and write persistent per-plugin configuration:

```typescript
// Write config
context.setConfig('apiKey', 'sk-...');
context.setConfig('sync.interval', 300);

// Read config (supports dot-notation)
const apiKey = context.getConfig('apiKey') as string;
const interval = context.getConfig('sync.interval') as number;
```

Config is stored in the plugin's config store and persisted across sessions.

### API Calls

Make authenticated HTTP calls to UniCore backend services:

```typescript
// GET request
const contacts = await context.callApi('/crm/contacts');

// POST request
const result = await context.callApi('/crm/contacts', {
  method: 'POST',
  body: { name: 'Acme Corp', email: 'hello@acme.com', type: 'LEAD' },
  timeout: 10000,
});

// Full URL also supported
const external = await context.callApi('https://api.example.com/data', {
  method: 'GET',
  headers: { 'X-API-Key': 'my-key' },
});
```

The `Authorization` header is automatically injected for calls to the UniCore API when a token is available. Default timeout is 30 seconds.

### Logging

```typescript
context.log('debug', 'Processing item', { itemId: '123' });
context.log('info', 'Sync complete', { count: 42 });
context.log('warn', 'Rate limit approaching', { remaining: 5 });
context.log('error', 'Failed to connect', { error: err.message });
```

Logs are written to `{dataDir}/plugin.log` with ISO timestamps. Warnings and errors also print to the console.

### Data Directory

Each plugin gets an isolated data directory for file storage:

```typescript
import { writeFileSync, readFileSync } from 'fs';
import { join } from 'path';

// context.dataDir = ~/.unicore/plugins/{plugin-name}/data
const cachePath = join(context.dataDir, 'cache.json');
writeFileSync(cachePath, JSON.stringify(data));
```

## Plugin Manager

The `PluginManager` handles the full plugin lifecycle:

```typescript
import { PluginManager } from '@unicore-geek/plugins';

const manager = new PluginManager({
  pluginsDir: '~/.unicore/plugins',
  apiBaseUrl: 'http://localhost:4000/api/v1',
  authToken: () => process.env.UNICORE_TOKEN,
});

// Discover and activate all installed plugins
const plugins = await manager.discoverPlugins();
await manager.activateAll();

// Query
manager.listPlugins();                    // All plugin records
manager.getPlugin('my-plugin');           // Single plugin
manager.getCommands();                    // All registered commands
manager.getCommand('my-plugin:hello');    // Single command

// Lifecycle
await manager.loadPlugin('my-plugin');
await manager.activatePlugin('my-plugin');
await manager.deactivatePlugin('my-plugin');
await manager.uninstallPlugin('my-plugin');

// Enable/disable auto-activation
await manager.enablePlugin('my-plugin');
await manager.disablePlugin('my-plugin');
```

## Plugin Registry

Install and search for plugins:

```typescript
import { PluginRegistry } from '@unicore-geek/plugins';

const registry = new PluginRegistry(manager);

// Install from local path
await registry.install('./path/to/plugin');

// Install from npm
await registry.install('@unicore/plugin-analytics');

// Install specific version
await registry.install('@unicore/plugin-analytics', { version: '1.2.0' });

// Force reinstall
await registry.install('my-plugin', { force: true });

// Search the registry
const results = await registry.search({ query: 'crm', limit: 10 });

// Check for updates
const updates = registry.checkForUpdates();
```

## Permission Scopes

Declare required permissions in your manifest. Users see a consent prompt on install.

| Permission | Access |
|-----------|--------|
| `crm:read` | Read contacts, companies, deals |
| `crm:write` | Create/update/delete CRM records |
| `erp:read` | Read invoices, products, orders |
| `erp:write` | Create/update ERP records |
| `files:read` | Read uploaded files and RAG documents |
| `files:write` | Upload files and trigger indexing |
| `workflows:read` | Read workflow definitions |
| `workflows:execute` | Trigger workflow runs |
| `agents:chat` | Send messages to AI agents |
| `settings:read` | Read system settings |
| `settings:write` | Modify system settings (requires OWNER role) |

## Examples

### Hello World

A minimal plugin that registers a single CLI command:

```typescript
import type { IUnicorePlugin, PluginManifest, PluginContext } from '@unicore-geek/plugins';

export class HelloWorldPlugin implements IUnicorePlugin {
  readonly manifest: PluginManifest = {
    name: 'hello-world',
    version: '1.0.0',
    description: 'A simple greeting plugin',
    author: 'UniCore Team',
    keywords: ['example', 'greeting'],
  };

  activate(context: PluginContext): void {
    context.registerCommand({
      name: 'greet',
      description: 'Print a personalised greeting',
      options: [
        { flags: '-n, --name <name>', description: 'Name to greet', defaultValue: 'World' },
        { flags: '-l, --loud', description: 'Print in uppercase' },
      ],
      handler: async (_args, options) => {
        let message = `Hello, ${options['name']}! Greetings from UniCore Geek.`;
        if (options['loud']) message = message.toUpperCase();
        console.log(message);
      },
    });

    context.log('info', 'hello-world plugin activated.');
  }

  deactivate(): void {}
}

export default new HelloWorldPlugin();
```

```bash
unicore hello-world:greet --name Developer --loud
# HELLO, DEVELOPER! GREETINGS FROM UNICORE GEEK.
```

### Auto-Backup

A plugin that hooks into config changes and creates automatic snapshots:

```typescript
import type { IUnicorePlugin, PluginManifest, PluginContext } from '@unicore-geek/plugins';
import { writeFileSync, readdirSync } from 'fs';
import { join } from 'path';

export class AutoBackupPlugin implements IUnicorePlugin {
  readonly manifest: PluginManifest = {
    name: 'auto-backup',
    version: '1.0.0',
    description: 'Automatically backs up config on changes',
    author: 'UniCore Team',
    keywords: ['backup', 'config', 'automation'],
  };

  activate(context: PluginContext): void {
    const maxBackups = (context.getConfig('plugins.autoBackup.maxBackups') as number) ?? 10;

    // Hook: auto-backup on config change
    context.registerHook({
      event: 'config:changed',
      priority: 50,
      handler: (payload) => {
        const ts = new Date().toISOString().replace(/:/g, '-');
        const entry = { timestamp: ts, event: 'config:changed', payload };
        writeFileSync(join(context.dataDir, `backup-${ts}.json`), JSON.stringify(entry, null, 2));
        context.log('info', `Config backup created: ${ts}`);
      },
    });

    // Command: list backups
    context.registerCommand({
      name: 'list-backups',
      description: 'List all config backups',
      handler: async () => {
        const files = readdirSync(context.dataDir).filter(f => f.startsWith('backup-'));
        console.log(`\nBackups (${files.length}):`);
        files.forEach(f => console.log(`  ${f}`));
      },
    });

    context.log('info', 'auto-backup plugin activated.');
  }

  deactivate(): void {}
}

export default new AutoBackupPlugin();
```

## Testing

### Unit Tests with Jest

```typescript
import { MyPlugin } from '../src';

describe('MyPlugin', () => {
  const mockContext = {
    registerCommand: jest.fn(),
    registerHook: jest.fn(),
    getConfig: jest.fn(),
    setConfig: jest.fn(),
    callApi: jest.fn(),
    emit: jest.fn(),
    log: jest.fn(),
    dataDir: '/tmp/test-plugin-data',
    pluginName: 'my-plugin',
  };

  it('registers commands on activate', () => {
    const plugin = new MyPlugin();
    plugin.activate(mockContext as any);

    expect(mockContext.registerCommand).toHaveBeenCalledWith(
      expect.objectContaining({ name: 'hello' })
    );
  });

  it('command handler produces correct output', async () => {
    const plugin = new MyPlugin();
    plugin.activate(mockContext as any);

    const cmd = mockContext.registerCommand.mock.calls[0][0];
    const spy = jest.spyOn(console, 'log');
    await cmd.handler({}, { name: 'Test' });
    expect(spy).toHaveBeenCalledWith(expect.stringContaining('Test'));
    spy.mockRestore();
  });
});
```

### Integration Testing

```bash
# Build
npm run build

# Install locally for testing
unicore plugins install ./

# Run commands
unicore my-plugin:hello --name "Test"

# Uninstall after testing
unicore plugins uninstall my-plugin
```

## Publishing

### To npm (public)

```bash
npm run build
npm publish --access public
```

Users install with:

```bash
unicore plugins install your-package-name
```

### Private registry

For internal plugins, use a private npm registry (Verdaccio, GitHub Packages, or npm Enterprise).

### Plugin Marketplace

The UniCore Plugin Marketplace launches Q3 2026 with:

- Browse and search plugins by category
- One-click install from the marketplace
- Plugin ratings and reviews
- Revenue sharing for creators (70% creator / 30% platform)
- Stripe Connect powered payments

## File Structure

```
~/.unicore/plugins/
├── .plugin-state.json            # Persistent state for all plugins
├── my-plugin/
│   ├── package.json              # Plugin manifest
│   ├── dist/
│   │   └── index.js              # Compiled entry point
│   └── data/                     # Plugin data directory (auto-created)
│       └── plugin.log            # Plugin log file
└── another-plugin/
    └── ...
```

## Related Documentation

- [Plugin SDK API Reference](../api-reference/plugin-sdk.md) — Full type reference
- [Client SDK Reference](../api-reference/sdk.md) — REST API client
- [Geek Edition](../editions/geek.md) — Geek edition overview
- [Contributing](./contributing.md) — Contribution guidelines

---

© 2026 BeMind Technology — Licensed under BSL 1.1
