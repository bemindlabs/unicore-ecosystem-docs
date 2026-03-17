# Authentication

UniCore uses [Passport.js](https://www.passportjs.org/) with two built-in strategies — **JWT** and **local** (username/password) — provided by the `api-gateway` service. Pro and Enterprise editions add SSO via the `unicore-pro/packages/sso` package.

---

## Strategies

### Local Strategy (username + password)

The local strategy handles initial credential validation against the PostgreSQL `User` table.

1. Client `POST /auth/login` with `{ email, password }`.
2. Passport `LocalStrategy` locates the user by email and verifies the bcrypt hash.
3. On success, the gateway generates an access token + refresh token pair (see [Token Structure](#token-structure)).
4. Both tokens are returned in the response body; the refresh token is also stored in Redis.

### JWT Strategy

Every subsequent request must carry a valid access token:

```
Authorization: Bearer <access_token>
```

The `JwtStrategy` validates the signature, expiry, and the `sub` (user ID) claim. If the token is expired, clients must use the [refresh flow](#token-refresh).

---

## Login Flow

```
Client                API Gateway            PostgreSQL / Redis
  │                        │                       │
  │  POST /auth/login       │                       │
  │ ──────────────────────► │                       │
  │                        │  SELECT user WHERE     │
  │                        │  email = ?             │
  │                        │ ──────────────────────►│
  │                        │ ◄──────────────────────│
  │                        │  bcrypt.compare()      │
  │                        │  generate tokens       │
  │                        │  SET refresh:<userId>  │
  │                        │ ──────────────────────►│
  │ ◄──────────────────────│                       │
  │ { accessToken,          │                       │
  │   refreshToken, user }  │                       │
```

**Endpoint summary**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/auth/login` | Authenticate with email + password |
| `POST` | `/auth/refresh` | Exchange refresh token for new access token |
| `POST` | `/auth/logout` | Revoke refresh token (removes Redis key) |
| `GET` | `/auth/me` | Return authenticated user profile |
| `POST` | `/auth/provision-admin` | Bootstrap initial admin (requires `X-Bootstrap-Secret` header) |

---

## Token Structure

### Access Token (JWT)

Access tokens are short-lived (default **15 minutes**) RS256-signed JWTs.

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "usr_01HXYZ...",
    "email": "user@example.com",
    "role": "OPERATOR",
    "tenantId": "ten_01HABC...",
    "iat": 1710000000,
    "exp": 1710000900
  }
}
```

| Claim | Type | Description |
|-------|------|-------------|
| `sub` | string | User ID (CUID) |
| `email` | string | User email address |
| `role` | string | One of `OWNER`, `OPERATOR`, `MARKETER`, `FINANCE`, `VIEWER` |
| `tenantId` | string | Tenant ID (Enterprise only; omitted in Community/Pro) |
| `iat` | number | Issued-at timestamp (Unix seconds) |
| `exp` | number | Expiry timestamp (Unix seconds) |

### Refresh Token

Refresh tokens are opaque random strings (256-bit hex), stored in Redis with the key pattern `refresh:<userId>`. They are long-lived (default **7 days**) and invalidated on logout or password change.

---

## Token Refresh

```
POST /auth/refresh
Content-Type: application/json

{ "refreshToken": "<refresh_token>" }
```

Flow:

1. Gateway validates the refresh token exists in Redis (`GET refresh:<userId>`).
2. Checks that the stored user is still active.
3. Issues a new access token (and optionally rotates the refresh token).
4. Returns `{ accessToken, refreshToken }`.

If the refresh token is missing or expired, the gateway returns `401 Unauthorized` and the client must re-authenticate.

---

## Session Management (Redis)

Redis stores refresh tokens using the following schema:

| Key | Value | TTL |
|-----|-------|-----|
| `refresh:<userId>` | `<refresh_token_hash>` | 7 days |
| `session:<sessionId>` | serialized session object | configurable |
| `blacklist:<jti>` | `1` | until access token expiry |

**Token revocation**: On logout or a forced sign-out (e.g. password reset, admin action), the access token's `jti` is inserted into the Redis blacklist. The `JwtStrategy` checks the blacklist on every request.

> **Pro tip**: Configure `REDIS_TTL_REFRESH_TOKEN` (seconds) and `REDIS_TTL_ACCESS_TOKEN` in your `.env` to tune lifetimes for your security requirements.

---

## SSO Integration (Pro / Enterprise)

The `unicore-pro/packages/sso` package extends authentication with federated identity providers.

### Supported Protocols

| Protocol | Editions | Use Case |
|----------|----------|---------|
| OAuth 2.0 / OIDC | Pro, Enterprise | Google Workspace, Microsoft Entra, GitHub |
| SAML 2.0 | Enterprise | Enterprise IdPs (Okta, OneLogin, Azure AD) |
| LDAP / Active Directory | Enterprise | On-premise directory integration |

### OIDC Login Flow

```
Client             API Gateway            Identity Provider
  │                    │                        │
  │  GET /auth/sso/:provider  │                 │
  │ ──────────────────►│                        │
  │ ◄── redirect ──────│                        │
  │                    │                        │
  │ ──── user logs in on IdP ─────────────────► │
  │ ◄──────────────────────── callback ─────────│
  │                    │                        │
  │  GET /auth/sso/callback?code=...            │
  │ ──────────────────►│                        │
  │                    │  exchange code         │
  │                    │ ──────────────────────►│
  │                    │ ◄── id_token ──────────│
  │                    │  upsert user           │
  │                    │  generate tokens       │
  │ ◄── { accessToken, refreshToken } ─────────│
```

### SSO Configuration

SSO providers are configured per-tenant in the dashboard under **Settings → SSO**.

```env
# Example: Google OIDC
SSO_GOOGLE_CLIENT_ID=your-google-client-id
SSO_GOOGLE_CLIENT_SECRET=your-google-client-secret
SSO_GOOGLE_CALLBACK_URL=https://yourdomain.com/auth/sso/google/callback

# SAML (Enterprise)
SAML_ENTRY_POINT=https://idp.example.com/sso/saml
SAML_ISSUER=https://yourdomain.com
SAML_CERT=/run/secrets/saml_idp_cert
```

> Never commit SSO secrets to source control. Use Docker secrets or a vault solution — see [Secret Management](best-practices.md#secret-management).

---

## API Key Authentication

Services and integrations can authenticate using long-lived API keys instead of user sessions.

### Creating an API Key

```
POST /api/v1/api-keys
Authorization: Bearer <access_token>

{
  "name": "My Integration",
  "scopes": ["erp:read", "crm:write"],
  "expiresAt": "2027-01-01T00:00:00Z"
}
```

Response:

```json
{
  "id": "key_01HXYZ...",
  "key": "unc_live_abc123...",
  "name": "My Integration",
  "scopes": ["erp:read", "crm:write"],
  "createdAt": "2026-03-17T00:00:00Z"
}
```

The raw key is only shown once. Store it securely.

### Using an API Key

```
GET /api/v1/contacts
X-API-Key: unc_live_abc123...
```

API keys bypass the refresh flow but are bound to the creating user's role. Revoke them at **Settings → API Keys**.

---

## Related

- [Authorization & RBAC](authorization.md)
- [Best Practices & Hardening](best-practices.md)
- [Pro SSO package source](../../../unicore-pro/packages/sso/)
