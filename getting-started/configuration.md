# Configuration

UniCore is configured entirely through environment variables loaded from a `.env` file at the repository root. This page documents every variable, grouped by category.

## Overview

Two `.env.example` templates are provided:

| File | Used by | Purpose |
|------|---------|---------|
| `.env.example` (root) | `docker-compose.yml` | Full system (production, all Pro features enabled) |
| `unicore/.env.example` | `unicore/docker-compose.yml` | Community edition only (development) |

For production deployments using the root `docker-compose.yml`, always copy the root template:

```bash
cp .env.example .env
```

For community-only development:

```bash
cp unicore/.env.example unicore/.env
```

---

## Required Variables

These variables have no sensible default and **must** be set before starting the platform.

| Variable | Example | Description |
|----------|---------|-------------|
| `POSTGRES_PASSWORD` | `s3cureP@ss!` | PostgreSQL superuser password. Use a strong, unique value in production. |
| `JWT_SECRET` | `a1b2c3...` (32+ chars) | Secret key for signing JWT access tokens. Minimum 32 characters. Generate with `openssl rand -hex 32`. |
| `BOOTSTRAP_SECRET` | `boot-secret-xyz` | Authorization header for the `/auth/provision-admin` wizard endpoint. Change before exposing publicly. |
| `LICENSE_ADMIN_SECRET` | `lic-secret-xyz` | Admin secret for the license API service. |
| `LICENSE_DB_PASSWORD` | `s3cureP@ss2!` | PostgreSQL password for the isolated license database (`unicore-license-db`). Separate from the main `POSTGRES_PASSWORD`. |
| `UNICORE_PUBLIC_KEY` | (Ed25519 PEM) | Ed25519 public key used by the license server for verifying license key signatures. |
| `UNICORE_PRIVATE_KEY` | (Ed25519 PEM) | Ed25519 private key used by the license server for signing license keys. Keep strictly confidential. |

---

## Database

```bash
# PostgreSQL connection
POSTGRES_USER=unicore                     # Database superuser (default: unicore)
POSTGRES_PASSWORD=change-me               # [REQUIRED] Superuser password
POSTGRES_DB=unicore                       # Main application database name
POSTGRES_DB_ERP=unicore_erp              # ERP database (created separately via make db/create-erp)
POSTGRES_DB_LICENSE=unicore_license       # License server database
```

Internally, services connect via Docker's service DNS:

| Service | Database URL |
|---------|-------------|
| API Gateway | `postgresql://unicore:<pass>@unicore-postgres:5432/unicore` |
| ERP | `postgresql://unicore:<pass>@unicore-postgres:5432/unicore_erp` |
| License API | `postgresql://unicore:<pass>@unicore-license-db:5432/unicore_license` |

The host-exposed PostgreSQL port is **5433** (maps to container's 5432) to avoid conflicts with a locally installed Postgres instance.

---

## Authentication

```bash
JWT_SECRET=<your-32-character-minimum-jwt-secret>
BOOTSTRAP_SECRET=<your-bootstrap-provisioning-secret>
LICENSE_ADMIN_SECRET=<your-license-admin-secret>
```

**JWT_SECRET**: All services that validate JWT tokens need this value. It must be identical across the API gateway and any service that independently validates tokens.

**BOOTSTRAP_SECRET**: Sent as the `X-Bootstrap-Secret` header when calling `POST /auth/provision-admin`. This endpoint is only active before the wizard completes. After setup, this endpoint is locked.

---

## Frontend

```bash
# URL the browser uses to reach the API (via nginx proxy in production)
NEXT_PUBLIC_API_URL=https://<your-domain>

# For local development without nginx, point directly to the gateway:
# NEXT_PUBLIC_API_URL=http://localhost:3000    # routed through Next.js proxy
# NEXT_PUBLIC_API_URL=http://localhost:4000    # direct to API gateway

# WebSocket URL for OpenClaw (auto-detected if not set)
# NEXT_PUBLIC_WS_URL=ws://localhost:18789

# Edition flag displayed in the UI
# NEXT_PUBLIC_EDITION=community    # community | pro | enterprise | geek
```

> In production, `NEXT_PUBLIC_API_URL` should point to your public domain. The nginx reverse proxy routes `/api/`, `/auth/`, and `/webhooks/` to the API gateway automatically.

---

## AI Providers

UniCore's AI engine supports multiple LLM providers. API keys are configured via the dashboard **Settings → AI → Providers** page after deployment.

```bash
# Local Ollama (no key required)
# OLLAMA_BASE_URL=http://localhost:11434

# Primary provider selection (falls back to next available)
# LLM_PRIMARY_PROVIDER=openai       # openai | anthropic | ollama
```

If no API key is configured, AI features (chat, agent queries, RAG responses) will return an error. The platform itself (ERP, dashboard, settings) continues to function without AI keys.

**Provider selection logic**: The AI engine picks the primary provider based on `LLM_PRIMARY_PROVIDER`. If that provider's key is missing, it falls back to the next available one. If no providers are configured, AI endpoints return `503`.

---

## Infrastructure (Internal Services)

These variables configure how application services connect to infrastructure. The defaults work correctly within Docker Compose's internal network and typically do not need to be changed.

```bash
# Redis (session cache, rate limiting)
# REDIS_URL=redis://<redis-host>:6379        # Default within Docker Compose internal network
# External: redis://:<password>@<host>:6380  # If using external Redis

# Qdrant vector database (used by RAG service)
# QDRANT_URL=http://<vectordb-host>:6333

# Kafka event streaming (used by workflow engine)
# KAFKA_BROKERS=<kafka-host>:9092
```

For external managed services (e.g., AWS ElastiCache, Confluent Cloud), override these values with the appropriate connection strings.

---

## Service-to-Service Communication

```bash
# Shared secret for X-Internal-Service header validation between microservices
# INTERNAL_SERVICE_SECRET=<your-internal-service-secret>

# URL of the UniCore Platform (public website) for callbacks
# PLATFORM_URL=https://<your-domain>
# PLATFORM_CALLBACK_SECRET=<your-platform-callback-secret>
```

`INTERNAL_SERVICE_SECRET` is used when services call each other internally (e.g., AI Engine fetching API keys from the API Gateway via `GET /api/v1/settings/ai-config/keys`). The receiving service validates the `X-Internal-Service` header against a whitelist.

---

## Demo Mode

```bash
# Enable demo mode — blocks write operations for non-OWNER users
DEMO_MODE=true    # true | false (default: true in root compose)
```

When `DEMO_MODE=true`, all mutating API requests from users without the `OWNER` role are rejected. This is useful for read-only demo environments. Set to `false` for real deployments.

---

## Proxy Services

```bash
# ChatGPT-to-API proxy admin password
# CHATGPT_PROXY_ADMIN_PASSWORD=<your-proxy-admin-password>

# SSH proxy (Sshwifty) shared key for the web-based SSH terminal
# SSH_PROXY_SHARED_KEY=<your-ssh-proxy-key>
```

The ChatGPT proxy (`unicore-chatgpt-proxy`) converts ChatGPT subscription access tokens into OpenAI-compatible API calls. The SSH proxy (`unicore-ssh-proxy`) provides a browser-based SSH terminal. Both services are bound to `127.0.0.1` and are not directly accessible from the internet.

---

## Integrations (Pro Edition)

Messaging channel integrations are configured via the dashboard **Settings → Channels** page. Channel credentials (Telegram bot tokens, LINE channel secrets, etc.) are stored securely in the database and do not need to be set as environment variables.

---

## Edition & Feature Flags

The root `docker-compose.yml` sets these flags automatically for the full (Pro) edition. They are passed to service containers as environment variables and do not need to be in `.env`.

| Flag | Default (full) | Description |
|------|---------------|-------------|
| `UNICORE_EDITION` | `full` | Edition identifier: `community`, `pro`, `enterprise`, `geek`, `full` |
| `ENABLE_SSO` | `true` | Single sign-on (SAML, OIDC, Active Directory) |
| `ENABLE_WHITE_LABEL` | `true` | Custom branding and domain support |
| `ENABLE_ADVANCED_WORKFLOWS` | `true` | Visual workflow builder |
| `ENABLE_ALL_CHANNELS` | `true` | All messaging channel integrations |
| `ENABLE_CUSTOM_DOMAINS` | `true` | Per-tenant custom domains |
| `ENABLE_ADVANCED_ANALYTICS` | `true` | Extended reporting and dashboards |
| `ENABLE_PRIORITY_SUPPORT` | `true` | Priority support flag |

To disable a Pro feature in your deployment, set the flag to `false` in your root `docker-compose.yml` or override via environment.

---

## License Keys

Pro and Enterprise editions require a valid license key issued by the UniCore license server.

```bash
# License server endpoint (defaults to bundled license-api container)
# LICENSE_API_URL=http://unicore-license-api:4600

# License key (format: UC-XXXX-XXXX-XXXX-XXXX)
# UNICORE_LICENSE_KEY=UC-ABCD-1234-EFGH-5678
```

License keys are validated on startup and periodically refreshed. A machine fingerprint (based on hardware identifiers) is sent to the license server during activation. See the [License Server documentation](../editions/pro.md) for key generation and management.

---

## Per-Service Configuration Reference

Each service reads a subset of environment variables. The table below shows which services consume each category of configuration.

| Category | API Gateway | ERP | AI Engine | RAG | Bootstrap | License API | OpenClaw |
|----------|:-----------:|:---:|:---------:|:---:|:---------:|:-----------:|:--------:|
| Database | + | + | | | | + | |
| JWT | + | + | + | | + | | + |
| Redis | + | | + | | | + | |
| Kafka | + | + | | | | | + |
| AI keys | | | + | + | | | + |
| Qdrant | | | | + | | | |
| Channels | | | | | | | + |

---

## Generating Secrets

Use these commands to generate cryptographically strong secrets:

```bash
# 32-byte JWT secret (hex)
openssl rand -hex 32

# 64-byte secret (base64)
openssl rand -base64 48

# UUID-style bootstrap secret
python3 -c "import uuid; print(str(uuid.uuid4()))"
```

Never use the placeholder values from `.env.example` in a production deployment.
