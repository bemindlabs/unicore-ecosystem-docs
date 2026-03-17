# Security Best Practices

This guide covers hardening a UniCore deployment for production. Work through each section before exposing the platform to the internet or handling real customer data.

---

## Production Deployment Checklist

Use this checklist before going live. Items marked **[CRITICAL]** are non-negotiable.

### Authentication & Secrets

- [ ] **[CRITICAL]** Change all default credentials (see [Default Credentials to Rotate](#default-credentials-to-rotate))
- [ ] **[CRITICAL]** Replace the default `JWT_SECRET` / RSA keys with freshly generated secrets
- [ ] **[CRITICAL]** Replace the default `X-Bootstrap-Secret` (`unicore-bootstrap-secret-local`) with a random 32-byte value and disable the provision-admin endpoint afterward
- [ ] Rotate Redis password from the default
- [ ] Rotate the PostgreSQL password from the default
- [ ] Set `NODE_ENV=production` on all services
- [ ] Disable the wizard endpoint after initial setup (`/api/v1/settings/wizard-status`)

### Network

- [ ] **[CRITICAL]** TLS terminated at Cloudflare or Nginx Proxy Manager — never serve plain HTTP
- [ ] Firewall: only expose ports 80/443 publicly; keep all service ports (3000, 4000–4600, etc.) on internal networks only
- [ ] Bind PostgreSQL, Redis, and Qdrant to `127.0.0.1` or the internal Docker bridge — never `0.0.0.0`
- [ ] Enable Cloudflare WAF and DDoS protection for the production hostname

### Containers

- [ ] All application containers run as non-root users (UID ≥ 1000)
- [ ] Container images pinned to specific digest tags (not `:latest`)
- [ ] Read-only root filesystem where possible (`read_only: true` in compose)
- [ ] No privileged containers (`privileged: false`)
- [ ] Resource limits set (`mem_limit`, `cpu_quota`) to prevent resource exhaustion

### Database

- [ ] PostgreSQL not exposed outside Docker network
- [ ] `unicore` DB user has minimum required privileges (not `SUPERUSER`)
- [ ] Database backups configured and tested
- [ ] Audit extension enabled (`pgaudit`) for compliance deployments

---

## Default Credentials to Rotate

| Service | Default Credential | Environment Variable |
|---------|-------------------|---------------------|
| Admin account | `admin@unicore.dev` / `admin123` | N/A — change via dashboard |
| PostgreSQL | `unicore` / `unicore` | `DATABASE_URL` |
| Redis | (no auth by default) | `REDIS_PASSWORD` |
| Bootstrap secret | `unicore-bootstrap-secret-local` | `BOOTSTRAP_SECRET` |
| NPM admin | `info@bemind.tech` / `unicore123` | Nginx Proxy Manager UI |

---

## Secret Management

### Never Hardcode Secrets

Secrets must never appear in:
- Source code or committed `.env` files
- Docker image layers (`ENV` instructions with real values)
- Container environment variables in plain `docker-compose.yml` checked into git

### Recommended Approaches

**Docker Secrets (Swarm / Compose v3.1+)**

```yaml
services:
  unicore-api-gateway:
    secrets:
      - db_password
      - jwt_private_key
    environment:
      DATABASE_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true
  jwt_private_key:
    file: ./secrets/jwt_private_key.pem  # keep outside repo
```

**HashiCorp Vault**

For high-security deployments, inject secrets at runtime via Vault agent:

```bash
vault kv put secret/unicore/prod \
  db_password="..." \
  jwt_secret="..."
```

**Environment File Outside the Repo**

At minimum, keep `.env` files outside version control and load them explicitly:

```bash
# .gitignore
.env
.env.*
!.env.example
```

Never commit actual values — only commit `.env.example` with placeholder descriptions.

### Generating Strong Secrets

```bash
# 32-byte hex secret (JWT, bootstrap, etc.)
openssl rand -hex 32

# RSA keypair for JWT (RS256)
openssl genrsa -out jwt_private.pem 2048
openssl rsa -in jwt_private.pem -pubout -out jwt_public.pem

# Random Redis password
openssl rand -base64 24
```

---

## Network Security

### Cloudflare Configuration

UniCore runs behind Cloudflare. Recommended settings:

| Setting | Value |
|---------|-------|
| SSL/TLS mode | **Full (strict)** |
| Always Use HTTPS | Enabled |
| Min TLS Version | 1.2 |
| WAF | Enabled (Managed Rules + OWASP ruleset) |
| Bot Fight Mode | Enabled |
| Rate Limiting | `/auth/login`: 5 req/min per IP |
| IP Access Rules | Block known malicious ASNs |

### Nginx Configuration Hardening

The Nginx config lives at `unicore/nginx/default.conf`. Add these headers in the server block:

```nginx
# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# Disable server signature
server_tokens off;

# Limit request size (prevent large payload attacks)
client_max_body_size 10m;
```

### Internal Firewall Rules

```
# Allow only
:80   <- public internet (Cloudflare IPs only via ufw)
:443  <- public internet (Cloudflare IPs only)
:81   <- NPM admin — restrict to management VPN/IP only

# Block externally
:3000  (dashboard)
:4000  (API gateway)
:4100-4600  (microservices)
:5433  (PostgreSQL)
:6380  (Redis)
:6333  (Qdrant)
:9092  (Kafka)
:18789-18790  (OpenClaw)
```

---

## Database Security

### PostgreSQL

```sql
-- Revoke dangerous defaults
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- The application user should not have superuser privileges
ALTER USER unicore NOSUPERUSER NOCREATEDB NOCREATEROLE;

-- Create a read-only reporting user (for Finance reports, analytics)
CREATE USER unicore_readonly WITH PASSWORD '...';
GRANT CONNECT ON DATABASE unicore TO unicore_readonly;
GRANT USAGE ON SCHEMA public TO unicore_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO unicore_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO unicore_readonly;
```

### PostgreSQL Connection Encryption

Enable `sslmode=require` in the `DATABASE_URL`:

```env
DATABASE_URL="postgresql://unicore:password@localhost:5433/unicore?sslmode=require"
```

### Backup Encryption

```bash
# Encrypted backup
pg_dump -U unicore unicore | \
  gpg --symmetric --cipher-algo AES256 --output unicore_$(date +%Y%m%d).sql.gpg

# Restore
gpg --decrypt unicore_20260317.sql.gpg | psql -U unicore unicore
```

---

## Redis Security

Redis has no authentication by default. Enable it before production:

```conf
# redis.conf
requirepass your-strong-redis-password
bind 127.0.0.1
protected-mode yes

# Disable dangerous commands
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command DEBUG ""
```

In docker-compose, pass the password via an environment variable or secret:

```yaml
unicore-redis:
  image: redis:7-alpine
  command: redis-server --requirepass ${REDIS_PASSWORD}
```

Update `REDIS_URL` in all services:

```env
REDIS_URL=redis://:your-strong-redis-password@localhost:6380
```

---

## Container Security

### Non-Root User

All UniCore service Dockerfiles should run as a non-root user:

```dockerfile
# Example NestJS service
FROM node:20-alpine AS runner
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nestjs
USER nestjs
```

### Read-Only Filesystem

```yaml
services:
  unicore-api-gateway:
    read_only: true
    tmpfs:
      - /tmp
      - /app/node_modules/.cache
```

### Resource Limits

```yaml
services:
  unicore-api-gateway:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          memory: 256M
```

### Image Scanning

Scan images for known CVEs before deploying:

```bash
# Using Trivy
trivy image unicores-unicore-api-gateway:latest

# Using Docker Scout
docker scout cves unicores-unicore-api-gateway:latest
```

Set up CI to fail builds on `HIGH` or `CRITICAL` severity CVEs.

---

## Dependency Scanning

### pnpm Audit

Run regularly and in CI:

```bash
# Audit all workspaces
pnpm audit --recursive

# Fail CI on high severity
pnpm audit --recursive --audit-level high
```

### Automated Updates

Use Dependabot or Renovate to receive dependency update PRs automatically. Example Renovate config (`.renovaterc.json`):

```json
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    },
    {
      "matchPackagePatterns": ["*"],
      "matchUpdateTypes": ["major"],
      "labels": ["dependencies", "major"]
    }
  ]
}
```

---

## OWASP Top 10 Mitigations

| Vulnerability | Mitigation in UniCore |
|--------------|----------------------|
| **A01 Broken Access Control** | JWT + RolesGuard on every route; global guard registration; no anonymous endpoints except `/auth/login` and wizard |
| **A02 Cryptographic Failures** | bcrypt for passwords (cost factor ≥ 12); RS256 JWT signing; TLS in transit; AES-256-GCM for PII at rest (Enterprise) |
| **A03 Injection** | Prisma ORM with parameterised queries (no raw SQL in application code); NestJS `ValidationPipe` with `whitelist: true` on all DTOs |
| **A04 Insecure Design** | Separate database for license server; tenant isolation in Enterprise; principle of least privilege for DB users |
| **A05 Security Misconfiguration** | `.env.example` with documented defaults; checklist in this document; `NODE_ENV=production` disables stack traces |
| **A06 Vulnerable Components** | `pnpm audit` in CI; Trivy image scanning; Dependabot PRs |
| **A07 Auth Failures** | Refresh token rotation; Redis token blacklist for logout; rate limiting on `/auth/login` via Cloudflare |
| **A08 Software Integrity Failures** | Pinned Docker image digests; signed commits enforced on main branch; provenance in CI |
| **A09 Logging & Monitoring Failures** | Audit log on every data mutation; Pro audit package with SIEM export; Kafka event stream for anomaly detection |
| **A10 Server-Side Request Forgery** | Outbound HTTP from AI Engine uses allowlist of permitted domains; no user-supplied URLs passed directly to `fetch()` |

### NestJS ValidationPipe Configuration

Ensure the global validation pipe rejects unknown fields:

```typescript
// main.ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,        // strip unknown properties
    forbidNonWhitelisted: true, // reject requests with unknown properties
    transform: true,
    disableErrorMessages: process.env.NODE_ENV === 'production',
  }),
);
```

### Rate Limiting

Apply NestJS Throttler for additional protection (beyond Cloudflare):

```typescript
ThrottlerModule.forRoot([{
  name: 'default',
  ttl: 60_000,  // 1 minute
  limit: 60,    // 60 requests per minute per IP
}]),
```

For auth endpoints, apply a stricter limit:

```typescript
@Throttle({ default: { limit: 5, ttl: 60_000 } })
@Post('login')
login() { ... }
```

---

## Incident Response

If you suspect a security incident:

1. **Contain**: Rotate the compromised credential immediately. Invalidate all active sessions by flushing the Redis refresh token namespace: `redis-cli -n 0 --scan --pattern "refresh:*" | xargs redis-cli DEL`
2. **Investigate**: Query the audit log for the affected user/IP in the relevant time window.
3. **Notify**: If personal data was accessed, GDPR requires notification to the supervisory authority within 72 hours.
4. **Remediate**: Patch the root cause. Re-run the deployment checklist before bringing the system back online.
5. **Report**: File a post-incident report and update runbooks.

---

## Related

- [Authentication](authentication.md)
- [Authorization & RBAC](authorization.md)
- [Data Privacy](data-privacy.md)
- [Deployment Guide](../deployment/docker.md)
