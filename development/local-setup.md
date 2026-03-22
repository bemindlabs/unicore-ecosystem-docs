# Local Development Setup

Updated: 2026-03-22

This guide walks you through setting up a full UniCore development environment on your local machine.

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Node.js | 20+ | [nodejs.org](https://nodejs.org) or `nvm` |
| pnpm | 10.30+ | `npm install -g pnpm` |
| Docker Desktop | 24+ | [docker.com](https://www.docker.com/products/docker-desktop/) |
| Docker Compose | V2 | Included with Docker Desktop |
| Git | 2.40+ | System package or [git-scm.com](https://git-scm.com) |

### Recommended Tools

- **VS Code** with extensions: ESLint, Prettier, Prisma, Docker, GitLens
- **TablePlus** or **pgAdmin** for PostgreSQL inspection
- **Insomnia** or **Postman** for API testing

## Clone the Repository

### Community Edition (Most Contributors)

```bash
git clone git@github.com:bemindlabs/unicore.git
cd unicore
```

### Full Ecosystem (Pro / License development)

```bash
git clone --recurse-submodules git@github.com:bemindlabs/unicore-ecosystem.git
cd unicore-ecosystem

# If you already cloned without submodules:
git submodule update --init --recursive
```

## Install Dependencies

```bash
# From the repo root (uses pnpm workspaces)
pnpm install
```

This installs all dependencies for all packages in the monorepo using Turborepo + pnpm workspaces.

## Environment Configuration

```bash
# Copy the example env file
cp .env.example .env

# Open and configure
nano .env
```

### Key Variables

```bash
# Database (matches Docker Compose defaults)
DATABASE_URL="postgresql://<db-user>:<db-password>@localhost:5433/unicore"
ERP_DATABASE_URL="postgresql://<db-user>:<db-password>@localhost:5433/unicore_erp"

# Redis
REDIS_URL="redis://localhost:6380"

# JWT
JWT_SECRET="<your-jwt-secret-min-32-chars>"
JWT_REFRESH_SECRET="<your-refresh-secret-min-32-chars>"

# Bootstrap
BOOTSTRAP_SECRET="<your-bootstrap-secret>"

# AI provider keys are configured via dashboard Settings → AI → Providers

# Qdrant (vector DB)
QDRANT_URL="http://localhost:6333"

# Kafka
KAFKA_BROKERS="localhost:9092"

# Service URLs (for development)
NEXT_PUBLIC_API_URL="http://localhost:4000"
NEXT_PUBLIC_WS_URL="ws://localhost:18789"
```

All default values work with the Docker Compose setup below.

## Start Infrastructure Services

Use Docker Compose to start the databases and message broker only (run application services natively for hot-reload):

```bash
# Option A: Start only infrastructure (PostgreSQL, Redis, Qdrant, Kafka)
docker compose -f docker-compose.dev.yml up -d

# Option B: Start the full stack (all 17 containers)
docker compose --profile apps --profile workflows up -d
```

> For active development, Option A is preferred — you run each service natively with hot-reload, while infrastructure runs in containers.

## Database Setup

```bash
# Ensure the postgres container is running
docker ps | grep postgres

# Create ERP database
docker exec unicores-unicore-postgres-1 psql -U unicore -d postgres \
  -c "CREATE DATABASE unicore_erp OWNER unicore;"

# Push API Gateway schema (users, sessions, settings)
cd services/api-gateway
npx prisma db push --accept-data-loss
cd ../..

# Push ERP schema (contacts, products, orders, invoices)
cd services/erp
npx prisma db push --accept-data-loss
cd ../..
```

### Prisma Studio (Visual DB Browser)

```bash
# Browse API Gateway data
cd services/api-gateway
npx prisma studio  # opens on http://localhost:5555

# Browse ERP data
cd services/erp
npx prisma studio  # opens on http://localhost:5555
```

## Start Application Services

### Option A: Start All Services via pnpm (Turborepo)

```bash
# Start everything in development mode
pnpm dev

# Or start a specific service
pnpm --filter @unicore/dashboard dev
pnpm --filter @unicore/api-gateway dev
pnpm --filter @unicore/erp dev
```

### Option B: Start Services Individually

```bash
# Terminal 1 — Dashboard (Next.js, hot-reload)
cd apps/dashboard
pnpm dev
# → http://localhost:3000

# Terminal 2 — API Gateway (NestJS, watch mode)
cd services/api-gateway
pnpm start:dev
# → http://localhost:4000

# Terminal 3 — ERP service
cd services/erp
pnpm start:dev
# → http://localhost:4100

# Terminal 4 — AI Engine
cd services/ai-engine
pnpm start:dev
# → http://localhost:4200

# Terminal 5 — RAG service
cd services/rag
pnpm start:dev
# → http://localhost:4300
```

## Provision Admin User

```bash
curl -s http://localhost:4000/auth/provision-admin -X POST \
  -H 'Content-Type: application/json' \
  -H 'X-Bootstrap-Secret: <your-bootstrap-secret>' \
  -d '{"email":"<your-admin-email>","password":"<your-admin-password>","name":"Admin"}'
```

Visit `http://localhost:3000` and log in with `<your-admin-email>` / `<your-admin-password>`.

## Verify Setup

```bash
# Check API Gateway health
curl http://localhost:4000/health
# → {"status":"ok","info":{"database":{"status":"up"},"redis":{"status":"up"}}}

# Check ERP health
curl http://localhost:4100/health

# Check RAG health
curl http://localhost:4300/health

# Check AI Engine health
curl http://localhost:4200/health
```

## Building Packages

```bash
# Build all packages (Turborepo, respects dependency order)
pnpm build

# Build a specific package
pnpm --filter @unicore/shared-types build
pnpm --filter @unicore/ui build

# Watch mode (for package development)
pnpm --filter @unicore/shared-types dev
```

## Running Tests

See the full [testing guide](./testing.md) for details.

```bash
# Unit tests (all packages)
pnpm test

# Unit tests for a specific service
pnpm --filter @unicore/erp test

# Watch mode
pnpm --filter @unicore/api-gateway test --watch

# E2E tests (requires full stack running)
pnpm test:e2e
```

## Linting and Formatting

```bash
# Lint all packages
pnpm lint

# Auto-fix lint issues
pnpm lint --fix

# Format with Prettier
pnpm format

# TypeScript type checking
pnpm typecheck
```

## Working with Prisma

```bash
# After modifying a schema file, push changes to the database
cd services/api-gateway
npx prisma db push

# Generate Prisma Client after schema changes
npx prisma generate

# Open Prisma Studio
npx prisma studio

# Reset the database (WARNING: destroys data)
npx prisma db push --force-reset
```

Note: UniCore uses `db push` (schema-first, no migrations). Do not run `prisma migrate`.

## Common Issues

### Port conflicts

```bash
# Find what's using a port
sudo lsof -i :4000
sudo lsof -i :3000
sudo lsof -i :5433
```

Adjust ports in `.env` if needed.

### pnpm workspace not resolving packages

```bash
# Clear node_modules and reinstall
find . -name node_modules -prune -exec rm -rf {} \;
pnpm install
```

### Prisma Client not generated

```bash
pnpm --filter @unicore/api-gateway exec npx prisma generate
```

### Kafka connection refused

Ensure the workflows profile is running:

```bash
docker compose --profile workflows up -d
docker logs unicores-unicore-kafka-1
```

## VS Code Configuration

Recommended `.vscode/settings.json` for the workspace:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.workingDirectories": [
    "apps/dashboard",
    "services/api-gateway",
    "services/erp",
    "services/ai-engine",
    "services/rag"
  ],
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

## Related Documentation

- [Contributing guide](./contributing.md)
- [Coding standards](./coding-standards.md)
- [Testing guide](./testing.md)
- [Docker Compose deployment](../deployment/docker.md)

---

© 2026 BeMind Technology
