# Workflow Automation Guide

Automate business processes with event-driven workflows powered by Kafka.

## Overview

UniCore's Workflow Engine listens for business events (orders, invoices, inventory changes) via Kafka and executes automated action sequences — invoking AI agents, updating ERP records, and sending notifications.

```
Kafka Event → Trigger Match → Condition Check → Action Sequence → Result
```

## Architecture

The Workflow Service runs on port `4400` and connects to:
- **Kafka** for event consumption
- **OpenClaw Gateway** for AI agent invocation
- **ERP Service** for data mutations
- **External APIs** for notifications (Telegram, LINE, email, Slack, webhooks)

## Event Topics

The engine consumes 8 Kafka event topics:

| Topic | Event | Key Payload Fields |
|-------|-------|-------------------|
| `order.created` | New order placed | orderId, customerId, customerEmail, lineItems, total, currency |
| `order.updated` | Order status changed | orderId, previousStatus, newStatus, updatedFields |
| `order.fulfilled` | Order shipped | orderId, trackingNumber, carrier, fulfilledAt |
| `invoice.created` | Invoice generated | invoiceId, orderId, amount, tax, total, dueDate |
| `invoice.overdue` | Payment past due | invoiceId, total, dueDate, daysOverdue, customerEmail |
| `invoice.paid` | Payment received | invoiceId, amountPaid, paymentMethod, transactionId |
| `inventory.low` | Stock below threshold | productId, productName, sku, currentQuantity, threshold |
| `inventory.restocked` | Inventory replenished | productId, quantityAdded, newQuantity, purchaseOrderId |

All events are wrapped in an envelope:
```json
{
  "eventId": "uuid",
  "occurredAt": "2026-03-18T12:00:00Z",
  "type": "order.created",
  "source": "erp-service",
  "schemaVersion": 1,
  "payload": { ... }
}
```

## Workflow Definition

A workflow definition specifies when to fire (trigger) and what to do (actions):

```json
{
  "id": "wf-order-to-invoice",
  "name": "Order to Invoice",
  "enabled": true,
  "schemaVersion": 1,
  "trigger": {
    "type": "erp.order.created",
    "conditions": [
      { "field": "payload.total", "operator": "gte", "value": 100 }
    ]
  },
  "actions": [
    {
      "id": "generate-invoice",
      "type": "call_agent",
      "label": "Generate Invoice",
      "config": {
        "agentName": "finance-agent",
        "promptTemplate": "Generate invoice for order {{payload.orderId}} totaling {{payload.total}} {{payload.currency}}"
      }
    },
    {
      "id": "send-confirmation",
      "type": "send_notification",
      "label": "Send Confirmation Email",
      "dependsOn": ["generate-invoice"],
      "config": {
        "channel": "email",
        "recipient": "{{payload.customerEmail}}",
        "subject": "Order {{payload.orderId}} Confirmed",
        "bodyTemplate": "Your invoice has been generated. Total: {{payload.total}} {{payload.currency}}"
      }
    }
  ]
}
```

## Trigger Types

| Type | Description | Extra Config |
|------|-------------|-------------|
| `erp.order.created` | Fires on new orders | `conditions[]` |
| `erp.order.updated` | Fires on order status changes | `conditions[]` |
| `erp.order.fulfilled` | Fires when order is shipped | `conditions[]` |
| `erp.inventory.low` | Fires when stock drops below threshold | `conditions[]` |
| `erp.inventory.restocked` | Fires on inventory replenishment | `conditions[]` |
| `erp.invoice.created` | Fires on invoice generation | `conditions[]` |
| `erp.invoice.overdue` | Fires when invoice is past due | `conditions[]` |
| `erp.invoice.paid` | Fires on payment received | `conditions[]` |
| `schedule.cron` | Time-based trigger | `cron: "0 9 * * *"` |
| `webhook` | HTTP POST trigger | `webhookSecret` |
| `manual` | API trigger via REST | — |

### Trigger Conditions

All conditions must match (AND logic) for the workflow to fire:

```json
{
  "conditions": [
    { "field": "payload.total", "operator": "gt", "value": 1000 },
    { "field": "payload.currency", "operator": "eq", "value": "USD" }
  ]
}
```

**Operators:** `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`, `not_contains`, `exists`, `not_exists`

## Action Types

### call_agent — Invoke AI Agent

```json
{
  "type": "call_agent",
  "config": {
    "agentName": "finance-agent",
    "promptTemplate": "Analyze order {{payload.orderId}}",
    "model": "gpt-4o"
  }
}
```

Calls `POST {OPENCLAW_GATEWAY_URL}/api/agents/{agentName}/execute`.

### update_erp — Mutate ERP Data

```json
{
  "type": "update_erp",
  "config": {
    "entity": "order",
    "entityId": "{{payload.orderId}}",
    "fields": { "status": "invoiced" }
  }
}
```

Calls `PATCH {ERP_SERVICE_URL}/api/{entity}/{entityId}`.

### send_notification — Email/Slack/LINE/Webhook

```json
{
  "type": "send_notification",
  "config": {
    "channel": "email",
    "recipient": "{{payload.customerEmail}}",
    "subject": "Invoice Overdue",
    "bodyTemplate": "Invoice {{payload.invoiceId}} is {{payload.daysOverdue}} days overdue."
  }
}
```

Channels: `email`, `slack`, `line`, `webhook`. Webhook sends a direct POST to the recipient URL.

### send_telegram — Telegram Message

```json
{
  "type": "send_telegram",
  "config": {
    "chatId": "{{payload.telegramChatId}}",
    "message": "Order {{payload.orderId}} has been shipped!",
    "parseMode": "HTML"
  }
}
```

### send_line — LINE Push Message

```json
{
  "type": "send_line",
  "config": {
    "to": "{{payload.lineUserId}}",
    "message": "Your invoice is ready: {{payload.total}} {{payload.currency}}"
  }
}
```

## Template Interpolation

All action config fields support `{{variable}}` tokens using dot-notation:

- `{{payload.orderId}}` — Value from the trigger event payload
- `{{outputs.generate-invoice.result}}` — Output from a previous action
- Nested paths: `{{payload.lineItems[0].name}}`

## Action Dependencies

Actions execute in topological order based on `dependsOn`:

```json
{
  "actions": [
    { "id": "step-1", "type": "call_agent", "dependsOn": [] },
    { "id": "step-2", "type": "update_erp", "dependsOn": ["step-1"] },
    { "id": "step-3", "type": "send_notification", "dependsOn": ["step-1"] }
  ]
}
```

In this example, `step-2` and `step-3` run after `step-1` completes (potentially in parallel). Circular dependencies are detected and rejected.

Each action supports:
- `continueOnError: true` — Skip failures and continue the workflow
- `timeoutMs: 30000` — Action timeout (default 30 seconds)

## Pre-Built Templates

UniCore ships with 3 workflow templates:

### Order to Invoice
**Trigger:** `erp.order.created`
1. Finance Agent generates invoice
2. Email confirmation sent to customer

### Invoice Overdue Reminder
**Trigger:** `erp.invoice.overdue`
1. Email reminder sent to customer
2. Finance Agent flags for AR review

### Low Stock Reorder
**Trigger:** `erp.inventory.low`
1. Ops Agent creates reorder request
2. Slack notification to ops team

## REST API

All endpoints are proxied via `POST /api/proxy/workflow/*`.

### Manage Definitions

```bash
# Register a workflow
curl -X POST http://localhost:4000/api/proxy/workflow/definitions \
  -H "Content-Type: application/json" \
  -d @workflow-definition.json

# List all workflows
curl http://localhost:4000/api/proxy/workflow/definitions

# Delete a workflow
curl -X DELETE http://localhost:4000/api/proxy/workflow/definitions/{id}
```

### Trigger Manually

```bash
curl -X POST http://localhost:4000/api/proxy/workflow/trigger \
  -H "Content-Type: application/json" \
  -d '{ "workflowId": "wf-order-to-invoice", "payload": { "orderId": "ORD-123" } }'
```

### Check Execution Status

```bash
# List all instances
curl http://localhost:4000/api/proxy/workflow/instances

# Get specific instance
curl http://localhost:4000/api/proxy/workflow/instances/{instanceId}
```

### Workflow Instance Status

```json
{
  "instanceId": "uuid",
  "workflowId": "wf-order-to-invoice",
  "workflowName": "Order to Invoice",
  "status": "completed",
  "actions": [
    { "actionId": "generate-invoice", "status": "completed", "output": { ... } },
    { "actionId": "send-confirmation", "status": "completed", "output": { ... } }
  ],
  "createdAt": "2026-03-18T12:00:00Z",
  "completedAt": "2026-03-18T12:00:03Z"
}
```

Status values: `pending`, `running`, `completed`, `failed`

## Advanced Workflows (Pro)

UniCore Pro adds a graph-based DSL with advanced execution features:

### Branching

```json
{
  "type": "condition",
  "conditionConfig": {
    "branches": [
      {
        "label": "High Value (> $10k)",
        "condition": { "type": "leaf", "field": "$.total", "operator": "gt", "value": 10000 },
        "nextNodeId": "escalate"
      }
    ],
    "defaultNextNodeId": "standard-process"
  }
}
```

### Parallel Execution

```json
{
  "type": "parallel",
  "parallelConfig": {
    "waitStrategy": "all",
    "maxConcurrency": 3
  },
  "branches": [
    { "id": "notify-sales", "entryNodeId": "email-sales" },
    { "id": "notify-ops", "entryNodeId": "slack-ops" }
  ]
}
```

Wait strategies: `all` (wait for every branch), `any` (first branch wins), `n_of_m` (N of M branches).

### Loops

```json
{
  "type": "loop",
  "loopConfig": {
    "loopType": "for_each",
    "collection": "$.payload.lineItems",
    "itemVariable": "item",
    "maxIterations": 100,
    "concurrency": 5
  },
  "bodyNodeId": "process-item"
}
```

Loop types: `for_each`, `while`, `count`.

### Additional Pro Features
- **Retry policies** with exponential backoff
- **Data transformation** via JSONata expressions
- **Workflow variables** with typed input/output schemas
- **Dry-run mode** for validation without side effects
- **Pause/resume/cancel** execution control
- **6 advanced templates:** lead nurture, order fulfillment, support escalation, onboarding, reporting, churn prevention

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `4400` | HTTP port |
| `KAFKA_BROKERS` | `localhost:9092` | Kafka broker addresses |
| `OPENCLAW_GATEWAY_URL` | `http://openclaw:18790` | AI agent gateway |
| `ERP_SERVICE_URL` | `http://erp:4100` | ERP service |

## Related

- [AI Agents Guide](agents.md) — Agents invoked by `call_agent` actions
- [Channels Guide](channels.md) — Channel adapters for notifications
- [Architecture: Messaging](../architecture/messaging.md) — Kafka topic details
- [REST API](../api-reference/rest-api.md)
