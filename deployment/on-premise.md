# On-Premise Deployment

This guide covers self-hosted, on-premise deployment of UniCore — running all services on your own hardware, within your own network, without any cloud dependency.

## Use Cases

- Organizations with strict data residency requirements
- Air-gapped environments (no internet access)
- Enterprise compliance (HIPAA, ISO 27001, government)
- Cost optimization for high-traffic workloads

## Hardware Requirements

### Minimum (Development / Small Production)

| Resource | Minimum |
|----------|---------|
| CPU | 4 cores (x86_64) |
| RAM | 8 GB |
| Disk | 100 GB SSD |
| Network | 100 Mbps |

### Recommended (Medium Production)

| Resource | Recommended |
|----------|------------|
| CPU | 8 cores |
| RAM | 16–32 GB |
| Disk | 500 GB SSD (RAID 1 recommended) |
| Network | 1 Gbps |

### High Availability (Large Production)

For HA deployments, use at minimum:

- 3 application nodes (load-balanced)
- 1 PostgreSQL primary + 1 read replica
- 2 Redis nodes (Sentinel)
- 3 Kafka brokers

## Operating System

| OS | Version | Status |
|----|---------|--------|
| Ubuntu | 22.04 LTS | Recommended |
| Ubuntu | 20.04 LTS | Supported |
| Debian | 12 (Bookworm) | Supported |
| CentOS Stream | 9 | Supported |
| RHEL | 9 | Supported (Enterprise) |

## Software Prerequisites

```bash
# Ubuntu 22.04 — install Docker and Docker Compose
sudo apt-get update
sudo apt-get install -y ca-certificates curl git make
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli \
  containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version     # Docker version 24+
docker compose version  # Docker Compose version v2+
```

## Network Configuration

### Port Requirements

The following ports must be accessible on the host:

| Port | Protocol | Service | Exposure |
|------|----------|---------|----------|
| 80 | TCP | Nginx (HTTP) | Public |
| 443 | TCP | Nginx (HTTPS) | Public |
| 22 | TCP | SSH | Admin only |

All other services communicate on the Docker bridge network and are not exposed to the host by default. Internal service ports (3000, 4000, 4100, etc.) should **not** be exposed publicly.

### Firewall Configuration (ufw)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

### DNS Configuration

Point your domain to the server's public or private IP:

```
A record: unicore.yourdomain.com → YOUR_SERVER_IP
```

For internal/intranet deployments, configure your internal DNS server or use `/etc/hosts` on client machines.

## SSL / TLS Configuration

### Option 1: Let's Encrypt (Nginx Proxy Manager)

Nginx Proxy Manager (NPM) runs on port 81 and handles SSL termination via Let's Encrypt ACME:

1. Access NPM at `http://YOUR_SERVER_IP:81`
2. Default credentials: `admin@example.com` / `changeme` (change immediately)
3. Add a proxy host for `unicore.yourdomain.com`
4. Enable SSL and request a Let's Encrypt certificate
5. Forward to the unicore-nginx container (e.g. `<workspace>-unicore-nginx-1:80`)

```
# NPM setup for UniCore
Proxy Host: unicore.yourdomain.com
Forward to: http://<nginx-container>:80
SSL: Let's Encrypt (auto-renew enabled)
```

### Option 2: Self-Signed Certificate (Internal Networks)

```bash
# Generate self-signed cert
sudo mkdir -p /etc/unicore/ssl
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
  -keyout /etc/unicore/ssl/unicore.key \
  -out /etc/unicore/ssl/unicore.crt \
  -subj "/C=TH/ST=Bangkok/O=YourOrg/CN=unicore.local"
```

Mount the certs into the Nginx container via your docker-compose override.

### Option 3: Corporate CA Certificate

```bash
# Install corporate root CA
sudo cp corporate-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Mount corporate cert into Nginx container
```

## Deployment

Follow the standard [Docker Compose deployment guide](./docker.md) for the full deployment procedure.

### Offline / Air-Gapped Deployment

For environments without internet access, pre-pull all Docker images on a connected machine and transfer them:

```bash
# On a connected machine: save all images
docker compose --profile apps --profile workflows pull
docker save \
  unicore-dashboard \
  unicore-api-gateway \
  unicore-erp \
  unicore-ai-engine \
  unicore-rag \
  unicore-bootstrap \
  unicore-openclaw-gateway \
  unicore-workflow \
  nginx:alpine \
  postgres:16 \
  redis:7 \
  qdrant/qdrant \
  confluentinc/cp-kafka:7.5 \
  confluentinc/cp-zookeeper:7.5 \
  | gzip > unicore-images.tar.gz

# Transfer to air-gapped machine (USB, SCP, etc.)
# Then load:
docker load < unicore-images.tar.gz
```

Use a private Docker registry (Harbor, Nexus, AWS ECR in isolated VPC) for ongoing image distribution in air-gapped environments.

## Data Storage

### Volume Locations

Docker volumes are stored at `/var/lib/docker/volumes/` by default. Consider moving the Docker data root to a dedicated disk:

```bash
# Move Docker data root to /data
sudo systemctl stop docker
sudo rsync -avzP /var/lib/docker/ /data/docker/
sudo rm -rf /var/lib/docker

# Edit /etc/docker/daemon.json
sudo tee /etc/docker/daemon.json <<EOF
{
  "data-root": "/data/docker"
}
EOF

sudo systemctl start docker
```

### Volume Backup

Replace `<postgres-container>` and `<qdrant-container>` with your actual container names (derived from the workspace directory name).

```bash
# Backup PostgreSQL (daily, via cron)
0 2 * * * docker exec <postgres-container> \
  pg_dumpall -U unicore | gzip > /backups/postgres/unicore-$(date +%Y%m%d).sql.gz

# Backup PostgreSQL ERP database
0 2 * * * docker exec <postgres-container> \
  pg_dump -U unicore unicore_erp | gzip > /backups/postgres/unicore-erp-$(date +%Y%m%d).sql.gz

# Backup Qdrant vectors
0 3 * * * docker run --rm \
  --volumes-from <qdrant-container> \
  -v /backups/qdrant:/backup \
  busybox tar czf /backup/qdrant-$(date +%Y%m%d).tar.gz /qdrant/storage

# Rotate backups (keep 30 days)
0 4 * * * find /backups -name "*.gz" -mtime +30 -delete
```

### Volume Restore

```bash
# Restore PostgreSQL
docker exec -i <postgres-container> \
  psql -U unicore < /backups/postgres/unicore-20260301.sql

# Restore ERP database
docker exec -i <postgres-container> \
  psql -U unicore unicore_erp < /backups/postgres/unicore-erp-20260301.sql
```

## Monitoring

### Container Health

```bash
# Check all containers
docker compose --profile apps --profile workflows ps

# Check resource usage
docker stats

# Check service health endpoints
curl http://localhost:4000/health
curl http://localhost:4100/health
curl http://localhost:4200/health
curl http://localhost:4300/health
```

### Log Aggregation

For centralized logging, configure Docker to use a syslog or JSON file driver:

```bash
# /etc/docker/daemon.json — limit log file sizes
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  }
}
```

For full log aggregation, integrate with a local ELK stack (Elasticsearch + Logstash + Kibana) or Grafana Loki.

## Security Hardening

### OS Hardening

```bash
# Disable root SSH login
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Enable automatic security updates
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

### Docker Security

```bash
# Run containers as non-root (set in Dockerfile)
# Use read-only filesystems where possible
# Limit container capabilities

# Scan images for vulnerabilities
docker scout cves unicore-api-gateway:latest
```

### Secrets Management

- Never commit secrets to version control
- Use Docker secrets or environment files with restricted permissions:

```bash
chmod 600 .env
chown root:root .env
```

For Enterprise deployments, use HashiCorp Vault for secret injection:

```bash
# Vault agent sidecar injects secrets at container startup
```

## Updates and Maintenance

### Updating UniCore

```bash
# From the root workspace directory
# Pull latest code
git pull origin main
git submodule update --remote --merge

# Rebuild and redeploy
docker compose --profile apps --profile workflows build --no-cache
docker compose --profile apps --profile workflows up -d

# Re-push schemas after updates
docker compose --profile apps exec unicore-api-gateway npx prisma db push --accept-data-loss
docker compose --profile apps exec unicore-erp npx prisma db push --accept-data-loss
```

### Maintenance Window

Recommended maintenance procedure:

1. Notify users (via email or in-app banner)
2. Enable maintenance mode (set in Settings > System)
3. Take a full backup
4. Apply updates
5. Verify health checks pass
6. Disable maintenance mode

## Related Documentation

- [Docker Compose deployment](./docker.md) — detailed deployment steps
- [Kubernetes deployment](./kubernetes.md) — for HA deployments
- [Enterprise Edition](../editions/enterprise.md) — HA, compliance, and SLA details
- [Security best practices](../security/best-practices.md)

---

© 2026 BeMind Technology
