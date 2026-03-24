# Virtual Office Guide

Updated: 2026-03-24

Manage your AI agent team from a visual, multi-agent workspace.

> **Pro Add-on:** The Virtual Office requires the `featVirtualOffice` license flag and a Pro or higher edition.

## Overview

The Virtual Office is a dedicated workspace for monitoring and interacting with your AI agents. It presents agents as team members in a virtual office environment, with real-time status tracking, direct messaging, and configuration controls. The Virtual Office connects to the OpenClaw WebSocket gateway to display live agent state and relay commands.

```
Virtual Office UI → OpenClaw Gateway (WS :18789 / HTTP :18790) → Agent Registry → Specialist Agents
```

## Accessing the Virtual Office

Navigate to `/virtual-office` in the dashboard, or click the **Virtual Office** link in the sidebar.

- **URL:** `https://<your-domain>/virtual-office`
- The page uses a dedicated layout under `(virtual-office)/virtual-office/page.tsx`, separate from the main dashboard shell.
- Agent data is fetched from `GET /api/proxy/openclaw/health/agents` and polled every 10 seconds for live updates.
- If the API is unreachable, the UI falls back to locally cached agent data and displays a warning banner.

## Features

The header provides three tabs — **Team Overview**, **Commander**, and **Settings** — plus a slide-in **Team Chat** panel.

### Team Overview

The default tab. It consists of two panels:

**Team Sidebar (left)** — A filterable list of all agents with status indicators.

- Filter agents by status: `ALL`, `WORKING`, or `IDLE`.
- Click an agent to open the edit modal.
- Each agent row shows a pixel avatar, name, current status, and a terminal shortcut (`>_`) that opens a direct agent terminal.
- On mobile, the sidebar slides in as an overlay.

**Office Floor Map (right)** — A 2D isometric map (1000x600) where agents appear as pixel avatars that wander between rooms.

- Three floors accessible via an elevator panel:
  - **F1: Engineering** — Standalone workstations with computer desks and bookshelves.
  - **F2: Operations** — Command center with server racks, console desks, and vending machines.
  - **F3: Executive & Quarters** — Conference room (with a large table) and bedroom (for offline agents).
- Agents are placed in rooms based on their `room` property (`standalone`, `main-office`, `conference`). Offline agents are automatically placed in the bedroom.
- Agents drift to new positions every 4.5 seconds to simulate activity.
- **Click** an agent to select it for relocation (then click the floor to place it), **double-click** to open the edit modal.
- Manual placements are locked for 30 seconds before auto-movement resumes.

### Commander

A chat interface for sending commands to individual agents.

- **Agent selector (left panel):** Lists all registered agents with their status and role. The router agent is selected by default.
- **Conversation area (right panel):** Displays the message thread with the selected agent. Messages support inline code, bold text, and code blocks.
- **Quick prompts:** Context-aware prompt buttons based on the agent type (e.g., "Generate monthly revenue report" for the finance agent, "Check system health" for ops).
- **Conversation persistence:** Messages are stored in `localStorage` per agent and auto-saved to the backend (`POST /api/v1/chat-history`) on tab switch, page leave, or manual save.
- **Keyboard shortcut:** `Ctrl+Enter` to send.
- Communication flows through the OpenClaw WebSocket on a per-agent channel (`command-{agentId}`).

### Settings

Displays all agents registered in the OpenClaw gateway with expandable configuration panels.

Each agent card shows:

- **Name, type, and state** with uptime and message count badges.
- **Capabilities** — Color-coded tags (routing, messaging, analytics, finance, security, marketing, etc.).
- **Editable fields:** Name, role, and status (`working` / `idle` / `offline`).
- **Autonomy level** — Controls how independently the agent acts:
  - `Suggest` — Agent proposes actions, user confirms.
  - `Approval` — Agent queues actions for approval.
  - `Full Auto` — Agent executes autonomously.
- **Channel assignment** — Toggle which channels the agent responds on: `web`, `email`, `slack`, `line`, `telegram`, `whatsapp`.
- Changes are saved per agent via `PUT /api/v1/settings/agent-config-{name}`.

### Team Chat

A slide-in panel (right edge) for real-time conversation with agents, toggled via the chat icon in the header.

- **General channel** (`chat-virtual-office`) for team-wide messages.
- **Direct messages** — Select a specific agent from the dropdown to open a private channel (`chat-agent-{agentId}`).
- **@mentions** — Type `@` to trigger autocomplete for agent names. Mentioned names are rendered in bold.
- **Reactions** — Hover over any message to add emoji reactions.
- **Search** — Filter messages by keyword within the current channel.
- **Sound notifications** — Toggle audible beep on incoming agent messages.
- Communication uses the same WebSocket connection as the Commander.

## Agent Management

### Adding an Agent

1. Click **+ Add Agent** in the header (visible on the Team Overview tab).
2. Fill in the modal form:
   - **Name** — Displayed in uppercase throughout the UI.
   - **Role** — Free-text description (e.g., "Communications Agent").
   - **Activity** — Current task description.
   - **Status** — `WORKING`, `IDLE`, or `OFFLINE`.
   - **Room** — Where the agent appears on the office floor: Conference Room, Command Center, or Standalone Workstation.
   - **Color** — Choose from 10 preset accent colors for the agent's avatar.
3. Click **Create Agent**. The agent is registered via `POST /api/proxy/openclaw/agents`.

### Editing an Agent

- Click an agent in the Team Sidebar, or double-click on the Office Floor Map.
- The same modal opens in edit mode with the agent's current values pre-filled.
- Click **Update Agent** to save via `PUT /api/proxy/openclaw/agents/{id}`.

### Deleting an Agent

- Open the edit modal for the agent.
- Click **Delete**, then click **Confirm?** to confirm.
- The agent is removed via `DELETE /api/proxy/openclaw/agents/{id}`.

### Agent Terminal

- Hover over an agent in the Team Sidebar and click the `>_` button.
- Opens a dedicated terminal overlay for direct interaction with that agent.

## Themes

The Virtual Office supports two visual themes, switchable via the **ThemeSelector** in the header.

### Default (Cyberpunk)

- Dark color scheme with CSS custom properties (`--bo-*`).
- Grid-line background pattern, glass-effect header, glow accents.
- Monospace typography throughout.

### RetroDesk (Pixel Art)

- Retro pixel-art aesthetic with a dedicated set of CSS custom properties (`--retrodesk-*`).
- Pixel characters for agents and a mascot in the header.
- Themed decorative elements on the office floor: pixel vending machines, flower pots, bookshelves, server racks, beds, and computer desks.
- CRT scanline overlay on the office floor map.
- Custom loading state, error state, and empty state components.
- The theme wraps components with `<RetroDeskOnly>` and `<DefaultOnly>` guards to render theme-specific markup.

## Architecture

### Data Flow

```
VirtualOfficePage
├── Fetches agents from OpenClaw gateway (GET /api/proxy/openclaw/health/agents)
├── Polls every 10 seconds for status updates
├── Caches agent data in localStorage (fallback on API failure)
└── VirtualOfficeApp
    ├── Header          — Tabs, agent count, theme selector, chat toggle
    ├── TeamSidebar     — Filterable agent list with terminal access
    ├── OfficeFloor     — 2D map with rooms, floors, and draggable agents
    ├── CommandCenter   — Per-agent chat via WebSocket (command-{agentId})
    ├── AgentSettings   — Gateway agent config with autonomy and channels
    ├── AgentModal      — Add/edit/delete agent form
    ├── AgentTerminal   — Direct agent terminal overlay
    └── ChatBox         — Team chat panel via WebSocket (chat-virtual-office)
```

### Key Types

```typescript
type AgentStatus = 'working' | 'idle' | 'offline';
type RoomId = 'conference' | 'main-office' | 'standalone' | 'bedroom';

interface VirtualOfficeAgent {
  id: string;
  name: string;
  role: string;
  status: AgentStatus;
  room: RoomId;
  activity?: string;
  color: string;
  deskItems?: string[];
}
```

### OpenClaw Gateway Integration

The Virtual Office communicates with agents through the OpenClaw WebSocket gateway:

- **Agent registry:** `GET /api/proxy/openclaw/health/agents` — returns all registered agents with state, capabilities, and heartbeat.
- **CRUD:** `POST`, `PUT`, `DELETE` on `/api/proxy/openclaw/agents/{id}`.
- **WebSocket channels:**
  - `command-{agentId}` — Commander tab conversations.
  - `chat-virtual-office` — General team chat.
  - `chat-agent-{agentId}` — Direct messages to a specific agent.

The gateway runs on ports 18789 (WebSocket) and 18790 (HTTP). See the [Agents Guide](agents.md) for full WebSocket message types.

## Related

- [Agents Guide](agents.md) — Agent architecture, tools, and routing
- [Channels Guide](channels.md) — Connect agents to messaging channels
- [Workflows Guide](workflows.md) — Trigger agents from workflow actions
