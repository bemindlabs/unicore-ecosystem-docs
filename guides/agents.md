# AI Agents Guide

Updated: 2026-03-22

Configure, customize, and deploy UniCore's AI agent system.

## Overview

UniCore uses a **Router + Specialist** architecture. When a user sends a message, the Router Agent classifies the intent and delegates to the appropriate specialist agent. Each specialist has domain-specific tools it can invoke.

```
User Message → Router Agent → Intent Classification → Specialist Agent → Tool Execution → Response
```

## Built-In Agents (7 Specialists)

UniCore ships with 7 specialist agents. Community edition includes 2 agents; Pro unlocks all 7.

| Agent | Type | Description | Edition |
|-------|------|-------------|---------|
| **Comms** | `comms` | Email drafting, inbox triage, social media scheduling | Community |
| **Finance** | `finance` | Transaction analysis, financial reports, cash flow forecasting | Community |
| **Growth** | `growth` | Funnel analysis, ad campaigns, audience segmentation | Pro |
| **Ops** | `ops` | Task management, scheduling, standup summaries | Pro |
| **Research** | `research` | Market intel, competitor tracking, web search | Pro |
| **ERP** | `erp` | Natural-language CRM/order/inventory queries | Pro |
| **Builder** | `builder` | Code generation, workflow scaffolding, API scaffolding | Pro |

## Agent Tools

Each agent has 7 specialized tools. Here is a summary:

### Comms Agent
- `draft_email` — Compose email with tone and intent
- `send_email` — Send a drafted email
- `list_inbox` — Fetch unread emails
- `reply_email` — Generate contextual reply to a thread
- `schedule_post` — Schedule social media post (Facebook, Instagram, Twitter, LinkedIn, LINE)
- `fetch_social_feed` — Retrieve mentions and engagement
- `moderate_comment` — Classify and action social comments

### Finance Agent
- `categorize_transaction` — Assign spending category with confidence score
- `list_transactions` — Query transactions by date, category, amount
- `generate_report` — P&L, cash flow, expenses, or revenue report (JSON/CSV/PDF)
- `forecast_cashflow` — Project future cash position
- `detect_anomalies` — Scan for unusual spending patterns
- `create_invoice` — Generate and persist an invoice
- `reconcile_accounts` — Match bank transactions against ERP

### Growth Agent
- `analyse_funnel` — Conversion rates per funnel stage
- `identify_drop_offs` — Top N stages with highest lead loss
- `get_ad_performance` — Campaign metrics (Google Ads, Meta Ads, TikTok Ads, LINE Ads)
- `adjust_ad_budget` — Reallocate campaign spend
- `create_campaign` — Scaffold new ad campaign
- `generate_utm` — Generate UTM-tagged URLs
- `segment_audience` — Define audience segments for ad targeting

### Ops Agent
- `create_task` — Create task with priority and due date
- `update_task` — Update task status, assignee, priority
- `list_tasks` — Query tasks by status, assignee, due date
- `schedule_event` — Book calendar events with attendees
- `check_availability` — Query free/busy slots
- `set_reminder` — Configure reminders (in-app, email, Slack, LINE)
- `generate_standup` — Daily standup summary for team

### Research Agent
- `search_web` — Web search with date and language filters
- `analyse_competitor` — Competitor data (pricing, features, reviews, news, SEO)
- `monitor_keywords` — Track brand mentions across web, Twitter, Reddit, news
- `summarise_document` — Extract key points from URL or document
- `generate_market_brief` — Compile market research report
- `track_trends` — Identify trending topics by niche and geography
- `benchmark_pricing` — Compare pricing against competitors

### ERP Agent
- `nl_query` — Natural language queries against any ERP module
- `create_record` — Create CRM contacts, orders, products, etc.
- `update_record` — Modify existing records with audit trail
- `delete_record` — Soft-delete records
- `bulk_update` — Batch update filtered records
- `export_data` — Export data as CSV, JSON, or PDF
- `generate_summary` — Natural language summary of module health

### Builder Agent
- `generate_code` — Generate code (TypeScript, Python, SQL, shell)
- `create_workflow` — Scaffold automation workflows
- `scaffold_api` — Generate NestJS CRUD module
- `write_tests` — Generate Jest unit/integration/E2E tests
- `review_code` — Analyze for bugs, security, performance issues
- `create_webhook` — Register inbound webhook endpoint
- `deploy_function` — Prepare serverless function deployment

## How Routing Works

### Intent Classification

The Router Agent uses an LLM call with low temperature (0.1) to classify user intent:

```json
{
  "intent": "finance",
  "confidence": 0.92,
  "reasoning": "User asked about monthly expenses",
  "alternates": [{ "intent": "erp", "confidence": 0.45 }]
}
```

**Confidence threshold:** `0.35` — below this, the intent is marked `unknown` and the user is asked to clarify.

### Delegation Flow

1. Router classifies intent → selects target agent type
2. Checks if agent is registered and available
3. If available: delegates to specialist, returns response
4. If unavailable: returns fallback response suggesting available agents

## Configuration

### Business Context

Agents are configured via business context passed through the session:

```json
{
  "businessName": "Acme Corp",
  "currency": "USD",
  "timezone": "Asia/Bangkok",
  "industry": "E-commerce",
  "commsDefaultTone": "professional and friendly",
  "erpModules": ["contacts", "orders", "invoices", "inventory"]
}
```

This context is injected into each agent's system prompt at runtime.

### Configuring via Dashboard

1. Go to **Settings > AI Configuration** in the dashboard
2. Connect your LLM providers (API keys encrypted with AES-256-GCM)
3. Go to **Settings > Agents** to configure:
   - Which agents are enabled
   - Channel bindings (which channels each agent responds on)
   - Autonomy level per agent
   - Working hours per agent

### LLM Provider Failover

Agents use the AI Engine service which supports 13+ LLM providers with automatic failover:

```
openai → anthropic → openrouter → deepseek → groq → ... → ollama
```

## Custom Agents (Pro)

UniCore Pro includes the **Agent Builder** — a 6-step wizard for creating custom agents:

### Step 1: Basic Info
Set agent name, description, and tags.

### Step 2: System Prompt
Write the agent's system prompt with variable injection (`{{businessName}}`, `{{currency}}`).

### Step 3: Tool Selection
Choose tools from the tool registry to attach to the agent.

### Step 4: Model Configuration
Select LLM provider and configure temperature, max tokens, top-p, and stop sequences.

### Step 5: Testing
Run simulated conversations to measure response quality, token usage, and latency.

### Step 6: Deployment
Deploy to sandbox or production. Agents are versioned — you can roll back to any previous version.

### Agent Lifecycle

Custom agents follow this lifecycle: `draft` → `active` | `testing` | `archived`

## WebSocket Integration

### Connecting

```javascript
const ws = new WebSocket('ws://localhost:18789');

// Authenticate
ws.send(JSON.stringify({
  type: 'agent:register',
  payload: {
    agentId: 'my-agent',
    name: 'My Custom Agent',
    agentType: 'custom',
    capabilities: [{ name: 'my_tool', version: '1.0.0' }]
  }
}));
```

### Message Types

| Event | Direction | Description |
|-------|-----------|-------------|
| `agent:register` | Client → Server | Register agent with capabilities |
| `agent:heartbeat` | Client → Server | Keep agent alive (TTL-based eviction) |
| `agent:state` | Client → Server | Update state (running/idle) |
| `message:direct` | Bidirectional | Send to specific agent |
| `message:broadcast` | Client → Server | Broadcast to all agents |
| `message:publish` | Client → Server | Publish to named channel |
| `message:subscribe` | Client → Server | Subscribe to named channel |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HTTP_PORT` | `18790` | HTTP API port |
| `AI_ENGINE_URL` | `http://ai-engine:4200` | LLM provider service |
| `ERP_SERVICE_URL` | `http://erp:4100` | ERP data access |
| `RAG_SERVICE_URL` | `http://rag:4300` | Knowledge base search |
| `JWT_SECRET` | — | Authentication |

## Related

- [Channels Guide](channels.md) — Connect agents to messaging channels
- [Workflows Guide](workflows.md) — Trigger agents from workflow actions
- [AI-DLC Edition](../editions/ai-dlc.md) — Developer-focused SDLC agents
- [WebSocket API](../api-reference/websocket-api.md)
