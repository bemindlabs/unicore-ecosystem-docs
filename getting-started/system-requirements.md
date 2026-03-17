# System Requirements

This page describes the hardware, software, and network requirements for running UniCore.

## Hardware

### Minimum (Development / Evaluation)

| Resource | Minimum |
|----------|---------|
| CPU | 4 cores, x86-64 |
| RAM | 8 GB |
| Disk | 20 GB free (5 GB for Docker images, remainder for data) |
| Network | Outbound internet access (for image pulls and AI provider APIs) |

### Recommended (Production)

| Resource | Recommended |
|----------|------------|
| CPU | 8+ cores, x86-64 |
| RAM | 16 GB+ |
| Disk | 60 GB+ SSD (NVMe preferred) |
| Network | 100 Mbps+, static IP or stable DNS |

> **Note**: The 17-container stack pulls approximately 5 GB of Docker images on the first run. PostgreSQL, Qdrant, and Kafka are the most memory-intensive services.

## Operating System

| OS | Support Status | Notes |
|----|---------------|-------|
| Ubuntu 22.04 / 24.04 LTS | Fully supported | Recommended for production |
| Debian 12 | Fully supported | |
| Other Linux (x86-64) | Community supported | Kernel 5.15+ required |
| macOS 13+ (Apple Silicon / Intel) | Supported for development | Use Docker Desktop 4.26+ |
| Windows 10/11 | WSL 2 only | See [WSL 2 notes](#windows-wsl-2) below |

### Windows WSL 2

UniCore is not natively supported on Windows. Use WSL 2 with Ubuntu 22.04 or 24.04:

1. Enable WSL 2: `wsl --install -d Ubuntu-22.04`
2. Install Docker Desktop with the WSL 2 backend enabled.
3. Clone and run all commands inside the WSL 2 terminal — not PowerShell or CMD.
4. Allocate at least 8 GB RAM to WSL 2 via `%USERPROFILE%\.wslconfig`:

```ini
[wsl2]
memory=8GB
processors=4
```

## Software Prerequisites

| Tool | Minimum Version | Purpose |
|------|----------------|---------|
| Docker Engine | 24.0+ | Container runtime |
| Docker Compose plugin | 2.20+ | Multi-container orchestration |
| Git | 2.38+ | Repository and submodule management |
| Make | 4.3+ | Makefile automation (optional but recommended) |
| Node.js | 20.0+ | Required only for local (non-Docker) development |
| pnpm | 10.30+ | Required only for local (non-Docker) development |

Check installed versions:

```bash
docker --version
docker compose version
git --version
make --version
```

## Port Requirements

The following ports must be available on the host machine. All services bind to `127.0.0.1` by default.

| Port | Service | Protocol | Notes |
|------|---------|----------|-------|
| 80 | Nginx reverse proxy | HTTP | Public-facing entry point |
| 3000 | Dashboard | HTTP | Next.js frontend |
| 4000 | API Gateway | HTTP | REST API, auth, webhooks |
| 4100 | ERP | HTTP | CRM, inventory, invoicing |
| 4200 | AI Engine | HTTP | Model orchestration |
| 4300 | RAG | HTTP | Vector search |
| 4500 | Bootstrap | HTTP | Setup wizard |
| 4600 | License API | HTTP | License validation |
| 5433 | PostgreSQL | TCP | Host port (container uses 5432) |
| 6333 | Qdrant | HTTP | Vector database |
| 6380 | Redis | TCP | Host port (container uses 6379) |
| 9092 | Kafka | TCP | Event streaming |
| 18789 | OpenClaw Gateway | TCP/WebSocket | Multi-agent entry point |
| 18790 | OpenClaw (secondary) | TCP | Agent-to-agent communication |
| 2181 | Zookeeper | TCP | Kafka coordinator |
| 81 | Nginx Proxy Manager | HTTP | Admin UI (if using NPM) |

> If any of these ports are already bound by another process, update the host-side port mapping in `docker-compose.yml` before starting.

Check for port conflicts:

```bash
# Linux
ss -tlnp | grep -E '80|3000|4000|4100|4200|4300|4500|4600|5433|6333|6380|9092|18789'

# macOS
lsof -iTCP -sTCP:LISTEN -nP | grep -E '80|3000|4000'
```

## Disk Space Breakdown

| Component | Approximate Size |
|-----------|----------------|
| Docker base images (postgres, redis, kafka, qdrant, nginx, node) | ~3.5 GB |
| UniCore application images (built locally) | ~1.5 GB |
| PostgreSQL data (grows with usage) | 500 MB+ |
| Qdrant vector data (grows with knowledge base) | 200 MB+ |
| Redis data | 50 MB+ |
| Kafka logs (grows with event volume) | 200 MB+ |
| Source code and node_modules | ~2 GB |

Total initial footprint: **~8–10 GB** after a full build.

## Network Access

The following outbound connections are required:

| Destination | Purpose |
|-------------|---------|
| `registry-1.docker.io` | Docker Hub image pulls |
| `api.openai.com` | OpenAI API (if `OPENAI_API_KEY` is set) |
| `api.anthropic.com` | Anthropic API (if `ANTHROPIC_API_KEY` is set) |
| Your license server | License validation (Pro/Enterprise editions) |

For air-gapped deployments, pre-pull images and provide a local AI model via Ollama.
