# Plugin SDK API Reference

Updated: 2026-03-22

Complete API reference for the `@unicore-geek/plugins` SDK (v0.0.2+).

## Package Exports

```typescript
// Core interfaces
import type {
  IUnicorePlugin,
  PluginContext,
  PluginManifest,
  PluginRecord,
  PluginState,
  CommandDefinition,
  CommandOption,
  CommandArgument,
  HookDefinition,
  ApiCallOptions,
  LogLevel,
  HttpMethod,
} from '@unicore-geek/plugins';

// Classes
import {
  PluginManager,
  PluginRegistry,
  PluginContextImpl,
  EventBus,
} from '@unicore-geek/plugins';

// Built-in examples
import {
  HelloWorldPlugin,
  AutoBackupPlugin,
} from '@unicore-geek/plugins';
```

---

## Types & Enums

### PluginState

```typescript
type PluginState = 'uninstalled' | 'installed' | 'loaded' | 'active' | 'inactive' | 'error';
```

### LogLevel

```typescript
type LogLevel = 'debug' | 'info' | 'warn' | 'error';
```

### HttpMethod

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
```

---

## IUnicorePlugin

The interface every plugin must implement.

```typescript
interface IUnicorePlugin {
  readonly manifest: PluginManifest;
  activate(context: PluginContext): Promise<void> | void;
  deactivate(): Promise<void> | void;
  commands?: CommandDefinition[];
  hooks?: HookDefinition[];
}
```

| Member | Type | Description |
|--------|------|-------------|
| `manifest` | `PluginManifest` | Plugin identity and metadata |
| `activate` | `(context) => void` | Called when the plugin is activated. Register commands and hooks here. |
| `deactivate` | `() => void` | Called when the plugin is deactivated. Clean up resources. |
| `commands` | `CommandDefinition[]` | Optional: pre-declared command definitions |
| `hooks` | `HookDefinition[]` | Optional: pre-declared hook definitions |

---

## PluginManifest

```typescript
interface PluginManifest {
  name: string;
  version: string;
  description: string;
  author: string | { name: string; email?: string; url?: string };
  engineVersion?: string;
  dependencies?: string[];
  keywords?: string[];
  main?: string;
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | Yes | Unique plugin identifier |
| `version` | `string` | Yes | Semver version |
| `description` | `string` | Yes | Short description |
| `author` | `string \| object` | Yes | Author name or object with name/email/url |
| `engineVersion` | `string` | No | Required SDK version (semver range, e.g., `">=0.0.2"`) |
| `dependencies` | `string[]` | No | Names of plugins this depends on |
| `keywords` | `string[]` | No | Keywords for marketplace search |
| `main` | `string` | No | Entry point file (default: `dist/index.js`) |

---

## PluginContext

Passed to `activate()`. Provides the full plugin API.

```typescript
interface PluginContext {
  registerCommand(definition: CommandDefinition): void;
  registerHook(definition: HookDefinition): void;
  getConfig(key: string): unknown;
  setConfig(key: string, value: unknown): void;
  callApi(endpoint: string, options?: ApiCallOptions): Promise<unknown>;
  emit(event: string, payload?: unknown): Promise<void>;
  log(level: LogLevel, message: string, meta?: Record<string, unknown>): void;
  dataDir: string;
  pluginName: string;
}
```

### registerCommand(definition)

Register a CLI command. The command name is auto-namespaced with the plugin name (`"greet"` → `"my-plugin:greet"`).

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | Yes | Command name |
| `description` | `string` | Yes | Help text |
| `aliases` | `string[]` | No | Alternative names |
| `options` | `CommandOption[]` | No | CLI flags |
| `arguments` | `CommandArgument[]` | No | Positional arguments |
| `handler` | `(args, options) => void` | Yes | Command implementation |

**CommandOption:**

```typescript
interface CommandOption {
  flags: string;          // e.g., "-n, --name <name>"
  description: string;
  defaultValue?: unknown;
}
```

**CommandArgument:**

```typescript
interface CommandArgument {
  name: string;
  description: string;
  required?: boolean;
}
```

### registerHook(definition)

Register an event hook.

```typescript
interface HookDefinition {
  event: string;
  handler: (payload: unknown) => Promise<void> | void;
  priority?: number;      // Lower = runs first (default: 100)
}
```

**Built-in events:**

| Event | Payload | When |
|-------|---------|------|
| `config:changed` | `{ key: string, value: unknown }` | Config key updated |
| `session:start` | `{}` | CLI session started |
| `plugin:activated` | `{ pluginName: string }` | Another plugin activated |

### getConfig(key)

Read persistent config. Supports dot-notation (`"sync.interval"`).

**Returns:** `unknown` — the stored value, or `undefined` if not set.

### setConfig(key, value)

Write persistent config.

### callApi(endpoint, options?)

Make an authenticated HTTP call to UniCore backend services.

```typescript
interface ApiCallOptions {
  method?: HttpMethod;              // default: 'GET'
  headers?: Record<string, string>;
  body?: unknown;                   // auto-serialized to JSON
  timeout?: number;                 // default: 30000ms
}
```

**Returns:** `Promise<unknown>` — parsed JSON response, or text if not JSON.

The `Authorization: Bearer <token>` header is automatically injected for UniCore API calls.

### emit(event, payload?)

Emit a custom event. All registered hooks for this event execute in priority order.

### log(level, message, meta?)

Write a log entry to `{dataDir}/plugin.log`.

| Level | Console | File |
|-------|---------|------|
| `debug` | No | Yes |
| `info` | No | Yes |
| `warn` | Yes | Yes |
| `error` | Yes | Yes |

### dataDir

`string` — absolute path to the plugin's data directory (`~/.unicore/plugins/{name}/data`). Auto-created on first access.

### pluginName

`string` — the plugin's name from its manifest.

---

## PluginRecord

Returned by `PluginManager.listPlugins()` and `PluginManager.getPlugin()`.

```typescript
interface PluginRecord {
  manifest: PluginManifest;
  state: PluginState;
  installPath: string;       // Absolute path on disk
  installedAt: string;       // ISO timestamp
  enabled: boolean;          // Auto-activate on startup
  errorMessage?: string;     // Set when state is 'error'
}
```

---

## PluginManager

Manages the full plugin lifecycle.

```typescript
import { PluginManager } from '@unicore-geek/plugins';

const manager = new PluginManager(options?: PluginManagerOptions);
```

### Constructor Options

```typescript
interface PluginManagerOptions {
  pluginsDir?: string;             // default: ~/.unicore/plugins
  apiBaseUrl?: string;             // default: http://localhost:4000/api/v1
  authToken?: () => string | undefined;
  configStore?: Map<string, unknown>;
}
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `discoverPlugins()` | `Promise<PluginRecord[]>` | Scan plugins directory for installed plugins |
| `loadPlugin(name)` | `Promise<void>` | Resolve and load the plugin module |
| `activatePlugin(name)` | `Promise<void>` | Activate the plugin (calls `activate()`) |
| `deactivatePlugin(name)` | `Promise<void>` | Deactivate the plugin (calls `deactivate()`) |
| `uninstallPlugin(name)` | `Promise<void>` | Remove plugin from disk and state |
| `installFromLocal(path)` | `Promise<PluginRecord>` | Install a plugin from a local directory |
| `activateAll()` | `Promise<void>` | Activate all enabled plugins |
| `deactivateAll()` | `Promise<void>` | Deactivate all active plugins |
| `enablePlugin(name)` | `Promise<void>` | Mark plugin for auto-activation on startup |
| `disablePlugin(name)` | `Promise<void>` | Skip plugin on startup |
| `listPlugins()` | `PluginRecord[]` | List all plugin records |
| `getPlugin(name)` | `PluginRecord \| undefined` | Get a single plugin record |
| `getCommands()` | `CommandDefinition[]` | List all registered commands across all active plugins |
| `getCommand(name)` | `CommandDefinition \| undefined` | Get a single command by full name |
| `getEventBus()` | `EventBus` | Access the event bus instance |
| `emit(event, payload?)` | `Promise<void>` | Emit an event through the event bus |

---

## PluginRegistry

Install, search, and update plugins.

```typescript
import { PluginRegistry } from '@unicore-geek/plugins';

const registry = new PluginRegistry(
  manager: PluginManager,
  registryUrl?: string,    // default: https://registry.unicore.dev/plugins
  pluginsDir?: string,
);
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `install(nameOrPath, options?)` | `Promise<PluginRecord>` | Install a plugin from local path or npm |
| `listInstalled()` | `PluginRecord[]` | List all installed plugins |
| `search(options?)` | `Promise<RegistryEntry[]>` | Search the remote plugin registry |
| `checkForUpdates()` | `Array<{plugin, latestVersion, updateAvailable}>` | Check for available updates |

### InstallOptions

```typescript
interface InstallOptions {
  force?: boolean;          // Overwrite existing (default: false)
  version?: string;         // Specific version (e.g., "1.0.5")
}
```

### SearchOptions

```typescript
interface SearchOptions {
  query?: string;           // Match name, description, keywords
  keywords?: string[];      // Filter by keyword
  limit?: number;           // Max results (default: 50)
}
```

### RegistryEntry

```typescript
interface RegistryEntry {
  name: string;
  version: string;
  description: string;
  author: string;
  keywords: string[];
  installUrl: string;
  latestVersion: string;
  publishedAt: string;
}
```

---

## EventBus

The event system used by plugins to communicate.

```typescript
import { EventBus } from '@unicore-geek/plugins';
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `register(pluginName, definition)` | `void` | Register a hook for an event |
| `unregisterPlugin(pluginName)` | `void` | Remove all hooks from a plugin |
| `emit(event, payload)` | `Promise<void>` | Execute all hooks for an event in priority order |
| `listEvents()` | `string[]` | List all event names with registered hooks |
| `listHooks(event)` | `Array<{pluginName, priority}>` | List hooks registered for an event |

Hooks execute sequentially in ascending priority order. Errors in one hook are logged but don't prevent other hooks from running.

---

## PluginContextImpl

The concrete implementation of `PluginContext`. Used internally by `PluginManager` — you typically don't instantiate this directly.

```typescript
import { PluginContextImpl } from '@unicore-geek/plugins';

const context = new PluginContextImpl(options: PluginContextOptions);
```

### PluginContextOptions

```typescript
interface PluginContextOptions {
  pluginName: string;
  dataDir: string;
  configStore: Map<string, unknown>;
  apiBaseUrl: string;
  authToken: () => string | undefined;
  onRegisterCommand: (definition: CommandDefinition) => void;
  onRegisterHook: (definition: HookDefinition) => void;
  onEmit: (event: string, payload: unknown) => Promise<void>;
}
```

---

## Permission Scopes

Declare in manifest to request access. Users see a consent prompt on install.

| Scope | Access |
|-------|--------|
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

---

## Requirements

| Dependency | Minimum Version |
|------------|----------------|
| Node.js | 20.0.0 |
| TypeScript | 5.5 |
| `@unicore-geek/plugins` | 0.0.2 |

---

## Source Code

The plugin SDK source is in the [unicore-geek](https://github.com/bemindlabs/unicore-geek) repository at `packages/plugins/`.

| File | Description |
|------|-------------|
| `src/plugin.interface.ts` | Core types and interfaces (162 lines) |
| `src/plugin-manager.ts` | Plugin lifecycle management (462 lines) |
| `src/plugin-context.impl.ts` | Context API implementation (145 lines) |
| `src/plugin-registry.ts` | Install, search, and update (228 lines) |
| `src/event-bus.ts` | Event hook system (52 lines) |
| `src/builtin-plugins/` | Hello World and Auto-Backup examples |

---

© 2026 BeMind Technology — Licensed under BSL 1.1
