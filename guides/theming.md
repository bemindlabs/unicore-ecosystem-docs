# Theme System Guide

Updated: 2026-03-22

Customize the UniCore dashboard appearance with built-in themes or create your own.

## Overview

UniCore ships with 4 theme options:

| ID | Label | Color Scheme | Description |
|----|-------|-------------|-------------|
| `default` | Default | System | Follows OS light/dark preference |
| `dark` | Dark | Dark | Dark cyberpunk mode |
| `light` | Light | Light | Light mode |
| `retrodesk` | RetroDesk | Dark | Chinjan pixel art office theme |

The theme selector is in the **backoffice header dropdown** (top-right of the dashboard). Selecting a theme applies it immediately and reloads the page to ensure full layout changes take effect.

## How It Works

### Tailwind Dark Mode

The dashboard uses Tailwind's `darkMode: 'class'` strategy. When dark mode is active, the `dark` class is added to `<html>`:

```html
<!-- Light mode -->
<html>

<!-- Dark mode -->
<html class="dark">
```

All Tailwind dark-mode utilities (`dark:bg-slate-900`, `dark:text-white`, etc.) respond to this class.

### CSS Custom Properties

Two layers of CSS variables control colors:

**1. shadcn/ui variables** (defined in `packages/ui/src/globals.css`)

Standard component variables scoped to `:root` (light) and `.dark` (dark):

```css
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --primary: 222.2 47.4% 11.2%;
  /* ... */
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  --primary: 210 40% 98%;
  /* ... */
}
```

**2. Backoffice `--bo-*` variables** (also in `globals.css`)

The backoffice layout uses a dedicated set of variables prefixed with `--bo-` for finer control over surfaces, text hierarchy, borders, and accent opacities:

```css
:root {
  --bo-bg-deep: #ffffff;
  --bo-bg: #f8fafc;
  --bo-bg-raised: #ffffff;
  --bo-text-bright: #0f172a;
  --bo-text-accent: #0e7490;
  --bo-border: #e2e8f0;
  --bo-glow: none;
  /* ... */
}

.dark {
  --bo-bg-deep: #060a14;
  --bo-bg: #0a0e1a;
  --bo-text-bright: #e2e8f0;
  --bo-text-accent: #22d3ee;
  --bo-border: rgba(22, 78, 99, 0.3);
  --bo-glow: 0 0 15px rgba(0, 229, 255, 0.1);
  /* ... */
}
```

Key `--bo-*` variable groups:

| Group | Variables | Purpose |
|-------|----------|---------|
| Backgrounds | `--bo-bg-deep`, `--bo-bg`, `--bo-bg-raised`, `--bo-bg-elevated`, `--bo-bg-input`, `--bo-bg-bubble`, `--bo-bg-glass` | Surface layering from deepest to glass overlay |
| Text | `--bo-text-bright`, `--bo-text-accent`, `--bo-text-body`, `--bo-text-muted`, `--bo-text-dim`, `--bo-text-dimmer` | Text hierarchy from prominent to faded |
| Borders | `--bo-border`, `--bo-border-subtle`, `--bo-border-strong`, `--bo-border-accent` | Border emphasis levels |
| Accents | `--bo-accent-5` through `--bo-accent-30` | Accent color at varying opacities |
| Effects | `--bo-glow` | Glow/shadow for dark mode cyberpunk feel |

### RetroDesk Character Theme

RetroDesk is a pixel art theme scoped under the `data-character-theme` attribute on `<html>`:

```html
<html class="dark" data-character-theme="retrodesk">
```

It defines its own set of `--retrodesk-*` variables and imports pixel art fonts (`Press Start 2P`, `VT323`). The CSS is in `apps/dashboard/src/styles/retrodesk-theme.css` and is scoped entirely under `[data-character-theme="retrodesk"]`, so it has zero effect on other themes.

RetroDesk supports both light and dark variants, with separate color palettes defined using:

```css
/* Light mode */
[data-character-theme="retrodesk"] {
  --retrodesk-pink: #d81e5b;
  --retrodesk-bg: #faf8f5;
  --retrodesk-surface: #ffffff;
  --radius: 0px;  /* sharp pixel corners */
}

/* Dark mode */
[data-character-theme="retrodesk"].dark {
  --retrodesk-bg: #1a1525;
  --retrodesk-surface: #251f35;
  --retrodesk-pink: #ff6b9d;  /* bright neon pastels */
}
```

## Theme Selector

The theme selector dropdown is in the **backoffice header** (top-right corner of the dashboard). Each option displays its label, icon, and description. Selecting a theme triggers `setThemeById()`, which applies the change and reloads the page.

The available options are defined in the theme registry:

```typescript
// lib/backoffice/theme-registry.ts
export const THEME_OPTIONS: ThemeOption[] = [
  { id: 'default', label: 'Default', icon: '', description: 'Follows system preference' },
  { id: 'dark',    label: 'Dark',    icon: '', description: 'Dark cyberpunk mode' },
  { id: 'light',   label: 'Light',   icon: '', description: 'Light mode' },
  { id: 'retrodesk', label: 'RetroDesk', icon: '', description: 'Chinjan - Pixel art office' },
];
```

## Persistence

Theme selection is persisted in three places, with localStorage as the source of truth:

### localStorage Keys

| Key | Values | Purpose |
|-----|--------|---------|
| `selected-theme` | `default`, `dark`, `light`, `retrodesk` | Primary theme ID (unified key) |
| `theme` | `light`, `dark` | Legacy color scheme key (backward compat) |
| `character-theme` | `retrodesk` or absent | Legacy character theme key (backward compat) |

On load, the hook reads `selected-theme` first. If absent, it falls back to the legacy keys for backward compatibility with pre-UNC-113 installations.

### API Persistence

When a user is logged in, theme selection is also saved server-side:

```
PUT /api/v1/settings/ui-theme
Content-Type: application/json
Authorization: Bearer <token>

{ "themeId": "retrodesk" }
```

This is a fire-and-forget call -- localStorage remains the source of truth. The API persistence allows the theme to follow the user across devices.

## FOUC Prevention

To prevent a Flash of Unstyled Content (white flash before dark mode loads), the root layout includes an inline script that runs before React hydration. This script reads the stored theme from localStorage and applies the `dark` class and `data-character-theme` attribute to `<html>` synchronously, before any CSS or components render.

This ensures the correct theme is visible from the very first paint.

## Adding Custom Themes

To add a new theme option:

### Step 1: Register the Theme

Add an entry to `THEME_OPTIONS` and a case to `resolveTheme()` in `apps/dashboard/src/lib/backoffice/theme-registry.ts`:

```typescript
// theme-registry.ts

export const THEME_OPTIONS: ThemeOption[] = [
  // ... existing themes
  { id: 'ocean', label: 'Ocean', icon: '', description: 'Deep sea calm' },
];

export function resolveTheme(id: string): ResolvedTheme {
  switch (id) {
    case 'retrodesk': return { characterTheme: 'retrodesk', colorScheme: 'dark' };
    case 'light':     return { characterTheme: null, colorScheme: 'light' };
    case 'dark':      return { characterTheme: null, colorScheme: 'dark' };
    case 'ocean':     return { characterTheme: 'ocean', colorScheme: 'dark' };
    default:          return { characterTheme: null, colorScheme: 'system' };
  }
}
```

The `characterTheme` value determines the `data-character-theme` attribute. Set it to `null` if your theme only changes colors via the existing `--bo-*` variables. Set it to a string if you need a completely separate CSS scope.

### Step 2: Define CSS Variables

If you set a `characterTheme`, create a new CSS file (e.g., `apps/dashboard/src/styles/ocean-theme.css`) scoped under your attribute:

```css
[data-character-theme="ocean"] {
  --bo-bg-deep: #0a1628;
  --bo-bg: #0d1f3c;
  --bo-text-accent: #4fc3f7;
  --bo-border: rgba(79, 195, 247, 0.2);
  /* Override as many --bo-* variables as needed */
}
```

Import the CSS file in the backoffice layout so it loads when the theme is active.

### Step 3: Rebuild and Deploy

```bash
cd /path/to/unicores
docker compose --profile apps build unicore-dashboard --no-cache
docker compose --profile apps up -d unicore-dashboard
```

## useTheme() Hook API

The `useTheme()` hook (from `hooks/use-theme.ts`) is the primary interface for reading and changing the theme in React components.

```typescript
const {
  theme,
  toggleTheme,
  characterTheme,
  setCharacterTheme,
  selectedThemeId,
  setThemeById,
} = useTheme();
```

| Property | Type | Description |
|----------|------|-------------|
| `theme` | `'light' \| 'dark'` | Current effective color scheme |
| `selectedThemeId` | `string` | Active theme ID from the registry (`default`, `dark`, `light`, `retrodesk`) |
| `characterTheme` | `string \| null` | Active character theme (`'retrodesk'` or `null`) |
| `setThemeById(id)` | `(id: string) => void` | Set theme by registry ID. Applies DOM changes, persists to localStorage and API, then reloads the page |
| `toggleTheme()` | `() => void` | Legacy toggle between light and dark. Does not reload. Used by non-backoffice pages |
| `setCharacterTheme(id)` | `(id: string \| null) => void` | Legacy setter for character theme attribute. Prefer `setThemeById` |

### Usage Example

```typescript
import { useTheme } from '@/hooks/use-theme';

function ThemeStatus() {
  const { theme, selectedThemeId, setThemeById } = useTheme();

  return (
    <div>
      <p>Current: {selectedThemeId} ({theme} mode)</p>
      <button onClick={() => setThemeById('retrodesk')}>
        Switch to RetroDesk
      </button>
    </div>
  );
}
```

## Key Files

| File | Purpose |
|------|---------|
| `apps/dashboard/src/hooks/use-theme.ts` | React hook for reading and changing themes |
| `apps/dashboard/src/lib/backoffice/theme-registry.ts` | Theme definitions and resolution logic |
| `packages/ui/src/globals.css` | CSS variables for light/dark + backoffice `--bo-*` tokens |
| `apps/dashboard/src/styles/retrodesk-theme.css` | RetroDesk pixel art theme CSS |

## Related

- [Development Guide](../development-guide.md) -- Dev setup and conventions
- [Editions and Licensing](../editions/pro.md) -- White-label branding (Pro feature)
