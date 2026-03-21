# Messaging Channels Guide

Connect UniCore agents to external messaging platforms — Telegram, LINE, Slack, WhatsApp, and more.

## Overview

UniCore Pro supports 10 messaging channel adapters. Each channel normalizes messages into a unified `ChannelMessage` format, enabling agents to respond consistently across all platforms.

```
End User (Telegram/LINE/Slack/...) → Webhook → Channel Adapter → Normalize → Agent Binding → AI Agent → Response → Channel Adapter → End User
```

## Supported Channels

### Social Channels

| Channel | Provider | Auth Method |
|---------|----------|-------------|
| **Telegram** | Telegram Bot API | Bot Token |
| **LINE** | LINE Messaging API | Channel Access Token + Secret |
| **Facebook** | Meta Graph API v19.0 | Page Access Token + App Secret |
| **Instagram** | Meta Graph API v19.0 | Access Token + App Secret |
| **WhatsApp** | Meta WhatsApp Business Cloud API | Access Token + Phone Number ID |
| **TikTok** | TikTok Open API v2 | Client Key + Secret + Access Token |

### Team Channels

| Channel | Provider | Auth Method |
|---------|----------|-------------|
| **Slack** | Slack Web API + Events API | Bot Token + Signing Secret |
| **Discord** | Discord REST API v10 | Bot Token + Application ID |
| **Email** | SMTP + IMAP | SMTP credentials + optional IMAP |
| **WebChat** | Embedded widget | API Key |

## Quick Start

### 1. Telegram Setup

**Prerequisites:** Create a bot via [@BotFather](https://t.me/BotFather) on Telegram.

**Configuration:** Add your Telegram Bot Token and webhook secret via the dashboard **Settings → Channels** page.

**Webhook endpoint:** `POST /webhooks/telegram`

The webhook validates the `X-Telegram-Bot-Api-Secret-Token` header against the configured webhook secret.

**Connection modes:**
- **Webhook** (recommended): Register webhook URL with Telegram; messages arrive instantly
- **Polling**: Adapter polls `getUpdates` every 2 seconds; useful for local development

### 2. LINE Setup

**Prerequisites:** Create a LINE Messaging API channel at [LINE Developers Console](https://developers.line.biz/console/).

**Configuration:** Add your LINE Channel ID, Channel Secret, and Channel Access Token via the dashboard **Settings → Channels** page.

**Webhook endpoint:** `POST /webhooks/line`

The webhook validates `X-Line-Signature` using HMAC-SHA256 with the channel secret (timing-safe comparison). LINE requires a 200 response within 1 second.

### 3. Slack Setup

**Prerequisites:** Create a Slack App at [api.slack.com/apps](https://api.slack.com/apps) with Bot Token Scopes.

**Configuration:**
```json
{
  "type": "slack",
  "channelId": "slack-support",
  "displayName": "Slack Support",
  "botToken": "xoxb-...",
  "signingSecret": "<your-slack-signing-secret>",
  "defaultChannel": "#support"
}
```

**Signature validation:** `v0=HMAC-SHA256(v0:timestamp:body)` — rejects requests older than 5 minutes.

**Socket Mode:** Set `socketMode: true` and provide `appToken` for development without a public URL.

### 4. WhatsApp Setup

**Prerequisites:** Set up WhatsApp Business via [Meta Business Suite](https://business.facebook.com).

**Configuration:**
```json
{
  "type": "whatsapp",
  "channelId": "whatsapp-main",
  "displayName": "WhatsApp",
  "accessToken": "<your-whatsapp-access-token>",
  "phoneNumberId": "<your-phone-number-id>",
  "businessAccountId": "<your-business-account-id>",
  "verifyToken": "<your-verify-token>"
}
```

### 5. Email Setup

**Configuration:**
```json
{
  "type": "email",
  "channelId": "email-support",
  "displayName": "Support Email",
  "from": "support@example.com",
  "fromName": "Support Team",
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "secure": false,
    "auth": { "user": "...", "pass": "..." }
  },
  "imap": {
    "host": "imap.example.com",
    "port": 993,
    "secure": true,
    "auth": { "user": "...", "pass": "..." },
    "mailbox": "INBOX",
    "pollingInterval": 30000
  }
}
```

### 6. WebChat Setup

Embed a chat widget on your website:

```json
{
  "type": "webchat",
  "channelId": "webchat-main",
  "displayName": "Website Chat",
  "apiKey": "<your-webchat-api-key>",
  "allowedOrigins": ["https://example.com"],
  "greeting": "Hello! How can I help?",
  "sessionTimeout": 1800000,
  "persistHistory": true
}
```

## Channel Registration

### Static Configuration (NestJS Module)

```typescript
import { ChannelModule } from '@unicore/channels';

@Module({
  imports: [
    ChannelModule.forRoot([
      {
        type: 'telegram',
        channelId: 'telegram-main',
        displayName: 'Telegram Bot',
        botToken: config.get('TELEGRAM_BOT_TOKEN'),
      },
      {
        type: 'line',
        channelId: 'line-main',
        displayName: 'LINE Official',
        channelAccessToken: config.get('LINE_CHANNEL_ACCESS_TOKEN'),
        channelSecret: config.get('LINE_CHANNEL_SECRET'),
      },
    ]),
  ],
})
```

### Async Configuration

```typescript
ChannelModule.forRootAsync({
  useFactory: (config: ConfigService) => config.get('channels'),
  inject: [ConfigService],
})
```

Channels with `enabled: false` are skipped during initialization.

## Agent Binding

Bind AI agents to channels so inbound messages are routed to the right agent:

```typescript
import { AgentBindingService } from '@unicore/channels';

// Bind Comms Agent to Telegram and LINE
agentBinding.bind('comms-agent', 'telegram-main');
agentBinding.bind('comms-agent', 'line-main');

// Bind Finance Agent to Slack only
agentBinding.bind('finance-agent', 'slack-support');
```

### Binding Features

- **One-to-many:** One agent can handle multiple channels
- **Many-to-one:** Multiple agents can receive from the same channel
- **Sender filtering:** Optional `senderFilter` to route only specific users
- **Outbound routing:** Agents send replies through their bound channels automatically

### Message Flow

**Inbound:**
1. External platform sends webhook to UniCore
2. Channel adapter validates signature and normalizes message
3. AgentBindingService routes to all bound agents via Observable stream
4. Agent processes message and generates response

**Outbound:**
1. Agent calls `routeOutbound(agentId, message)`
2. If `message.channelId` is set, routes only to that channel
3. Otherwise routes to all channels bound to the agent

## Unified Message Format

All channels normalize to a common `ChannelMessage`:

```typescript
{
  id: "uuid",
  channel: "telegram",           // Channel type
  channelId: "telegram-main",    // Adapter instance
  direction: "inbound",
  text: "Hello!",
  contentType: "text",           // text, image, video, audio, file, location, etc.
  sender: {
    id: "123456",
    name: "John",
    username: "john_doe"
  },
  conversation: {
    id: "chat-789",
    type: "direct"               // direct, group, channel, thread
  },
  timestamp: "2026-03-18T12:00:00Z",
  status: "delivered"
}
```

**Supported content types:** text, image, video, audio, file, location, contact, sticker, template, carousel, quick_reply, button, rich_menu, interactive, reaction.

## Channel Dashboard

Monitor channel health via the dashboard service:

```typescript
const summary = channelDashboard.getSummary();
// {
//   totalChannels: 4,
//   connected: 3,
//   disconnected: 0,
//   errors: 1,
//   channels: [...],
//   bindings: [...]
// }

const health = channelDashboard.healthCheck();
// [{ channelId: 'telegram-main', healthy: true, status: 'connected', lastActivity: '...' }]
```

## Docker Compose Configuration

Channel-related environment variables in `docker-compose.yml`:

```yaml
unicore-api-gateway:
  environment:
    # Pro feature flag — enables all channel adapters
    ENABLE_ALL_CHANNELS: "true"
```

All channel credentials (Telegram, LINE, Facebook, Instagram, WhatsApp, Slack, Discord, Email, WebChat) are configured via the dashboard **Settings → Channels** page and stored in the database.

## Related

- [AI Agents Guide](agents.md) — Configure the agents that respond on channels
- [Workflows Guide](workflows.md) — Trigger workflows from channel events
- [Pro Edition](../editions/pro.md) — Channel features require Pro license
- [REST API](../api-reference/rest-api.md) — Webhook endpoints
