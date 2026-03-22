# UniCore Community Edition

Updated: 2026-03-22

The Community Edition is the open-core foundation of the UniCore platform — free forever, self-hosted, and fully functional for small teams and individual operators.

## Overview

| Attribute | Value |
|-----------|-------|
| **License** | Business Source License 1.1 (BSL 1.1) |
| **Repository** | [github.com/bemindlabs/unicore](https://github.com/bemindlabs/unicore) |
| **Price** | Free |
| **Hosting** | Self-hosted (Docker Compose) |
| **Edition flag** | `UNICORE_EDITION: community` |

## Included Features

### Core Services

| Service | Port | Description |
|---------|------|-------------|
| Dashboard | 3000 | Next.js 16 frontend with full UI |
| API Gateway | 4000 | NestJS REST API, JWT authentication |
| ERP | 4100 | CRM, inventory, invoicing, reporting |
| AI Engine | 4200 | AI model orchestration |
| RAG | 4300 | Vector search via Qdrant |
| Bootstrap | 4500 | Setup wizard |
| OpenClaw Gateway | 18789/18790 | Multi-agent WebSocket |
| Workflow | — | Kafka-based workflow engine |

### AI Agents

The Community Edition includes **2 built-in agents** via OpenClaw:

- **Assistant Agent** — General-purpose conversational AI
- **ERP Agent** — CRM, inventory, and invoicing assistant

### Roles & Permissions

Community Edition supports **2 roles**:

| Role | Access |
|------|--------|
| `OWNER` | Full access to all settings, billing, and users |
| `OPERATOR` | Day-to-day operations, no billing or user management |

### Messaging Channels

Community Edition includes **3 channels**:

- Web Chat (embedded widget)
- REST API channel (custom integrations)
- Email (SMTP)

### RAG / Knowledge Base

- **Storage limit**: 1 GB vector storage in Qdrant
- Supports PDF, DOCX, TXT, and CSV uploads
- Manual re-indexing via dashboard

### Workflow Engine

- Kafka-based event-driven workflows
- Trigger types: manual, webhook, scheduled (cron)
- Up to **10 active workflows**

## Limitations

| Feature | Community | Pro | Enterprise |
|---------|-----------|-----|------------|
| AI Agents | 2 | 7 | Unlimited |
| Roles | 2 | 5 | Custom |
| Channels | 3 | 21+ | 21+ |
| RAG storage | 1 GB | Unlimited | Unlimited |
| Active workflows | 10 | Unlimited | Unlimited |
| SSO | — | OIDC/SAML | SAML/AD/OIDC |
| White-label | — | Yes | Yes |
| Multi-tenancy | — | — | Yes |
| SLA support | — | — | Yes |
| Audit logging | Basic | Full | Full + SIEM |

## Infrastructure

### Databases

| Database | Purpose |
|----------|---------|
| PostgreSQL 16 | Auth, sessions, settings, ERP data |
| Redis 7 | Cache, session store |
| Qdrant | Vector embeddings (RAG) |
| Kafka + Zookeeper | Workflow event streaming |

### Minimum Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk | 20 GB | 50 GB |
| OS | Linux (Ubuntu 22.04+) | Ubuntu 22.04 LTS |
| Docker | 24+ | Latest stable |

## Getting Started

```bash
# Clone the community repo
git clone https://github.com/bemindlabs/unicore.git
cd unicore

# Copy environment config
cp .env.example .env

# Start all services
docker compose --profile apps up -d

# Push database schemas
# Container names are prefixed by your clone directory name (e.g. unicore-unicore-api-gateway-1)
docker exec <workspace>-unicore-api-gateway-1 npx prisma db push --accept-data-loss
docker exec <workspace>-unicore-erp-1 npx prisma db push --accept-data-loss

# Provision admin user
curl -s http://localhost:4000/auth/provision-admin -X POST \
  -H 'Content-Type: application/json' \
  -H 'X-Bootstrap-Secret: <your-bootstrap-secret>' \
  -d '{"email":"<your-admin-email>","password":"<your-admin-password>","name":"Admin"}'
```

Visit `http://localhost:3000` to access the dashboard.

## Upgrading to Pro

To unlock all agents, roles, channels, and advanced features, see the [Pro Edition guide](./pro.md).

A valid license key (`UC-XXXX-XXXX-XXXX-XXXX`) is required. Purchase licenses at [unicore.bemind.tech](https://unicore.bemind.tech).

---

© 2026 BeMind Technology — Licensed under BSL 1.1
