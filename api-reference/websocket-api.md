# WebSocket API Reference — OpenClaw Gateway

Updated: 2026-03-22

OpenClaw is the multi-agent WebSocket hub that coordinates communication between AI agents and the broader UniCore platform. It runs on two ports:

| Port | Purpose |
|------|---------|
| `18789` | WebSocket server (agent connections) |
| `18790` | HTTP server (health checks only) |

**WebSocket URL (local)**: `ws://localhost:18789`
**Production**: `wss://unicore.example.com/ws` (proxied by Nginx)

---

## Connection

Connect using any standard WebSocket client. No handshake authentication is required at the transport layer — agents identify themselves via an `agent:register` message after connecting.

```typescript
const ws = new WebSocket('ws://localhost:18789');

ws.on('open', () => {
  // Send agent:register immediately after connection
});
```

Each connection is assigned an internal `socketId` (UUID v4) by the server. When a connection is closed, the associated agent is automatically unregistered.

---

## Message Envelope

All messages — in both directions — share a common envelope:

```typescript
interface BaseMessage {
  type: MessageType;       // Message type identifier
  messageId: string;       // UUID v4, unique per message
  timestamp: string;       // ISO 8601 datetime
  payload?: unknown;       // Type-specific payload
}
```

Generate `messageId` as a UUID v4 on the client side. The server uses it to correlate acknowledgements.

```json
{
  "type": "agent:register",
  "messageId": "7f3e4a2b-1c5d-4e8f-9a0b-2c3d4e5f6a7b",
  "timestamp": "2026-01-15T10:00:00.000Z",
  "payload": { ... }
}
```

---

## Message Types

### Outgoing (client → server)

| Type | Description |
|------|-------------|
| `agent:register` | Register the agent and its capabilities |
| `agent:unregister` | Gracefully deregister before disconnecting |
| `agent:heartbeat` | Keep the agent alive (send every 30 s) |
| `agent:state` | Report state change (`running` / `idle`) |
| `message:direct` | Send a message to a specific agent |
| `message:broadcast` | Send a message to all connected agents |
| `message:publish` | Publish to a named channel |
| `message:subscribe` | Subscribe to a named channel |
| `message:unsubscribe` | Unsubscribe from a named channel |
| `system:ping` | Ping the server |

### Incoming (server → client)

| Type | Description |
|------|-------------|
| `system:ack` | Acknowledgement for a successfully processed message |
| `system:error` | Error response for a failed operation |
| `system:pong` | Response to `system:ping` |
| `message:direct` | Inbound direct message from another agent |
| `message:broadcast` | Inbound broadcast from another agent |
| `message:publish` | Inbound channel message |

---

## Agent Registration

### agent:register

Register the connecting process as a named agent with declared capabilities.

**Request**

```json
{
  "type": "agent:register",
  "messageId": "7f3e4a2b-1c5d-4e8f-9a0b-2c3d4e5f6a7b",
  "timestamp": "2026-01-15T10:00:00.000Z",
  "payload": {
    "agentId": "crm-agent-01",
    "name": "CRM Agent",
    "agentType": "crm",
    "version": "1.0.0",
    "capabilities": [
      {
        "name": "contact.lookup",
        "version": "1.0.0",
        "description": "Look up a contact by email or ID",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": { "type": "string" }
          }
        },
        "outputSchema": {
          "type": "object",
          "properties": {
            "contact": { "type": "object" }
          }
        }
      }
    ],
    "tags": ["crm", "contacts"]
  }
}
```

**Response** — `system:ack`

```json
{
  "type": "system:ack",
  "messageId": "a1b2c3d4-...",
  "timestamp": "2026-01-15T10:00:00.001Z",
  "payload": {
    "originalMessageId": "7f3e4a2b-...",
    "result": {
      "agentId": "crm-agent-01",
      "state": "running"
    }
  }
}
```

**Error** — `system:error` with code `REGISTRATION_FAILED` if `agentId` is already registered.

---

### agent:unregister

Deregister an agent before disconnecting.

**Request**

```json
{
  "type": "agent:unregister",
  "messageId": "b2c3d4e5-...",
  "timestamp": "2026-01-15T10:05:00.000Z",
  "payload": {
    "agentId": "crm-agent-01",
    "reason": "Scheduled shutdown"
  }
}
```

**Response** — `system:ack`

```json
{
  "type": "system:ack",
  "messageId": "c3d4e5f6-...",
  "timestamp": "2026-01-15T10:05:00.001Z",
  "payload": {
    "originalMessageId": "b2c3d4e5-...",
    "result": { "agentId": "crm-agent-01" }
  }
}
```

---

## Heartbeats

### agent:heartbeat

Send every 30 seconds to prevent the server from marking the agent as stale. Agents that miss heartbeats are automatically evicted.

**Request**

```json
{
  "type": "agent:heartbeat",
  "messageId": "d4e5f6a7-...",
  "timestamp": "2026-01-15T10:00:30.000Z",
  "payload": {
    "agentId": "crm-agent-01"
  }
}
```

**Response** — `system:ack`

---

## State Changes

### agent:state

Report a lifecycle state change.

**Request**

```json
{
  "type": "agent:state",
  "messageId": "e5f6a7b8-...",
  "timestamp": "2026-01-15T10:01:00.000Z",
  "payload": {
    "agentId": "crm-agent-01",
    "state": "idle"
  }
}
```

Valid states: `running`, `idle`.

**Response** — `system:ack`

---

## Messaging

### message:direct

Send a message directly to a specific agent by ID.

**Request**

```json
{
  "type": "message:direct",
  "messageId": "f6a7b8c9-...",
  "timestamp": "2026-01-15T10:02:00.000Z",
  "payload": {
    "fromAgentId": "orchestrator-01",
    "toAgentId": "crm-agent-01",
    "topic": "contact.lookup",
    "data": {
      "query": "jane@example.com"
    },
    "correlationId": "task-abc-123"
  }
}
```

The target agent receives the same `message:direct` envelope on its WebSocket connection. Use `correlationId` to match responses to originating requests.

---

### message:broadcast

Send a message to all currently connected agents.

**Request**

```json
{
  "type": "message:broadcast",
  "messageId": "a7b8c9d0-...",
  "timestamp": "2026-01-15T10:03:00.000Z",
  "payload": {
    "fromAgentId": "orchestrator-01",
    "topic": "system.reload",
    "data": {
      "reason": "Config updated"
    }
  }
}
```

---

### message:subscribe / message:unsubscribe

Subscribe to a named channel to receive published messages.

**Subscribe request**

```json
{
  "type": "message:subscribe",
  "messageId": "b8c9d0e1-...",
  "timestamp": "2026-01-15T10:04:00.000Z",
  "payload": {
    "agentId": "crm-agent-01",
    "channel": "orders.created"
  }
}
```

**Unsubscribe request**

```json
{
  "type": "message:unsubscribe",
  "messageId": "c9d0e1f2-...",
  "timestamp": "2026-01-15T10:04:30.000Z",
  "payload": {
    "agentId": "crm-agent-01",
    "channel": "orders.created"
  }
}
```

---

### message:publish

Publish a message to a channel. All agents subscribed to that channel receive it.

**Request**

```json
{
  "type": "message:publish",
  "messageId": "d0e1f2a3-...",
  "timestamp": "2026-01-15T10:05:00.000Z",
  "payload": {
    "fromAgentId": "erp-agent-01",
    "channel": "orders.created",
    "data": {
      "orderId": "550e8400-...",
      "contactId": "550e8400-...",
      "totalAmount": 299.97
    },
    "correlationId": "event-xyz"
  }
}
```

---

## Ping / Pong

### system:ping

**Request**

```json
{
  "type": "system:ping",
  "messageId": "e1f2a3b4-...",
  "timestamp": "2026-01-15T10:06:00.000Z"
}
```

**Response** — `system:pong`

```json
{
  "type": "system:pong",
  "messageId": "f2a3b4c5-...",
  "timestamp": "2026-01-15T10:06:00.001Z",
  "payload": {
    "originalMessageId": "e1f2a3b4-...",
    "timestamp": "2026-01-15T10:06:00.001Z"
  }
}
```

---

## Error Responses

When an operation fails, the server returns `system:error`:

```json
{
  "type": "system:error",
  "messageId": "a3b4c5d6-...",
  "timestamp": "2026-01-15T10:07:00.000Z",
  "payload": {
    "originalMessageId": "7f3e4a2b-...",
    "code": "AGENT_NOT_FOUND",
    "message": "Agent crm-agent-99 not registered"
  }
}
```

### Error Codes

| Code | Description |
|------|-------------|
| `REGISTRATION_FAILED` | Agent ID already in use or invalid payload |
| `AGENT_NOT_FOUND` | Target agent is not registered |
| `UNAUTHORIZED` | Message from unregistered agent |
| `INVALID_MESSAGE` | Malformed envelope or unknown message type |

---

## Agent Lifecycle

```
connect
  └── agent:register  →  state: spawning → running
        └── agent:heartbeat (every 30 s)
        └── agent:state (idle ↔ running as needed)
        └── messaging (direct / broadcast / channels)
  └── agent:unregister  →  state: terminated
disconnect (auto-unregisters if not already done)
```

---

## Default Agents

OpenClaw auto-registers 9 built-in agents on startup. These handle core platform tasks including CRM, inventory, AI orchestration, and RAG retrieval. Custom agents connect on top of these defaults.

---

## HTTP Health Check

```
GET http://localhost:18790/health
```

**Response** `200 OK`

```json
{
  "status": "ok",
  "connectedAgents": 9,
  "uptime": 3600
}
```
