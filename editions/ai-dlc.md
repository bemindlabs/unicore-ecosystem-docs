# UniCore AI-DLC — Developer Lifecycle Chat

Updated: 2026-03-22

AI-powered development team collaboration platform with 5 specialized SDLC agents.

## Overview

| Detail | Value |
|-----------|-------|
| **License** | Business Source License 1.1 (BSL 1.1) |
| **Repository** | Private (available with license) |
| **Package** | `@unicore/ai-dlc` |
| **Ports** | WebSocket `19789`, HTTP `19790` |
| **Framework** | NestJS 10.4, TypeScript 5.5 |

UniCore AI-DLC (Developer Lifecycle Chat) provides a team of 5 AI agents that collaborate on your codebase like a real development team. Each agent specializes in a phase of the software development lifecycle — architecture, coding, testing, deployment, and project management.

## Architecture

```
unicore-ai-dlc/
├── packages/
│   └── dlc-types/              ← Shared TypeScript interfaces
└── services/
    └── dlc-gateway/            ← NestJS distributed WebSocket gateway
        ├── agents/             ← 5 SDLC agents + registry + delegation
        ├── gateway/            ← WebSocket gateway + federation
        ├── rooms/              ← 6 room types with specialized metadata
        ├── messaging/          ← Messages, DMs, threads, read receipts
        ├── rag/                ← Fusion search (personal, central, room scopes)
        ├── presence/           ← Online/away/busy/in_meeting/offline
        └── notifications/      ← Mentions, invites, agent completions
```

## The 5 SDLC Agents

### Arch (Architect)

System design, ADR generation, architecture reviews, and tech debt analysis.

| Tool | Description |
|------|-------------|
| `generate_adr` | Generate Architecture Decision Records |
| `review_architecture` | Review for scalability, security, reliability issues |
| `analyze_tech_debt` | Identify and categorize technical debt with remediation roadmap |
| `design_system` | High-level system design for new features (Mermaid, ASCII, prose) |
| `delegate_to_agent` | Delegate tasks to Developer, QA, DevOps, or PM |

### Dev (Developer)

Code generation, code review, refactoring, PR summaries, and bug diagnosis.

| Tool | Description |
|------|-------------|
| `generate_code` | Production-ready code (TypeScript, Python, SQL, Go, shell) |
| `review_code` | Code review for correctness, style, security, performance |
| `refactor_code` | Refactor for readability, performance, or architecture |
| `summarize_pr` | PR summary with risk assessment and test plan |
| `diagnose_bug` | Root-cause analysis with fix suggestions |
| `delegate_to_agent` | Delegate to other agents |

### QA (Tester)

Test strategy, test generation, coverage analysis, bug triage, and quality gates.

| Tool | Description |
|------|-------------|
| `generate_test_strategy` | Comprehensive test strategy by risk level |
| `generate_tests` | Test cases for Jest, Playwright, or SuperTest |
| `analyze_coverage` | Coverage gap identification (default threshold: 80%) |
| `triage_bug` | Classify severity (P0-P3), reproduction steps, root cause |
| `define_quality_gates` | CI/CD pass/fail criteria |
| `delegate_to_agent` | Delegate to other agents |

### Ops (DevOps)

CI/CD pipelines, deployment planning, monitoring, incident response, and log analysis.

| Tool | Description |
|------|-------------|
| `generate_pipeline` | CI/CD config (GitHub Actions, GitLab CI, Jenkins) |
| `plan_deployment` | Deployment plan with rollback and risk assessment |
| `analyze_logs` | Docker/app log analysis for errors and patterns |
| `create_incident_runbook` | Incident response runbook (SEV1-SEV4) |
| `review_infrastructure` | Review docker-compose, nginx, k8s configs |
| `delegate_to_agent` | Delegate to other agents |

### Scrum (Project Manager)

Sprint planning, user stories, standup facilitation, burndown tracking, and retrospectives.

| Tool | Description |
|------|-------------|
| `plan_sprint` | Sprint plan with capacity and story selection |
| `write_user_story` | INVEST-compliant stories with BDD acceptance criteria |
| `facilitate_standup` | Async standup summaries (Slack, Markdown, Jira) |
| `analyze_burndown` | Sprint burndown with risk assessment |
| `run_retrospective` | Retrospective facilitation (start-stop-continue, mad-sad-glad, 4Ls) |
| `delegate_to_agent` | Delegate to other agents |

## Agent Delegation

Agents can delegate tasks to each other with priority levels:

```
Arch designs system → delegates implementation to Dev
Dev writes code → delegates test writing to QA
QA finds issues → delegates fix to Dev
Dev fixes & deploys → delegates deploy verification to Ops
Ops confirms → Scrum updates sprint board
```

Priority levels: `low`, `normal`, `high`, `urgent`

## Room Types

DLC provides 6 specialized room types for team collaboration:

| Type | Purpose | Special Features |
|------|---------|------------------|
| `general` | Standard chat/discussion | Topic, max members, retention settings |
| `meeting` | Structured meetings | Agenda items, timebox, action items, participants |
| `war` | Incident response | Priority escalation (critical/high/medium/low), incident timeline, auto-invite |
| `standup` | Daily standups | Yesterday/today/blockers, speaker queue, async mode with schedule |
| `retro` | Sprint retrospectives | Columns (good/bad/action), anonymous cards, voting, action assignment |
| `dm` | Direct messages | 1-to-1 or group DMs |

## RAG Integration

Each agent has access to 3 scopes of contextual knowledge:

| Scope | Description | Example Content |
|-------|-------------|-----------------|
| **Personal** | User-specific notes and preferences | Personal coding standards, bookmarks |
| **Central** | Team/org-wide knowledge base | ADRs, coding guidelines, API docs |
| **Room** | Current room discussion context | Recent conversation, shared documents |

RAG uses fusion search across all scopes with Qdrant vector database, deduplication, and re-ranking.

### Agent-Specific Context Categories

| Agent | RAG Categories |
|-------|---------------|
| Arch | adr, architecture, design, tech-debt, standards |
| Dev | code, pr, review, conventions, patterns |
| QA | test-strategy, coverage, bugs, qa-standards |
| Ops | deployment, runbook, incident, infrastructure, monitoring |
| Scrum | sprint, backlog, story, velocity, retro |

## Messaging

- Thread-based conversations with reply chains
- Read receipts and typing indicators
- Message search across rooms
- Soft deletes with audit trail
- Reactions and file attachments
- DM rooms (1-to-1 and group)

## Event Streaming

All DLC events are published to Kafka topics:

| Category | Topics | Retention |
|----------|--------|-----------|
| Messages | `dlc.message.created`, `updated`, `deleted`, `reaction` | 3-7 days |
| DMs | `dlc.message.dm`, `dm.room.created` | 7 days |
| Presence | `dlc.presence.update`, `heartbeat`, `offline` | 1 hour - 1 day |
| Rooms | `dlc.room.created`, `joined`, `left`, `updated`, `archived`, `typing` | 1 hour - 7 days |
| Notifications | `dlc.notification.created`, `read`, `mention`, `invite`, `agent.complete` | 1 hour - 7 days |

## WebSocket Gateway

### Connection

```javascript
// Browser/Node.js
const ws = new WebSocket('ws://localhost:19789?token=<JWT>');

// Or with header
ws.setHeader('Authorization', 'Bearer <JWT>');
```

### Events

```javascript
// Send message to room
ws.send(JSON.stringify({
  type: 'room:message',
  payload: { roomId: 'room-1', content: 'Hello team!' }
}));

// Mention an agent
ws.send(JSON.stringify({
  type: 'room:message',
  payload: { roomId: 'room-1', content: '@Dev review this PR' }
}));
```

## Database Schema

DLC uses PostgreSQL with Prisma ORM. Key models:

- **Room** — id, name, type, state, members, settings
- **Message** — id, roomId, authorId, content, threadId, reactions, attachments
- **RoomMember** — userId, role (owner/admin/member)
- **MeetingMeta** — agenda, participants, timebox, action items
- **WarMeta** — priority, incident timeline, auto-invited teams
- **StandupMeta** — responses, speaker queue, blockers, async mode
- **RetroMeta** — columns, cards, votes, anonymous mode
- **UserPresence** — status, statusText, lastSeenAt
- **Notification** — type, title, body, read status
- **RagDocument** — scope, content, category, tags, confidentiality, version

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `WS_PORT` | `19789` | WebSocket gateway port |
| `HTTP_PORT` | `19790` | HTTP API port |
| `DATABASE_URL` | — | PostgreSQL connection string |
| `REDIS_URL` | — | Redis for distributed state |
| `QDRANT_URL` | `http://vectordb:6333` | Qdrant vector database |
| `JWT_SECRET` | — | JWT signing key |
| `KAFKA_BROKERS` | `localhost:9092` | Kafka broker addresses |
| `AI_ENGINE_URL` | `http://ai-engine:4200` | AI Engine for LLM calls |
| `API_GATEWAY_URL` | `http://api-gateway:4000` | API Gateway |

## Related

- [Architecture Overview](../architecture/overview.md)
- [AI Agents Guide](../guides/agents.md)
- [WebSocket API Reference](../api-reference/websocket-api.md)
