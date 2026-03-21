# REST API Reference

UniCore exposes REST endpoints across several services, all routed through Nginx at `https://unicore.bemind.tech`. Direct service ports are listed for local development.

## Base URLs

| Service | Local Base URL | Nginx Path |
|---------|---------------|------------|
| API Gateway | `http://localhost:4000` | `/api/`, `/auth/`, `/webhooks/` |
| ERP | `http://localhost:4100/api/v1` | `/api/v1/` (proxied) |
| AI Engine | `http://localhost:4200` | internal |
| RAG | `http://localhost:4300` | internal |
| Bootstrap | `http://localhost:4500` | `/api/v1/` (wizard only) |

## Authentication

All protected endpoints require a JWT Bearer token in the `Authorization` header.

```
Authorization: Bearer <accessToken>
```

Tokens are obtained from `POST /auth/login`. Access tokens expire after 15 minutes; use `POST /auth/refresh` to renew.

---

## Auth Endpoints

### POST /auth/register

Register a new user account.

**Request**

```json
{
  "email": "user@example.com",
  "name": "Jane Doe",
  "password": "Password1",
  "confirmPassword": "Password1"
}
```

Password rules: minimum 8 characters, at least one uppercase letter, one lowercase letter, and one digit.

**Response** `201 Created`

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "a1b2c3d4...",
  "expiresIn": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "name": "Jane Doe",
    "role": "VIEWER"
  }
}
```

---

### POST /auth/login

Authenticate with email and password.

**Request**

```json
{
  "email": "user@example.com",
  "password": "YourPassword1"
}
```

**Response** `200 OK`

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "a1b2c3d4...",
  "expiresIn": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "name": "Admin",
    "role": "OWNER"
  }
}
```

---

### POST /auth/refresh

Exchange a refresh token for a new access token.

**Request**

```json
{
  "refreshToken": "a1b2c3d4..."
}
```

**Response** `200 OK`

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "e5f6g7h8...",
  "expiresIn": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "admin@unicore.dev",
    "name": "Admin",
    "role": "OWNER"
  }
}
```

---

### POST /auth/logout

Invalidate the current refresh token. Requires authentication.

**Request**

```json
{
  "refreshToken": "a1b2c3d4..."
}
```

**Response** `200 OK`

```json
{
  "message": "Logged out successfully"
}
```

---

### GET /auth/me

Return the authenticated user's profile. Requires authentication.

**Response** `200 OK`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "admin@unicore.dev",
  "name": "Admin",
  "role": "OWNER",
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

---

### POST /auth/provision-admin

Bootstrap the first admin user. Protected by a static `X-Bootstrap-Secret` header (set via `BOOTSTRAP_SECRET` env var). Does not require JWT authentication.

**Headers**

```
X-Bootstrap-Secret: unicore-bootstrap-secret-local
Content-Type: application/json
```

**Request**

```json
{
  "email": "admin@unicore.dev",
  "name": "Admin",
  "password": "admin123",
  "role": "OWNER"
}
```

Valid roles: `OWNER`, `OPERATOR`.

**Response** `201 Created`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "admin@unicore.dev",
  "name": "Admin",
  "role": "OWNER"
}
```

---

### GET /auth/profile

Return the authenticated user's profile. Requires authentication. Equivalent to `GET /auth/me`.

**Response** `200 OK`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "admin@unicore.dev",
  "name": "Admin",
  "role": "OWNER",
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

---

### PATCH /auth/profile

Update the authenticated user's profile. Requires authentication.

**Request**

```json
{
  "name": "Jane Smith",
  "email": "jane@unicore.dev"
}
```

Both fields are optional. `name` must be 2-100 characters. `email` must be a valid email address.

**Response** `200 OK` — updated user object.

---

### PATCH /auth/password

Change the authenticated user's password. Requires authentication.

**Request**

```json
{
  "currentPassword": "OldPassword1",
  "newPassword": "NewPassword2",
  "confirmPassword": "NewPassword2"
}
```

Password rules: minimum 8 characters (max 128), at least one uppercase letter, one lowercase letter, and one digit. `confirmPassword` must match `newPassword`.

**Response** `200 OK`

```json
{
  "message": "Password changed successfully"
}
```

---

## Notifications

Base path: `/api/v1/notifications`

All notification endpoints require authentication via `Authorization: Bearer <token>`.

### GET /api/v1/notifications

List notifications for the authenticated user.

**Query Parameters**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number |
| `limit` | number | 20 | Items per page |

**Response** `200 OK`

```json
{
  "notifications": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "title": "New order received",
      "body": "Order #1042 has been placed by Acme Corp.",
      "read": false,
      "createdAt": "2026-03-15T09:30:00.000Z"
    }
  ],
  "total": 12,
  "page": 1,
  "limit": 20
}
```

---

### GET /api/v1/notifications/unread-count

Return the number of unread notifications for the authenticated user.

**Response** `200 OK`

```json
{
  "count": 5
}
```

---

### PATCH /api/v1/notifications/read-all

Mark all notifications as read for the authenticated user.

**Response** `200 OK`

```json
{
  "ok": true
}
```

---

### PATCH /api/v1/notifications/:id/read

Mark a single notification as read.

**Response** `200 OK`

```json
{
  "ok": true
}
```

---

### DELETE /api/v1/notifications/:id

Delete a single notification.

**Response** `204 No Content`

---

## Settings Endpoints

Base path: `GET|PUT /api/v1/settings/:key`

### GET /api/v1/settings/wizard-status

Return the wizard completion state. Public — no authentication required.

**Response** `200 OK`

```json
{
  "completed": true,
  "completedAt": "2026-01-01T00:00:00.000Z"
}
```

---

### PUT /api/v1/settings/wizard-status

Save the wizard completion state. Public — no authentication required.

**Request**

```json
{
  "completed": true,
  "completedAt": "2026-01-01T00:00:00.000Z"
}
```

**Response** `200 OK` — returns the saved settings object.

---

### GET /api/v1/settings/:key

Retrieve a named settings object. Requires authentication.

**Response** `200 OK` — returns the stored JSON value, or `{}` if not set.

---

### PUT /api/v1/settings/:key

Store a named settings object. Requires authentication.

**Request** — any valid JSON object.

**Response** `200 OK` — returns the saved value.

---

### GET /api/v1/settings/team/count

Return the number of users in the workspace. Requires authentication.

**Response** `200 OK`

```json
{
  "count": 5
}
```

---

## ERP — Contacts

Base path: `/api/v1/contacts`

### POST /api/v1/contacts

Create a new contact.

**Request**

```json
{
  "name": "Acme Corp",
  "email": "contact@acme.com",
  "phone": "+1-555-0100",
  "type": "LEAD",
  "company": "Acme Corporation",
  "notes": "Interested in Pro plan"
}
```

**Response** `201 Created`

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "name": "Acme Corp",
  "email": "contact@acme.com",
  "phone": "+1-555-0100",
  "type": "LEAD",
  "leadScore": 0,
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

---

### GET /api/v1/contacts

List contacts with optional filtering.

**Query Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Filter by contact type (`LEAD`, `CUSTOMER`, `VENDOR`) |
| `search` | string | Full-text search on name/email |
| `page` | number | Page number (default: 1) |
| `limit` | number | Items per page (default: 20) |

**Response** `200 OK`

```json
{
  "data": [...],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

---

### GET /api/v1/contacts/leads/top

Return top leads by lead score.

**Query Parameters**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `minScore` | number | 50 | Minimum lead score threshold |
| `limit` | number | 20 | Maximum results to return |

---

### GET /api/v1/contacts/:id

Return a single contact by UUID.

**Response** `200 OK` — full contact object.

---

### PUT /api/v1/contacts/:id

Update a contact.

**Request** — partial contact fields.

**Response** `200 OK` — updated contact object.

---

### PUT /api/v1/contacts/:id/lead-score

Update a contact's lead score directly.

**Request**

```json
{
  "score": 85
}
```

---

### DELETE /api/v1/contacts/:id

Delete a contact.

**Response** `204 No Content`

---

## ERP — Orders

Base path: `/api/v1/orders`

### POST /api/v1/orders

Create a new order. Initial state: `DRAFT`.

**Request**

```json
{
  "contactId": "550e8400-...",
  "items": [
    {
      "productId": "550e8400-...",
      "quantity": 3,
      "unitPrice": 99.99
    }
  ],
  "notes": "Rush order"
}
```

**Response** `201 Created` — order object with `status: "DRAFT"`.

---

### GET /api/v1/orders

List orders.

**Query Parameters**: `status`, `contactId`, `page`, `limit`.

---

### GET /api/v1/orders/:id

Return a single order.

---

### PUT /api/v1/orders/:id

Update a draft order.

---

### POST /api/v1/orders/:id/confirm

Transition order from `DRAFT` to `CONFIRMED`. **Response** `200 OK`.

---

### POST /api/v1/orders/:id/process

Transition order from `CONFIRMED` to `PROCESSING`. **Response** `200 OK`.

---

### POST /api/v1/orders/:id/ship

Mark order as `SHIPPED`.

**Request**

```json
{
  "trackingNumber": "1Z999AA10123456784",
  "carrier": "UPS"
}
```

---

### POST /api/v1/orders/:id/fulfill

Mark order as `DELIVERED`.

**Request**

```json
{
  "deliveredAt": "2026-01-15T14:30:00.000Z"
}
```

---

### POST /api/v1/orders/:id/cancel

Cancel an order.

**Request**

```json
{
  "reason": "Customer requested cancellation"
}
```

---

### POST /api/v1/orders/:id/refund

Mark order as refunded. **Response** `200 OK`.

---

## ERP — Inventory

Base path: `/api/v1/inventory`

### POST /api/v1/inventory

Create a product/SKU.

**Request**

```json
{
  "name": "Wireless Headphones",
  "sku": "WH-001",
  "description": "Noise-cancelling headphones",
  "price": 149.99,
  "cost": 75.00,
  "stockQuantity": 100,
  "reorderPoint": 20,
  "warehouseId": "550e8400-..."
}
```

---

### GET /api/v1/inventory

List products.

**Query Parameters**: `search`, `warehouseId`, `page`, `limit`.

---

### GET /api/v1/inventory/low-stock

Return products at or below their reorder point.

---

### GET /api/v1/inventory/:id

Return a single product.

---

### PUT /api/v1/inventory/:id

Update product details.

---

### DELETE /api/v1/inventory/:id

Delete a product. **Response** `204 No Content`.

---

### POST /api/v1/inventory/:id/adjust-stock

Apply a manual stock adjustment.

**Request**

```json
{
  "delta": -5,
  "reason": "Damaged goods write-off"
}
```

---

### POST /api/v1/inventory/:id/restock

Record incoming stock.

**Request**

```json
{
  "quantity": 50,
  "unitCost": 74.00,
  "supplier": "Supplier Ltd"
}
```

---

### GET /api/v1/inventory/:id/movements

Return all stock movement history for a product.

---

## ERP — Invoices

Base path: `/api/v1/invoices`

### POST /api/v1/invoices

Create an invoice.

**Request**

```json
{
  "contactId": "550e8400-...",
  "orderId": "550e8400-...",
  "dueDate": "2026-02-01T00:00:00.000Z",
  "lineItems": [
    {
      "description": "Consulting hours",
      "quantity": 10,
      "unitPrice": 150.00
    }
  ],
  "notes": "Net 30"
}
```

---

### GET /api/v1/invoices

List invoices.

**Query Parameters**: `status`, `contactId`, `from`, `to`, `page`, `limit`.

---

### GET /api/v1/invoices/:id

Return a single invoice.

---

### PUT /api/v1/invoices/:id

Update an invoice (while still in `DRAFT` state).

---

### POST /api/v1/invoices/:id/send

Mark the invoice as `SENT`. **Response** `200 OK`.

---

### POST /api/v1/invoices/:id/record-payment

Record a payment against the invoice.

**Request**

```json
{
  "amount": 1500.00,
  "method": "BANK_TRANSFER",
  "reference": "TXN-20260115",
  "paidAt": "2026-01-15T10:00:00.000Z"
}
```

---

### POST /api/v1/invoices/:id/cancel

Cancel an invoice. **Response** `200 OK`.

---

### POST /api/v1/invoices/mark-overdue

Batch-update all past-due unpaid invoices to `OVERDUE` status. **Response** `200 OK`.

---

### DELETE /api/v1/invoices/:id

Delete a draft invoice. **Response** `204 No Content`.

---

## ERP — Expenses

Base path: `/api/v1/expenses`

### POST /api/v1/expenses

Submit an expense.

**Request**

```json
{
  "title": "Team lunch",
  "amount": 245.00,
  "currency": "USD",
  "category": "MEALS",
  "date": "2026-01-10T00:00:00.000Z",
  "notes": "Q1 all-hands"
}
```

---

### GET /api/v1/expenses

List expenses.

**Query Parameters**: `status`, `category`, `from`, `to`, `page`, `limit`.

---

### GET /api/v1/expenses/:id

Return a single expense.

---

### PUT /api/v1/expenses/:id

Update an expense (while in `PENDING` state).

---

### POST /api/v1/expenses/:id/approve

Approve a pending expense.

**Request**

```json
{
  "notes": "Approved for Q1 budget"
}
```

---

### POST /api/v1/expenses/:id/reject

Reject an expense.

**Request**

```json
{
  "reason": "Exceeds category budget"
}
```

---

### DELETE /api/v1/expenses/:id

Delete an expense. **Response** `204 No Content`.

---

## ERP — Reports

Base path: `/api/v1/reports`

All report endpoints require authentication. Responses vary by report type.

### GET /api/v1/reports/dashboard

Return KPI summary for the main dashboard (total revenue, open orders, outstanding invoices, expense total).

---

### GET /api/v1/reports/revenue

Return monthly revenue summary.

**Query Parameters**: `from` (ISO date), `to` (ISO date).

---

### GET /api/v1/reports/expenses/categories

Return expenses grouped by category.

**Query Parameters**: `from` (ISO date), `to` (ISO date).

---

### GET /api/v1/reports/inventory

Return inventory report (stock levels, value, low-stock alerts).

---

### GET /api/v1/reports/products/top

Return best-selling products by revenue.

**Query Parameters**: `limit` (default: 10).

---

### GET /api/v1/reports/contacts/top

Return highest-value contacts.

**Query Parameters**: `limit` (default: 10).

---

## AI Engine

Base path: `http://localhost:4200` (internal, not publicly routed through Nginx).

### POST /llm/complete

Generate a text completion using the configured LLM provider.

**Request**

```json
{
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Summarize our Q1 sales." }
  ],
  "model": "gpt-4o",
  "temperature": 0.7,
  "maxTokens": 1024,
  "provider": "openai",
  "tenantId": "tenant-abc",
  "agentId": "agent-xyz"
}
```

Fields `model`, `temperature`, `maxTokens`, `provider`, `tenantId`, and `agentId` are optional.

**Response** `200 OK`

```json
{
  "content": "In Q1, total revenue was...",
  "model": "gpt-4o",
  "provider": "openai",
  "usage": {
    "promptTokens": 42,
    "completionTokens": 128,
    "totalTokens": 170
  }
}
```

---

### POST /llm/stream

Stream a completion as Server-Sent Events.

**Request** — same schema as `/llm/complete`.

**Response** `200 OK` with `Content-Type: text/event-stream`

```
data: {"delta":"In Q1","done":false}

data: {"delta":", total","done":false}

data: {"delta":"","done":true}
```

---

## RAG Service

Base path: `http://localhost:4300` (internal).

### POST /ingest

Ingest a single document: chunk, embed, and store in Qdrant.

**Request**

```json
{
  "content": "Full document text goes here...",
  "metadata": {
    "workspaceId": "ws-123",
    "documentId": "doc-456",
    "agentId": "agent-xyz",
    "title": "Product Manual v2",
    "source": "upload"
  }
}
```

**Response** `201 Created`

```json
{
  "documentId": "doc-456",
  "chunksCreated": 8,
  "workspaceId": "ws-123"
}
```

---

### POST /ingest/batch

Ingest multiple documents in a single request.

**Request**

```json
{
  "documents": [
    {
      "content": "...",
      "metadata": { "workspaceId": "ws-123", "documentId": "doc-001" }
    },
    {
      "content": "...",
      "metadata": { "workspaceId": "ws-123", "documentId": "doc-002" }
    }
  ]
}
```

**Response** `201 Created`

```json
{
  "processed": 2,
  "results": [...]
}
```

---

### DELETE /ingest

Delete vectors by scope.

**Request**

```json
{
  "scope": "document",
  "workspaceId": "ws-123",
  "documentId": "doc-456"
}
```

Valid `scope` values: `document`, `agent`, `workspace`.

---

### DELETE /ingest/:documentId

Delete all chunks for a specific document from the default workspace.

**Response** `200 OK`

---

### GET /ingest/info/:workspaceId

Return collection statistics for a workspace.

**Response** `200 OK`

```json
{
  "workspaceId": "ws-123",
  "exists": true,
  "vectorCount": 1024,
  "indexedDocuments": 42
}
```

---

### POST /query

Semantic search — find the most relevant document chunks for a query.

**Request**

```json
{
  "query": "What is the return policy?",
  "workspaceId": "ws-123",
  "agentId": "agent-xyz",
  "topK": 5,
  "scoreThreshold": 0.7
}
```

`agentId` and `scoreThreshold` are optional.

**Response** `200 OK`

```json
{
  "results": [
    {
      "chunkIndex": 2,
      "documentId": "doc-456",
      "content": "...relevant passage...",
      "score": 0.92,
      "metadata": { "title": "Product Manual v2" }
    }
  ],
  "total": 3
}
```

---

### GET /query/document/:workspaceId/:documentId

Return all chunks for a specific document, ordered by `chunkIndex`.

**Response** `200 OK` — array of chunk objects.

---

## Bootstrap Service

Base path: `http://localhost:4500`. Used only during initial setup via the wizard.

### POST /provision

Provision the platform with initial configuration.

**Request**

```json
{
  "organization": "Acme Corp",
  "adminEmail": "admin@acme.com",
  "templateId": "starter"
}
```

**Response** `201 Created`

```json
{
  "success": true,
  "message": "Platform provisioned successfully"
}
```

---

### GET /templates

Return all available setup templates.

**Response** `200 OK`

```json
{
  "success": true,
  "data": [
    {
      "id": "starter",
      "name": "Starter",
      "description": "Minimal configuration for new workspaces"
    },
    {
      "id": "ecommerce",
      "name": "E-Commerce",
      "description": "Pre-configured for online retail workflows"
    }
  ]
}
```

---

### GET /templates/:id

Return a single template by ID.

**Response** `200 OK`

```json
{
  "success": true,
  "data": {
    "id": "starter",
    "name": "Starter",
    "description": "...",
    "config": { ... }
  }
}
```

---

## Error Responses

All endpoints return standard error envelopes on failure.

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request"
}
```

```json
{
  "statusCode": 401,
  "message": "Unauthorized",
  "error": "Unauthorized"
}
```

```json
{
  "statusCode": 404,
  "message": "Resource not found",
  "error": "Not Found"
}
```

## Roles

| Role | Description |
|------|-------------|
| `OWNER` | Full access including user management and system settings |
| `OPERATOR` | Operational access: CRM, orders, inventory, invoices |
| `MARKETER` | Contacts, campaigns, lead management |
| `FINANCE` | Invoices, expenses, reports |
| `VIEWER` | Read-only access across all modules |
