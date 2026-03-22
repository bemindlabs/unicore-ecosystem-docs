# Architecture Overview

Updated: 2026-03-22

UniCore is a modular, AI-powered business platform built on a multi-repo monorepo structure. It ships in multiple editions — from an open-core community release through enterprise clustering — all sharing the same container topology and service contracts.

## C4 Model — System Context

```mermaid
C4Context
  title UniCore System Context

  Person(user, "Business User", "Manages CRM, orders, invoices, and AI workflows via the dashboard")
  Person(admin, "Platform Admin", "Provisions licenses, manages deployments, monitors audit logs")
  Person(dev, "Developer / Integrator", "Connects external systems via REST API or webhooks")

  System(unicore, "UniCore Platform", "AI-powered ERP, CRM, inventory, invoicing, and multi-agent automation")

  System_Ext(llm, "LLM Providers", "OpenAI, Anthropic (Claude), and compatible APIs")
  System_Ext(channels, "Messaging Channels", "Telegram, LINE, and custom webhook channels")
  System_Ext(cloudflare, "Cloudflare", "SSL termination and DNS")
  System_Ext(licenseServer, "UniCore License Server", "Ed25519 license validation and machine binding")

  Rel(user, unicore, "Uses", "HTTPS")
  Rel(admin, unicore, "Administers", "HTTPS")
  Rel(dev, unicore, "Integrates", "REST/WebSocket")
  Rel(unicore, llm, "Orchestrates AI models", "HTTPS")
  Rel(unicore, channels, "Sends/receives messages", "HTTPS/webhooks")
  Rel(unicore, cloudflare, "DNS + SSL", "HTTPS")
  Rel(unicore, licenseServer, "Validates license", "HTTPS")
```

## C4 Model — Container Diagram

```mermaid
C4Container
  title UniCore Platform — Container Diagram

  Boundary(frontend, "Frontend") {
    Container(dashboard, "Dashboard", "Next.js 16", "Single-page application — CRM, inventory, invoicing, AI agents, settings")
  }

  Boundary(backend, "Backend Services") {
    Container(gateway, "API Gateway", "NestJS 10", "JWT auth, REST proxy, webhook ingestion")
    Container(erp, "ERP Service", "NestJS 10", "CRM, inventory, orders, invoicing, expenses, reports")
    Container(ai, "AI Engine", "NestJS 10", "Multi-model orchestration — OpenAI, Anthropic")
    Container(rag, "RAG Service", "NestJS 10", "Vector search, knowledge base management")
    Container(bootstrap, "Bootstrap Service", "NestJS 10", "Initial setup wizard, admin provisioning")
    Container(openclaw, "OpenClaw Gateway", "NestJS 10", "Multi-agent WebSocket hub — 9 default agents")
    Container(workflow, "Workflow Engine", "NestJS 10", "Kafka-driven workflow automation")
    Container(licenseApi, "License API", "NestJS 10", "License key validation, machine binding")
  }

  Boundary(data, "Data Stores") {
    ContainerDb(postgres, "PostgreSQL 16", "postgres:16-alpine", "Primary relational database")
    ContainerDb(redis, "Redis 7", "redis:7-alpine", "Cache, sessions, pub/sub")
    ContainerDb(qdrant, "Qdrant", "qdrant/qdrant", "Vector embeddings for RAG")
    ContainerDb(kafka, "Kafka 7.5", "cp-kafka:7.5.0", "Event streaming bus")
  }

  Boundary(proxy, "Reverse Proxy") {
    Container(nginx, "Nginx", "nginx:alpine", "Internal reverse proxy — routes /api/, /auth/, /ws, /")
  }

  Rel(dashboard, gateway, "API calls", "HTTP/JSON")
  Rel(dashboard, openclaw, "Agent chat", "WebSocket")
  Rel(gateway, erp, "Proxies ERP calls", "HTTP")
  Rel(gateway, ai, "Proxies AI calls", "HTTP")
  Rel(gateway, rag, "Proxies RAG calls", "HTTP")
  Rel(gateway, bootstrap, "Proxies wizard calls", "HTTP")
  Rel(gateway, openclaw, "Proxies agent calls", "HTTP/WS")
  Rel(erp, kafka, "Publishes domain events", "Kafka")
  Rel(workflow, kafka, "Consumes domain events", "Kafka")
  Rel(workflow, erp, "Triggers ERP actions", "HTTP")
  Rel(workflow, openclaw, "Triggers agents", "WebSocket")
  Rel(rag, qdrant, "Stores/queries vectors", "HTTP gRPC")
  Rel(gateway, postgres, "Auth, settings, tasks", "Prisma")
  Rel(erp, postgres, "ERP data (unicore_erp DB)", "Prisma")
  Rel(bootstrap, postgres, "Wizard state", "Prisma")
  Rel(gateway, redis, "Session cache", "Redis protocol")
  Rel(openclaw, redis, "Agent state", "Redis protocol")
  Rel(licenseApi, postgres, "License records", "Prisma")
```

## Multi-Repo Structure

UniCore is maintained across eight GitHub repositories, five of which are active submodules in the production deployment:

```
unicores/                          (root workspace — production compose)
├── unicore/                       BSL 1.1   — Community edition
├── unicore-pro/                   BSL 1.1   — Pro extensions
├── unicore-license/               Proprietary — License server
├── unicore-enterprise/            Proprietary — Enterprise clustering & compliance
├── unicore-geek/                  BSL 1.1   — Terminal-first TUI edition
├── unicore-ai-dlc/                BSL 1.1   — AI Developer Lifecycle Chat
├── unicore-platform/              BSL 1.1   — Public website (landing, pricing, showcases)
└── unicore-ecosystem-docs/        CC BY 4.0 — This documentation
```

| Repo | GitHub | License | Purpose |
|------|--------|---------|---------|
| `unicore` | github.com/bemindlabs/unicore | BSL 1.1 | Open-core platform: dashboard, API gateway, ERP, AI engine, RAG, OpenClaw, workflow |
| `unicore-pro` | github.com/bemindlabs/unicore-pro | BSL 1.1 | Pro packages: SSO, RBAC, AUDIT, advanced workflows, all channels, white-label, custom domains |
| `unicore-license` | github.com/bemindlabs/unicore-license | Proprietary | License server: Ed25519-signed keys, machine binding, feature flag enforcement |
| `unicore-enterprise` | github.com/bemindlabs/unicore-enterprise | Proprietary | Multi-tenancy, HA clustering, GDPR/SOC2 compliance, SAML/AD/OIDC enterprise SSO |
| `unicore-geek` | github.com/bemindlabs/unicore-geek | BSL 1.1 | Terminal-first edition: TUI dashboard, REPL console, game mode, no web UI |
| `unicore-ai-dlc` | github.com/bemindlabs/unicore-ai-dlc | BSL 1.1 | AI Developer Lifecycle Chat: distributed WebSocket gateway for developer workflows |
| `unicore-platform` | github.com/bemindlabs/unicore-platform | BSL 1.1 | Public website: landing pages, pricing, showcases (Next.js 16, port 3100) |

## Edition Model

```mermaid
graph TD
  CE[Community Edition<br/>unicore/] -->|extends| PRO[Pro Edition<br/>unicore-pro/]
  PRO -->|extends| ENT[Enterprise Edition<br/>unicore-enterprise/]
  CE -.->|alternative| GEEK[Geek Edition<br/>unicore-geek/]

  subgraph Community Features
    CE --> DASH[Dashboard]
    CE --> GW[API Gateway + Auth]
    CE --> ERP2[ERP: CRM/Inventory/Invoicing]
    CE --> AI2[AI Engine]
    CE --> RAG2[RAG / Vector Search]
    CE --> OC[OpenClaw Multi-Agent]
    CE --> WF[Workflow Engine]
  end

  subgraph Pro Features
    PRO --> SSO[SSO + RBAC]
    PRO --> WL[White-Label Branding]
    PRO --> CH[All Channels: Telegram, LINE]
    PRO --> AWF[Advanced Workflows]
    PRO --> CD[Custom Domains]
    PRO --> AUD[Audit Logs]
  end

  subgraph Enterprise Features
    ENT --> MT[Multi-Tenancy]
    ENT --> HA[HA Clustering]
    ENT --> COMP[GDPR/SOC2 Compliance]
    ENT --> EIDS[SAML/AD/OIDC SSO]
  end
```

The `UNICORE_EDITION` environment variable (`community`, `pro`, `full`) controls which feature branches are active at runtime. Pro feature flags (`ENABLE_SSO`, `ENABLE_WHITE_LABEL`, `ENABLE_ALL_CHANNELS`, etc.) are injected via Docker Compose.

## Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | Next.js + React + TypeScript | 14.2 / 18.3 / 5.5 |
| Frontend UI | Tailwind CSS + shadcn/ui | — |
| Backend | NestJS + Node.js + TypeScript | 10.4 / 20+ / 5.5 |
| ORM | Prisma (`db push`, not migrations) | 6 |
| Auth | Passport.js (JWT + local strategy) | — |
| Primary DB | PostgreSQL | 16 |
| Cache / Sessions | Redis | 7 |
| Vector DB | Qdrant | latest |
| Event Streaming | Apache Kafka + Zookeeper | 7.5 (KafkaJS 2.2) |
| AI Providers | OpenAI, Anthropic (Claude) | — |
| Build System | Turborepo + pnpm workspaces | 2.0 / 10.30+ |
| Container Runtime | Docker + Docker Compose | — |
| Reverse Proxy | Nginx | alpine |
| License Crypto | Ed25519 signing + SHA-256 fingerprinting | — |
| Testing | Jest (unit), Playwright (E2E), SuperTest | 29 / 1.58 |
| Package Namespace | `@unicore/*` (community+pro), `@unicore-license/*` | — |

## Repository Structure (unicore)

The `unicore/` repository follows a Turborepo monorepo layout:

```
unicore/
├── apps/
│   └── dashboard/             Next.js 16 frontend (port 3000)
├── services/
│   ├── api-gateway/           NestJS REST + auth (port 4000)
│   ├── erp/                   NestJS ERP: CRM, inventory, invoicing (port 4100)
│   ├── ai-engine/             NestJS AI model orchestration (port 4200)
│   ├── rag/                   NestJS vector search + Qdrant (port 4300)
│   ├── bootstrap/             NestJS wizard + initial setup (port 4500)
│   ├── openclaw-gateway/      Multi-agent WebSocket (port 18789/18790)
│   └── workflow/              Kafka-based workflow engine (port 4400)
├── packages/
│   ├── shared-types/          Shared TypeScript interfaces
│   ├── config/                Configuration management
│   ├── ui/                    shadcn/ui component library
│   └── integrations/          3rd-party API wrappers
└── nginx/
    └── default.conf           Nginx reverse proxy config
```
