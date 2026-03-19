# Webhook API Reference

UniCore accepts incoming webhook payloads from third-party messaging platforms. Webhooks are received at the API Gateway and routed to the OpenClaw agent pipeline for processing.

## Webhook Endpoints

| Platform | Endpoint | Method |
|----------|----------|--------|
| LINE Messaging API | `POST /webhooks/line` | `POST` |
| Telegram Bot API | `POST /webhooks/telegram` | `POST` |

Both endpoints are **publicly accessible** (no Bearer token required) because they are called by external platforms. Authentication is performed via platform-specific signature/secret header validation instead.

---

## LINE Webhook

**Endpoint**: `POST /webhooks/line`

LINE sends webhook events to this URL whenever a user interacts with the connected LINE Official Account.

### Signature Verification

Every request from LINE includes an `X-Line-Signature` header containing an HMAC-SHA256 digest of the raw request body, base64-encoded, signed with your LINE Channel Secret.

UniCore validates this signature automatically when the LINE Channel Secret is configured via dashboard Settings. Requests with a missing or invalid signature return `403 Forbidden`.

**Validation algorithm**:

```
HMAC-SHA256(body_bytes, channel_secret)
→ base64-encode
→ compare (timing-safe) with X-Line-Signature header value
```

```typescript
import { createHmac, timingSafeEqual } from 'node:crypto';

function validateLineSignature(
  bodyString: string,
  signature: string,
  channelSecret: string,
): boolean {
  const digest = createHmac('SHA256', channelSecret)
    .update(bodyString)
    .digest('base64');

  const sigBuffer = Buffer.from(signature, 'base64');
  const digestBuffer = Buffer.from(digest, 'base64');
  if (sigBuffer.length !== digestBuffer.length) return false;
  return timingSafeEqual(sigBuffer, digestBuffer);
}
```

### Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes | `application/json` |
| `X-Line-Signature` | Yes (if LINE Channel Secret is configured) | HMAC-SHA256 signature |

### Request Payload

```json
{
  "destination": "Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "events": [
    {
      "type": "message",
      "timestamp": 1705312800000,
      "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
      "source": {
        "type": "user",
        "userId": "Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      },
      "message": {
        "id": "444573844083572737",
        "type": "text",
        "text": "Hello, I need help with my order"
      }
    }
  ]
}
```

### Event Types

| `event.type` | Description |
|--------------|-------------|
| `message` | User sent a message (text, image, audio, video, file, sticker, location) |
| `follow` | User followed or unblocked the account |
| `unfollow` | User unfollowed or blocked the account |
| `join` | Bot was added to a group or room |
| `leave` | Bot was removed from a group or room |
| `postback` | User tapped a postback action |
| `beacon` | User entered the beacon's reception range |

### Message Types

| `event.message.type` | Description |
|----------------------|-------------|
| `text` | Plain text message |
| `image` | Image file |
| `video` | Video file |
| `audio` | Audio file |
| `file` | Generic file attachment |
| `location` | GPS coordinates |
| `sticker` | Sticker |

### Response

UniCore returns `200 OK` immediately, as required by LINE (response must arrive within 1 second).

```json
{ "ok": true }
```

### Webhook URL Verification

LINE verifies the webhook URL during setup by sending a request with an empty `events` array. UniCore handles this automatically — it logs the verification and returns `200 OK`.

### Configuration

LINE Channel Secret and Channel Access Token are configured via the dashboard **Settings → Channels** page.

---

## Telegram Webhook

**Endpoint**: `POST /webhooks/telegram`

Telegram sends Update objects to this URL when the bot receives messages or other events.

### Secret Token Verification

Telegram supports an optional `X-Telegram-Bot-Api-Secret-Token` header. When a Telegram webhook secret is configured via dashboard Settings, UniCore validates this header on every incoming request. Requests with an incorrect token return `403 Forbidden`.

Set the secret token when registering the webhook URL with Telegram:

```
https://api.telegram.org/bot<token>/setWebhook
  ?url=https://unicore.bemind.tech/webhooks/telegram
  &secret_token=<your-webhook-secret>
```

### Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes | `application/json` |
| `X-Telegram-Bot-Api-Secret-Token` | Conditional | Required if a Telegram webhook secret is configured via dashboard Settings |

### Request Payload

Telegram sends a single `Update` object per request.

```json
{
  "update_id": 10000001,
  "message": {
    "message_id": 1365,
    "from": {
      "id": 1234567890,
      "is_bot": false,
      "first_name": "Jane",
      "last_name": "Doe",
      "username": "janedoe"
    },
    "chat": {
      "id": 1234567890,
      "type": "private",
      "first_name": "Jane",
      "last_name": "Doe",
      "username": "janedoe"
    },
    "date": 1705312800,
    "text": "I need to check my invoice status"
  }
}
```

### Update Types

| Field | Description |
|-------|-------------|
| `message` | New message (text, photo, document, etc.) |
| `edited_message` | An edited message |
| `channel_post` | New post in a channel |
| `inline_query` | Inline mode query |
| `callback_query` | Callback from an inline keyboard button |
| `my_chat_member` | Bot's own status changed in a chat |

UniCore currently processes `message` updates. Other update types are accepted (returning `200 OK`) but not yet forwarded.

### Response

UniCore returns `200 OK` for all valid requests. Telegram retries failed requests (non-2xx responses) up to 3 times with exponential backoff.

```json
{ "ok": true }
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | Telegram Bot API token |
| `TELEGRAM_WEBHOOK_SECRET` | Optional secret token for request validation |

---

## Retry Policy

### LINE

LINE does not retry webhook deliveries. If your server is unavailable, the event is lost. Ensure the endpoint responds within 1 second (or return 200 immediately and process asynchronously).

### Telegram

Telegram retries delivery if the server returns a non-2xx status code or does not respond within 60 seconds. Retries occur at increasing intervals up to approximately 24 hours. Each `update_id` is unique and monotonically increasing — use it to detect duplicates if necessary.

---

## Registering Webhook URLs

### LINE

1. Go to the [LINE Developers Console](https://developers.line.biz/console/).
2. Select your channel → Messaging API settings.
3. Set **Webhook URL** to `https://unicore.bemind.tech/webhooks/line`.
4. Enable **Use webhook**.
5. Use **Verify** to confirm delivery (sends an empty events array).

### Telegram

```bash
curl -X POST "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://unicore.bemind.tech/webhooks/telegram",
    "secret_token": "<TELEGRAM_WEBHOOK_SECRET>",
    "allowed_updates": ["message", "callback_query", "inline_query"]
  }'
```

Verify the webhook is set:

```bash
curl "https://api.telegram.org/bot<BOT_TOKEN>/getWebhookInfo"
```

---

## Processing Pipeline

Incoming webhook events are currently logged and will be forwarded to the OpenClaw agent pipeline in a future release. The intended flow is:

```
LINE / Telegram platform
  → POST /webhooks/line | /webhooks/telegram
    → Signature/secret validation
      → Log event
        → Forward to OpenClaw (message:publish to channel)
          → Intent classification (RouterModule)
            → Delegate to appropriate agent
```

---

## Error Responses

| Status | Condition |
|--------|-----------|
| `200 OK` | Event accepted (even if payload is empty or unrecognised) |
| `403 Forbidden` | Missing or invalid signature/secret token |
| `500 Internal Server Error` | Unhandled processing error |
