# UniCore Enterprise Edition

The Enterprise Edition is designed for organizations requiring multi-tenancy, compliance certifications, high-availability clustering, enterprise identity federation, and dedicated SLA support.

## Overview

| Attribute | Value |
|-----------|-------|
| **License** | Proprietary (contact sales) |
| **Repository** | Private (available with Enterprise license) |
| **Price** | Custom pricing (annual contract) |
| **Hosting** | Self-hosted or managed cloud |
| **Edition flag** | `UNICORE_EDITION: enterprise` |

Enterprise Edition includes everything in Pro Edition plus the features documented in this guide.

## Multi-Tenancy

Enterprise Edition supports true multi-tenancy — a single deployment serves multiple isolated organizations (tenants).

### Tenant Isolation

- Each tenant has its own isolated database schema or dedicated database instance
- Data is never shared across tenants at the application or database layer
- Tenant-aware routing via API Gateway middleware
- Per-tenant feature flag overrides
- Per-tenant branding (via `@unicore-pro/branding`)

### Tenant Management

- Superadmin console for provisioning and managing tenants
- Tenant-level usage quotas (agents, workflows, RAG storage, API rate limits)
- Tenant onboarding API for programmatic provisioning
- Tenant suspension and deletion with data retention policies

## High Availability (HA)

### Architecture

```
Load Balancer (HAProxy / AWS ALB / GCP LB)
  ├── API Gateway   (2+ replicas)
  ├── Dashboard     (2+ replicas)
  ├── ERP           (2+ replicas)
  ├── AI Engine     (2+ replicas)
  └── RAG           (2+ replicas)

PostgreSQL        → Primary + Read Replicas (Patroni / RDS / Cloud SQL)
Redis             → Sentinel or Redis Cluster
Qdrant            → Distributed cluster mode
Kafka             → Multi-broker cluster (3+ brokers)
```

### Session Affinity

All stateful session data is stored in Redis, enabling stateless horizontal scaling of API Gateway and Dashboard replicas.

### Health Checks

All services expose `/health` endpoints conforming to NestJS Terminus health check format. Load balancers route traffic only to healthy instances.

### Zero-Downtime Deployments

Rolling deployments are supported via Kubernetes (see [kubernetes deployment guide](../deployment/kubernetes.md)) or Docker Swarm with health check gates.

## Compliance

### GDPR

- Data Subject Access Request (DSAR) API — export all data for a user
- Right to Erasure — permanent deletion of user data across all services
- Data Processing Agreements (DPA) available
- Data residency configuration (choose region for data storage)
- Configurable data retention policies per data type

### SOC 2 Type II

- Access control logs (via `@unicore-pro/audit`)
- Change management audit trail
- Incident response documentation
- Penetration test reports available under NDA

### ISO 27001

- Information security management controls
- Risk assessment documentation
- Business continuity plan

### HIPAA (optional add-on)

- BAA (Business Associate Agreement) available
- PHI data encryption at rest and in transit
- Audit log immutability

## Enterprise SSO

Enterprise Edition extends Pro SSO with additional identity providers and protocols:

| Protocol | Providers |
|----------|----------|
| SAML 2.0 | Okta, Microsoft Entra ID (Azure AD), PingFederate, ADFS |
| OIDC | Google Workspace, Auth0, Keycloak, Ping Identity |
| LDAP/AD | Microsoft Active Directory, OpenLDAP |
| SCIM 2.0 | Automated user provisioning and deprovisioning |

### SCIM Provisioning

SCIM 2.0 support allows enterprise identity platforms (Okta, Entra ID) to automatically:
- Create user accounts on hire
- Update user attributes on profile changes
- Deactivate accounts on termination

## Enterprise API Gateway

In addition to the standard API Gateway, Enterprise Edition includes:

- **Rate limiting** — per-tenant and per-user rate limits with configurable tiers
- **API key management** — long-lived API keys with scope-based permissions
- **IP allowlisting** — restrict API access to known IP ranges
- **Webhook signing** — HMAC-SHA256 signatures on all outbound webhooks
- **Request logging** — full request/response logging to configurable sinks (S3, GCS, SIEM)

## Advanced Analytics

- Real-time business dashboards with configurable KPIs
- Cross-tenant analytics for superadmins
- Export to BI tools (Power BI, Tableau, Looker) via JDBC/ODBC connectors
- Scheduled report delivery via email

## Disaster Recovery

| Metric | Target |
|--------|--------|
| RTO (Recovery Time Objective) | < 1 hour |
| RPO (Recovery Point Objective) | < 15 minutes |
| Backup frequency | Continuous (WAL streaming) + daily snapshots |
| Backup retention | 30 days |

Backup and restore procedures are documented in the [on-premise deployment guide](../deployment/on-premise.md).

## Support SLA

| Tier | Response Time | Channels |
|------|--------------|----------|
| Critical (P1) | 1 hour | Phone, Slack, email |
| High (P2) | 4 hours | Slack, email |
| Medium (P3) | 1 business day | Email, portal |
| Low (P4) | 3 business days | Portal |

Enterprise customers receive a dedicated Customer Success Manager (CSM) and access to the private support Slack workspace.

## Deployment

Enterprise Edition supports multiple deployment targets:

- [Docker Compose](../deployment/docker.md) (single-node)
- [Kubernetes / Helm](../deployment/kubernetes.md) (recommended for HA)
- [AWS](../deployment/cloud/aws.md)
- [GCP](../deployment/cloud/gcp.md)
- [Azure](../deployment/cloud/azure.md)
- [On-premise](../deployment/on-premise.md)

## Licensing

Enterprise licenses are issued as site licenses (unlimited machines within a single organization). Contact [info@bemind.tech](mailto:info@bemind.tech) for pricing and a proof-of-concept deployment.

---

© 2026 BeMind Technology — Proprietary License
