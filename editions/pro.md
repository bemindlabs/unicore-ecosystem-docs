# UniCore Pro Edition

The Pro Edition extends the Community Edition with advanced agent capabilities, all messaging channels, role-based access control, SSO, white-label branding, and unlimited RAG storage. Activation requires a valid license key.

## Overview

| Attribute | Value |
|-----------|-------|
| **License** | Business Source License 1.1 (BSL 1.1) |
| **Repository** | [github.com/bemindlabs/unicore-pro](https://github.com/bemindlabs/unicore-pro) |
| **Price** | Paid (license key required) |
| **Hosting** | Self-hosted (Docker Compose) |
| **Edition flag** | `UNICORE_EDITION: full` |

## License Key Activation

Pro Edition requires a license key in the format `UC-XXXX-XXXX-XXXX-XXXX`, issued by the UniCore License Server.

### Activation Steps

1. Purchase a license at [unicore.bemind.tech](https://unicore.bemind.tech)
2. Set the license key in your environment:

```bash
# In your .env file or docker-compose override
LICENSE_KEY=UC-XXXX-XXXX-XXXX-XXXX
LICENSE_SERVER_URL=https://license.unicore.bemind.tech
```

3. The license client (`@unicore-pro/license-client`) validates the key on startup and periodically via the License API (port 4600).
4. Features are gated at the API Gateway level — unlicensed feature calls return `402 Payment Required`.

### License Binding

Licenses are machine-bound using a SHA-256 fingerprint of the host hardware. A single license supports one active machine binding by default. Contact support for multi-instance or floating licenses.

## Pro Packages

The `unicore-pro` repository adds the following packages on top of the Community Edition:

| Package | Path | Description |
|---------|------|-------------|
| `@unicore-pro/agent-builder` | `packages/agent-builder/` | Visual agent builder UI |
| `@unicore-pro/agents-pro` | `packages/agents-pro/` | Extended agent library |
| `@unicore-pro/audit` | `packages/audit/` | Full audit logging with Prisma schema |
| `@unicore-pro/branding` | `packages/branding/` | White-label theming engine |
| `@unicore-pro/channels` | `packages/channels/` | All messaging channel integrations |
| `@unicore-pro/domains` | `packages/domains/` | Custom domain routing |
| `@unicore-pro/license-client` | `packages/license-client/` | License validation SDK |
| `@unicore-pro/rbac` | `packages/rbac/` | Role-based access control |
| `@unicore-pro/sso` | `packages/sso/` | Single sign-on (OIDC/SAML) |
| `@unicore-pro/workflows-advanced` | `packages/workflows-advanced/` | Advanced workflow builder |

## AI Agents

Pro Edition includes **7 built-in agents** via OpenClaw:

| Agent | Description |
|-------|-------------|
| Assistant | General-purpose conversational AI |
| ERP | CRM, inventory, and invoicing assistant |
| Sales | Lead scoring, pipeline management |
| Support | Customer support with knowledge base |
| Analytics | Business insights and reporting |
| Workflow | Automated workflow orchestration |
| Custom | User-defined agent via Agent Builder |

Additional agents can be created using the **Agent Builder** UI — drag-and-drop configuration with tool bindings, prompt templates, and RAG knowledge bases.

## Roles & Permissions

Pro Edition supports **5 roles** with granular permission sets:

| Role | Description |
|------|-------------|
| `OWNER` | Full access including billing, license, and all settings |
| `OPERATOR` | Day-to-day operations, workflow management |
| `MARKETER` | CRM, campaigns, channel management |
| `FINANCE` | Invoicing, payments, reports, expenses |
| `VIEWER` | Read-only access to dashboards and reports |

Role assignments are managed via the RBAC package (`@unicore-pro/rbac`) with per-resource permission overrides.

## Messaging Channels

Pro Edition includes **21+ channels**:

### Direct Messaging
- Telegram
- WhatsApp Business API
- LINE
- Facebook Messenger
- Instagram Direct
- WeChat (via bridge)
- Viber

### Email & Ticketing
- SMTP / IMAP
- Intercom
- Zendesk

### Team Collaboration
- Slack
- Microsoft Teams
- Discord

### Web & Widgets
- Web Chat (embedded widget)
- Live Chat (real-time dashboard)

### Business Channels
- SMS (Twilio, AWS SNS)
- Voice (Twilio)
- Push Notifications (FCM/APNs)
- REST API channel
- Webhooks (inbound/outbound)
- RSS/Atom feeds

Channel configuration is managed via the `@unicore-pro/channels` package and dashboard UI.

## RAG / Knowledge Base

- **Storage**: Unlimited vector storage in Qdrant
- Supports PDF, DOCX, TXT, CSV, HTML, and Markdown
- Auto-indexing on document upload
- Namespace isolation per agent or workspace
- Hybrid search (dense + sparse vectors)

## Advanced Workflows

The `@unicore-pro/workflows-advanced` package adds:

- Visual drag-and-drop workflow builder
- Conditional branching and parallel execution
- Sub-workflow nesting
- HTTP action nodes with retry logic
- AI action nodes (agent calls within workflows)
- Unlimited active workflows

## SSO (Single Sign-On)

The `@unicore-pro/sso` package supports:

- **OIDC** (OpenID Connect) — Google Workspace, Auth0, Keycloak
- **SAML 2.0** — Okta, Microsoft Entra ID (Azure AD)

Configuration is done via the SSO settings page in the dashboard. SSO sessions are issued as JWT tokens compatible with the API Gateway.

## White-Label Branding

The `@unicore-pro/branding` package enables:

- Custom logo, favicon, and color palette
- Custom application name and metadata
- Per-workspace theme overrides
- Email template customization

## Custom Domains

The `@unicore-pro/domains` package enables:

- Multiple custom domains per workspace
- Automatic SSL via Let's Encrypt (ACME)
- Domain-based routing via Nginx

## Audit Logging

The `@unicore-pro/audit` package provides:

- Full audit trail for all user actions
- Filterable by user, resource, action, and time range
- Export to CSV or JSON
- Retention policy configuration (default: 90 days)

## Feature Flags (Environment Variables)

```bash
UNICORE_EDITION=full
ENABLE_SSO=true
ENABLE_WHITE_LABEL=true
ENABLE_ADVANCED_WORKFLOWS=true
ENABLE_ALL_CHANNELS=true
ENABLE_CUSTOM_DOMAINS=true
ENABLE_ADVANCED_ANALYTICS=true
ENABLE_PRIORITY_SUPPORT=true
```

These are set automatically in the root `docker-compose.yml` for the full production stack.

## Deployment

Pro Edition uses the root workspace compose file which includes all feature flags:

```bash
cd /var/platforms/unicores

# Start full stack (community + pro)
docker compose --profile apps --profile workflows up -d
```

See the [Docker deployment guide](../deployment/docker.md) for complete instructions.

## Upgrading from Community

Pro Edition is additive — no data migration required. Steps:

1. Obtain a license key
2. Update your compose file to use `UNICORE_EDITION: full` with all Pro flags
3. Rebuild and restart the affected services
4. Activate the license key via the dashboard Settings > License

---

© 2026 BeMind Technology — Licensed under BSL 1.1
