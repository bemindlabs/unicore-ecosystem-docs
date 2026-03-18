# UniCore Ecosystem Documentation

Official public documentation, wiki, and guides for the UniCore AI-powered business platform ecosystem by BeMind Technology.

## Overview

This repository contains all public-facing documentation for the UniCore ecosystem — a modular, AI-powered business platform available in multiple editions tailored to different needs.

## Ecosystem Editions

| Edition | Repository | Description |
|---------|-----------|-------------|
| **Community** | [unicore](https://github.com/bemindlabs/unicore) | Open-core edition with dashboard, API gateway, ERP, AI engine, RAG, and workflow engine |
| **Pro** | [unicore-pro](https://github.com/bemindlabs/unicore-pro) | Extended features: SSO, RBAC, white-label, advanced workflows, all messaging channels |
| **Enterprise** | unicore-enterprise | Multi-tenancy, compliance (GDPR/SOC2), HA clustering, enterprise SSO (SAML/AD/OIDC) |
| **Geek** | unicore-geek | Terminal-first edition with game mode, TUI, REPL console — no dashboard |
| **License Server** | unicore-license | License validation, key management, and machine binding |
| **AI DLC** | unicore-ai-dlc | AI Developer Lifecycle Chat — distributed WebSocket gateway for developer workflows |
| **Platform** | unicore-platform | Public website — landing pages, pricing, showcases (Next.js 14) |

## Documentation Structure

```
unicore-ecosystem-docs/
├── README.md                     ← You are here
├── getting-started/
│   ├── installation.md           ← Quick start for all editions
│   ├── system-requirements.md    ← Hardware & software prerequisites
│   ├── configuration.md          ← Environment variables & config
│   └── first-steps.md            ← Post-install walkthrough
├── architecture/
│   ├── overview.md               ← High-level system design
│   ├── services.md               ← 19-container service map
│   ├── networking.md             ← Routing, proxies, ports
│   ├── databases.md              ← PostgreSQL, Redis, Qdrant schemas
│   └── messaging.md              ← Kafka topics & event flows
├── guides/
│   ├── dashboard/                ← Dashboard UI guides
│   ├── api/                      ← REST API reference & examples
│   ├── erp/                      ← CRM, inventory, invoicing guides
│   ├── ai-engine/                ← AI model configuration & usage
│   ├── rag/                      ← Vector search & knowledge base setup
│   ├── workflows/                ← Workflow builder & automation
│   ├── agents/                   ← OpenClaw multi-agent setup
│   └── channels/                 ← Messaging channel integrations
├── editions/
│   ├── community.md              ← Community edition features & limits
│   ├── pro.md                    ← Pro edition features & upgrade path
│   ├── enterprise.md             ← Enterprise features & deployment
│   └── geek.md                   ← Geek edition CLI/TUI reference
├── deployment/
│   ├── docker.md                 ← Docker Compose deployment
│   ├── kubernetes.md             ← K8s manifests & Helm charts
│   ├── cloud/                    ← AWS, GCP, Azure guides
│   └── on-premise.md             ← Self-hosted deployment
├── development/
│   ├── contributing.md           ← Contribution guidelines
│   ├── local-setup.md            ← Developer environment setup
│   ├── coding-standards.md       ← TypeScript conventions & linting
│   ├── testing.md                ← Jest, Playwright, E2E testing
│   └── plugin-development.md     ← Building custom plugins
├── api-reference/
│   ├── rest-api.md               ← Full REST API documentation
│   ├── websocket-api.md          ← WebSocket (OpenClaw) API
│   ├── webhook-api.md            ← Webhook event reference
│   └── sdk.md                    ← Client SDK documentation
├── security/
│   ├── authentication.md         ← JWT, SSO, OAuth flows
│   ├── authorization.md          ← RBAC & permission model
│   ├── data-privacy.md           ← GDPR, data handling policies
│   └── best-practices.md         ← Security hardening guide
├── changelog/
│   └── CHANGELOG.md              ← Release notes & version history
├── faq.md                        ← Frequently asked questions
└── LICENSE                       ← Documentation license (CC BY 4.0)
```

## Tech Stack

The UniCore platform is built with:

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14, React 18, TypeScript 5.5, Tailwind CSS, shadcn/ui |
| Backend | NestJS 10.4, Node.js 20+, TypeScript 5.5 |
| ORM | Prisma 6 |
| Auth | Passport.js (JWT + local strategy) |
| Databases | PostgreSQL 16, Redis 7, Qdrant (vectors) |
| Streaming | Kafka 7.5 (KafkaJS) |
| AI | Multi-model orchestration, RAG with vector search |
| Build | Turborepo 2.0, pnpm workspaces |

## Quick Links

- **Website**: [unicore.bemind.tech](https://unicore.bemind.tech)
- **Dashboard**: [unicore-dashboard.bemind.tech](https://unicore-dashboard.bemind.tech)
- **Main Repo**: [unicore](https://github.com/bemindlabs/unicore)
- **Issues**: [GitHub Issues](https://github.com/bemindlabs/unicore-ecosystem-docs/issues)
- **Wiki**: [GitHub Wiki](https://github.com/bemindlabs/unicore-ecosystem-docs/wiki)

## Contributing

We welcome contributions to the documentation! See [development/contributing.md](development/contributing.md) for guidelines.

## License

Documentation is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

Platform source code is licensed under Business Source License 1.1 (BSL 1.1).

© 2026 BeMind Technology
