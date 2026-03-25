# Data Privacy

Updated: 2026-03-22

This document describes how UniCore handles personal data, outlines compliance obligations, and explains the controls available to operators for meeting GDPR and similar regulatory requirements.

> **Edition note**: Full GDPR tooling (data subject request automation, consent management UI, audit export) is an **Enterprise** feature. Community and Pro editions provide the underlying data architecture and manual controls described below.

---

## Regulatory Scope

| Regulation | Coverage | Available From |
|------------|---------|---------------|
| GDPR (EU 2016/679) | EU/EEA data subjects | Enterprise |
| PDPA (Thailand) | Thai data subjects | Enterprise |
| CCPA (California) | California consumers | Enterprise |
| General privacy hygiene | All deployments | Community |

If your deployment processes personal data of individuals in covered jurisdictions, you must configure the appropriate controls before going live.

---

## Data Classification

UniCore classifies stored data into three tiers:

| Tier | Examples | Retention Default | Encryption at Rest |
|------|---------|------------------|--------------------|
| **PII** | Name, email, phone, address | Configurable (default 3 years) | Required |
| **Sensitive PII** | Payment data, identity documents | As legally required | Required + field-level |
| **Operational** | Logs, events, metrics | 90 days | Recommended |

---

## PII Handling

### Where PII Lives

| Service | Model / Table | PII Fields |
|---------|--------------|-----------|
| API Gateway | `User` | `email`, `name` |
| ERP | `Contact` | `email`, `phone`, `address`, `company` |
| ERP | `Invoice` | billing address, contact reference |
| RAG | Vector embeddings | chunks may contain PII from uploaded documents |
| OpenClaw | `ChatHistory` | message content |
| Redis | Sessions, refresh tokens | user ID (indirect reference only) |

### Field-Level Encryption (Enterprise)

Sensitive PII fields are encrypted at the application layer before being written to PostgreSQL:

- **Algorithm**: AES-256-GCM
- **Key management**: Master key stored in a secrets vault (HashiCorp Vault or AWS KMS); never in environment variables
- **Key rotation**: Supported via re-encryption job; zero downtime

### Data Minimisation

- Collect only fields required for the stated business purpose.
- The ERP contact model has optional fields — do not populate them unless necessary.
- AI/RAG pipelines: strip PII from documents before ingestion where possible using the built-in PII scrubber (Enterprise).

---

## Encryption

### In Transit

All traffic is encrypted with TLS 1.2+ enforced at the Cloudflare edge and the Nginx Proxy Manager layer. Internal service-to-service traffic within the Docker network uses plaintext by default; enable mTLS for production deployments handling regulated data:

```yaml
# In docker-compose.yml service definition
environment:
  MTLS_ENABLED: "true"
  MTLS_CA_CERT: /run/secrets/internal_ca
  MTLS_CERT: /run/secrets/service_cert
  MTLS_KEY: /run/secrets/service_key
```

### At Rest

| Storage | Encryption |
|---------|-----------|
| PostgreSQL | Filesystem encryption (LUKS / cloud provider disk encryption) |
| Redis | Filesystem encryption (in-memory data is not encrypted by Redis itself) |
| Qdrant vectors | Filesystem encryption |
| Docker volumes | Encrypt the host volume or use encrypted block storage |
| Backups | AES-256 encrypted archives; keys stored separately from backup data |

---

## GDPR Compliance (Enterprise)

### Data Subject Rights

Enterprise Edition provides an automated Data Subject Request (DSR) portal accessible at `/admin/privacy/requests`.

| Right | Mechanism | SLA |
|-------|-----------|-----|
| **Right of Access** (Art. 15) | Export all data linked to a user ID / email | 30 days |
| **Right to Erasure** (Art. 17) | Anonymise or delete PII across all services | 30 days |
| **Right to Portability** (Art. 20) | Export data in JSON or CSV | 30 days |
| **Right to Rectification** (Art. 16) | Update incorrect data fields | Immediate |
| **Right to Object** (Art. 21) | Opt out of processing for specified purposes | Configurable |

### Submitting a DSR via API

```bash
POST /api/v1/privacy/dsr
Authorization: Bearer <admin_token>
Content-Type: application/json

{
  "type": "erasure",
  "subjectEmail": "user@example.com",
  "requestedBy": "DPO",
  "notes": "User submitted via web form on 2026-03-01"
}
```

The system creates a DSR record, enqueues a Kafka event (`privacy.dsr.requested`), and each service processes the erasure asynchronously. A completion report is emailed to the configured Data Protection Officer address.

### Erasure Implementation

"Erasure" in UniCore means **anonymisation** rather than hard deletion for records required by law (e.g. invoices with legal retention obligations):

- PII fields replaced with `[REDACTED]` or a deterministic anonymised token.
- Foreign key references preserved to maintain referential integrity.
- ChatHistory and vector embeddings containing PII are hard-deleted.

---

## Consent Management

### Consent Records

Enterprise Edition tracks processing consent per data subject:

```prisma
model ConsentRecord {
  id          String   @id @default(cuid())
  subjectId   String
  purpose     String   // e.g. "marketing_email", "analytics"
  granted     Boolean
  ipAddress   String?
  userAgent   String?
  grantedAt   DateTime
  revokedAt   DateTime?
}
```

### Consent Collection

Embed the consent widget in your customer-facing flows:

```tsx
import { ConsentBanner } from '@bemindlabs/unicore-ui';

<ConsentBanner
  purposes={['analytics', 'marketing_email']}
  onAccept={(purposes) => recordConsent(purposes)}
  onDecline={() => recordConsent([])}
/>
```

### Respecting Consent

Before sending marketing messages via the Channels service, verify consent:

```typescript
const consented = await privacyService.hasConsent(contactId, 'marketing_email');
if (!consented) return; // skip silently — do not send
```

---

## Data Retention Policies

Configure retention per data category in **Settings → Privacy → Retention Policies**.

| Category | Default Retention | Legal Minimum | Action on Expiry |
|----------|------------------|--------------|-----------------|
| User accounts | Until deletion request | — | Anonymise |
| CRM contacts | 3 years after last interaction | — | Anonymise |
| Invoices / financial records | 7 years | Tax law (varies by country) | Archive (do not delete) |
| Chat history | 1 year | — | Delete |
| Audit logs | 2 years | SOC 2: 1 year | Archive |
| Session data (Redis) | 7 days | — | Auto-expire (TTL) |
| Backups | 30 days | — | Auto-purge |

Retention jobs run daily via a Kafka-triggered workflow. Configure the schedule in `unicore/services/workflow/src/jobs/retention.job.ts`.

---

## Audit Logging

All data access and modifications are written to the `AuditLog` table. The Pro `@bemindlabs/unicore-audit` package extends this with:

- Who accessed what data and when
- Export to SIEM (Splunk, Datadog, ELK)
- Tamper-evident log hashing

Audit logs must themselves be protected: only `OWNER` role can query them, and they are read-only (no update/delete endpoints).

---

## Data Processing Agreements

For deployments processing EU personal data:

1. Sign a Data Processing Agreement (DPA) with BeMind Technology if using the hosted platform.
2. For self-hosted deployments, your organisation is the data controller and processor — you are responsible for DPAs with any sub-processors (e.g. cloud provider, email delivery).
3. Maintain a Record of Processing Activities (RoPA) — the audit log export can seed this.

---

## Related

- [Authentication](authentication.md)
- [Authorization & RBAC](authorization.md)
- [Best Practices & Hardening](best-practices.md)
