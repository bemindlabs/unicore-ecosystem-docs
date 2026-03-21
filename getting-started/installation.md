# Installation

This guide walks you through cloning the UniCore ecosystem, configuring the environment, starting all 17 containers, and verifying a working installation.

> Before starting, confirm your system meets the [system requirements](system-requirements.md).

## Quick Start (TL;DR)

```bash
git clone --recurse-submodules git@github.com:bemindlabs/unicore-ecosystem.git unicores
cd unicores
cp .env.example .env          # edit secrets
make up                       # start all containers
make db/setup                 # initialize databases and provision admin
```

Then open [http://localhost:3000](http://localhost:3000).

---

## Step 1 — Clone the Repository

UniCore uses Git submodules. You must pass `--recurse-submodules` to clone all three repos in one command:

```bash
git clone --recurse-submodules git@github.com:bemindlabs/unicore-ecosystem.git unicores
cd unicores
```

If you already cloned without `--recurse-submodules`, initialize submodules manually:

```bash
git submodule update --init --recursive
```

After cloning, the directory structure should look like:

```
unicores/
├── unicore/             ← Community edition (NestJS services + Next.js dashboard)
├── unicore-pro/         ← Pro extensions
├── unicore-license/     ← License server
├── docker-compose.yml   ← Root compose (full system, production)
├── Makefile             ← Automation shortcuts
└── .env.example         ← Environment template
```

## Step 2 — Install Dependencies (Optional, for local dev only)

Docker deployments do not require a local Node.js installation. Skip to Step 3 if you only need to run via Docker.

For local (non-Docker) development:

```bash
# Requires Node.js 20+ and pnpm 10.30+
npm install -g pnpm@latest
make install
# or: pnpm install
```

This installs all workspace packages across `unicore/`, `unicore-pro/`, and `unicore-license/` via pnpm workspaces.

## Step 3 — Configure Environment

Copy the root `.env.example` to `.env` and fill in the required values:

```bash
cp .env.example .env
```

Open `.env` in your editor and set the required secrets:

```bash
# Minimum required changes:
POSTGRES_PASSWORD=<your-strong-database-password>
JWT_SECRET=<your-32-character-minimum-secret-key>
BOOTSTRAP_SECRET=<your-bootstrap-secret>
LICENSE_ADMIN_SECRET=<your-license-admin-secret>
```

See [configuration.md](configuration.md) for a full reference of all environment variables, including AI provider keys and integration tokens.

> **Security note**: Never commit your `.env` file to version control. The `.gitignore` already excludes it.

## Step 4 — Start Services

Start the full platform using Make or Docker Compose directly:

```bash
# Using Make (recommended)
make up

# Or directly with Docker Compose
docker compose --profile apps --profile workflows up -d
```

This starts all 17 containers. On the first run, Docker will pull base images (~5 GB). Subsequent starts are fast.

To start only infrastructure (databases, Redis, Kafka, Qdrant) without application services:

```bash
make up/infra
# or: docker compose up -d
```

## Step 5 — Database Setup

New deployments require schema initialization. Run the full setup with one command:

```bash
make db/setup
```

This performs the following steps automatically:

1. Creates the `unicore_erp` database in PostgreSQL
2. Pushes the API Gateway Prisma schema (User, Session, Settings, etc.)
3. Pushes the ERP Prisma schema (Contact, Product, Order, Invoice, etc.)
4. Pushes the License API Prisma schema
5. Restarts the API gateway and ERP services
6. Provisions the default admin user

Or run each step manually:

```bash
# Create ERP database
make db/create-erp

# Push all Prisma schemas
make db/push-all

# Restart affected services
docker compose --profile apps restart unicore-api-gateway unicore-erp

# Provision admin user
make db/provision-admin
```

## Step 6 — Verify Installation

After startup, run a basic health check:

```bash
# Check all containers are running
make ps
# or: docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

All containers should show `Up` or `healthy` status. Expected containers:

```
unicores-unicore-dashboard-1       Up
unicores-unicore-api-gateway-1     Up
unicores-unicore-erp-1             Up
unicores-unicore-ai-engine-1       Up
unicores-unicore-rag-1             Up
unicores-unicore-bootstrap-1       Up
unicores-unicore-license-api-1     Up
unicores-unicore-openclaw-gateway-1 Up
unicores-unicore-workflow-1        Up
unicores-unicore-nginx-1           Up
unicores-unicore-postgres-1        Up (healthy)
unicores-unicore-redis-1           Up (healthy)
unicores-unicore-vectordb-1        Up
unicores-unicore-kafka-1           Up
unicores-unicore-zookeeper-1       Up
unicores-unicore-license-db-1      Up
unicores-unicore-license-redis-1   Up
```

Test the API gateway health endpoint:

```bash
curl -s http://localhost:4000/health
# Expected: {"status":"ok"}
```

Test authentication (using the provisioned admin):

```bash
curl -s http://localhost:4000/auth/login \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"email":"<your-admin-email>","password":"<your-admin-password>"}'
# Expected: {"access_token":"...","user":{...}}
```

Open the dashboard in your browser: [http://localhost:3000](http://localhost:3000)

Default login credentials (set during provisioning):
- **Email**: `<your-admin-email>`
- **Password**: `<your-admin-password>`

> Use a strong password and change it immediately after your first login in production environments.

## Rebuilding Individual Services

After making code changes to a service, rebuild and redeploy only that container:

```bash
# Rebuild the dashboard
make build/dashboard

# Rebuild the API gateway
make build/api-gateway

# Rebuild any other service
make build/erp
make build/ai-engine
make build/rag
make build/bootstrap
make build/openclaw
make build/workflow
make build/license-api
```

## Full Fresh Deploy

To tear down everything and start from scratch:

```bash
make fresh
# Equivalent to: make down && make build && make up && make db/setup
```

> **Warning**: `make fresh` removes all containers and rebuilds all images. It does not delete volumes (database data). To also wipe data volumes, run `docker compose --profile apps --profile workflows down -v` before `make fresh`.

## Troubleshooting

**Containers keep restarting**: Check logs for the failing service:

```bash
make logs/api-gateway
make logs/erp
# or any other service name
```

**Database connection refused**: Ensure PostgreSQL is healthy before running `db/setup`:

```bash
docker compose ps unicore-postgres
# Wait until Status shows "healthy", then retry db/setup
```

**Port already in use**: Find and stop the conflicting process, or edit the host port in `docker-compose.yml`. See [System Requirements](system-requirements.md) for the full port list.

**Submodule directory empty**: Run `git submodule update --init --recursive` from the repo root.
