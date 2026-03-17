# Docker Compose Deployment

Docker Compose is the primary and recommended deployment method for UniCore. The root workspace compose file orchestrates all 17 containers across the full stack.

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Docker Engine | 24+ | [Install guide](https://docs.docker.com/engine/install/) |
| Docker Compose | V2 (plugin) | Included with Docker Desktop and Engine 24+ |
| RAM | 8 GB minimum | 16 GB recommended for full stack |
| CPU | 4 cores minimum | 8 cores recommended |
| Disk | 50 GB free | For images, volumes, and data |

## Compose Files

| File | Purpose |
|------|---------|
| `/var/platforms/unicores/docker-compose.yml` | **Production** — full stack, all Pro flags, `UNICORE_EDITION: full` |
| `unicore/docker-compose.yml` | Community dev — `UNICORE_EDITION: community`, no Pro flags |
| `unicore-pro/docker-compose.yml` | Pro overlay — service overrides only, no new containers |

**Always use the root compose file in production.**

## Profiles

The root compose uses Docker profiles to group services:

| Profile | Services |
|---------|---------|
| `apps` | All application services (dashboard, api-gateway, erp, ai-engine, rag, bootstrap, openclaw, nginx) |
| `workflows` | Kafka, Zookeeper, and workflow engine |

## Full Stack Deployment

### 1. Clone and Configure

```bash
git clone --recurse-submodules git@github.com:bemindlabs/unicore-ecosystem.git
cd unicore-ecosystem

# Copy root environment file
cp .env.example .env

# Edit secrets and configuration
nano .env
```

Key environment variables:

```bash
# Database
POSTGRES_USER=unicore
POSTGRES_PASSWORD=your-secure-password
POSTGRES_DB=unicore

# Redis
REDIS_PASSWORD=your-redis-password

# JWT
JWT_SECRET=your-256-bit-secret

# License (Pro/Enterprise only)
LICENSE_KEY=UC-XXXX-XXXX-XXXX-XXXX

# Bootstrap
BOOTSTRAP_SECRET=your-bootstrap-secret
```

### 2. Start All Services

```bash
cd /var/platforms/unicores

# Start application services and workflows
docker compose --profile apps --profile workflows up -d
```

### 3. Database Setup

```bash
# Create ERP database
docker exec unicores-unicore-postgres-1 psql -U unicore -d postgres \
  -c "CREATE DATABASE unicore_erp OWNER unicore;"

# Push auth + settings schema
docker exec unicores-unicore-api-gateway-1 npx prisma db push --accept-data-loss

# Push ERP schema
docker exec unicores-unicore-erp-1 npx prisma db push --accept-data-loss
```

### 4. Restart Services

```bash
docker compose --profile apps restart unicore-api-gateway unicore-erp
```

### 5. Provision Admin User

```bash
curl -s http://localhost:4000/auth/provision-admin -X POST \
  -H 'Content-Type: application/json' \
  -H 'X-Bootstrap-Secret: your-bootstrap-secret' \
  -d '{"email":"admin@example.com","password":"yourpassword","name":"Admin"}'
```

### 6. Verify

```bash
# Check all containers are running
docker compose --profile apps --profile workflows ps

# Test the API gateway
curl http://localhost:4000/health

# Access the dashboard
open http://localhost:3000
```

## Service Details

| Service | Container Name | Port | Health Check |
|---------|---------------|------|-------------|
| Dashboard | `unicores-unicore-dashboard-1` | 3000 | `GET /` |
| API Gateway | `unicores-unicore-api-gateway-1` | 4000 | `GET /health` |
| ERP | `unicores-unicore-erp-1` | 4100 | `GET /health` |
| AI Engine | `unicores-unicore-ai-engine-1` | 4200 | `GET /health` |
| RAG | `unicores-unicore-rag-1` | 4300 | `GET /health` |
| Bootstrap | `unicores-unicore-bootstrap-1` | 4500 | `GET /health` |
| License API | `unicores-unicore-license-api-1` | 4600 | `GET /health` |
| OpenClaw | `unicores-unicore-openclaw-gateway-1` | 18789/18790 | `GET /health` |
| Nginx | `unicores-unicore-nginx-1` | 80 | — |
| PostgreSQL | `unicores-unicore-postgres-1` | 5433 | — |
| Redis | `unicores-unicore-redis-1` | 6380 | — |
| Qdrant | `unicores-unicore-vectordb-1` | 6333 | `GET /health` |
| Kafka | `unicores-unicore-kafka-1` | 9092 | — |
| Zookeeper | `unicores-unicore-zookeeper-1` | 2181 | — |

## Rebuilding Services

### Rebuild a Single Service

```bash
# Rebuild dashboard (e.g., after frontend changes)
docker compose --profile apps build unicore-dashboard --no-cache
docker compose --profile apps up -d unicore-dashboard

# Rebuild API gateway
docker compose --profile apps build unicore-api-gateway --no-cache
docker compose --profile apps up -d unicore-api-gateway

# Rebuild ERP service
docker compose --profile apps build unicore-erp --no-cache
docker compose --profile apps up -d unicore-erp
```

### Rebuild All Services

```bash
docker compose --profile apps --profile workflows build --no-cache
docker compose --profile apps --profile workflows up -d
```

### Restart Without Rebuild

```bash
# Restart a specific service
docker compose --profile apps restart unicore-nginx

# Restart all services
docker compose --profile apps --profile workflows restart
```

## Viewing Logs

```bash
# Follow logs for a specific service
docker logs -f unicores-unicore-dashboard-1
docker logs -f unicores-unicore-api-gateway-1
docker logs -f unicores-unicore-erp-1

# Last 100 lines
docker logs --tail 100 unicores-unicore-api-gateway-1

# All services (compose)
docker compose --profile apps logs -f
docker compose --profile apps logs --tail 50 unicore-api-gateway
```

## Stopping Services

```bash
# Stop all services (keep volumes)
docker compose --profile apps --profile workflows down

# Stop and remove volumes (WARNING: destroys data)
docker compose --profile apps --profile workflows down -v

# Stop a single service
docker compose --profile apps stop unicore-dashboard
```

## Nginx Configuration

Nginx is the reverse proxy routing all traffic to internal services:

```
Cloudflare (unicore.bemind.tech)
  → Nginx Proxy Manager (:80)
    → unicores-unicore-nginx-1 (:80)
      → Dashboard    /:3000
      → API Gateway  /api/, /auth/, /webhooks/ :4000
      → OpenClaw     /ws  :18789 (WebSocket)
```

The Nginx config lives at `unicore/nginx/default.conf`. After editing:

```bash
# Reload nginx config (no downtime)
docker exec unicores-unicore-nginx-1 nginx -s reload

# Or restart the container
docker compose --profile apps restart unicore-nginx
```

## Community Edition (Development)

To run only the community services (no Pro flags):

```bash
cd /var/platforms/unicores/unicore

docker compose --profile apps up -d
```

## Updating Submodules

When submodule code is updated:

```bash
cd /var/platforms/unicores

# Pull latest submodule commits
git submodule update --remote --merge

# Rebuild affected services
docker compose --profile apps build --no-cache
docker compose --profile apps up -d
```

## Troubleshooting

### Container exits immediately

```bash
# Check exit logs
docker logs unicores-unicore-api-gateway-1

# Common causes: missing .env vars, database not ready, port conflicts
```

### Port already in use

```bash
# Find what's using a port
sudo lsof -i :4000

# Or use a different host port in docker-compose override
```

### Database connection refused

Ensure PostgreSQL container is healthy before running `prisma db push`:

```bash
docker exec unicores-unicore-postgres-1 pg_isready -U unicore
```

### Prisma schema out of sync

```bash
docker exec unicores-unicore-api-gateway-1 npx prisma db push --accept-data-loss
docker exec unicores-unicore-erp-1 npx prisma db push --accept-data-loss
```

---

© 2026 BeMind Technology
