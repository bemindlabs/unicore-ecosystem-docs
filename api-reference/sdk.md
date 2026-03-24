# SDK Reference — JavaScript / TypeScript Client

Updated: 2026-03-22

The UniCore SDK provides a typed client for the REST API, WebSocket gateway, and webhook utilities. It targets Node.js 20+ and modern browsers.

## Installation

```bash
# pnpm (recommended)
pnpm add @bemindlabs/unicore-sdk

# npm
npm install @bemindlabs/unicore-sdk

# yarn
yarn add @bemindlabs/unicore-sdk
```

The SDK is part of the `@bemindlabs/unicore-*` package family and is built with TypeScript 5.5 in strict mode.

---

## Quick Start

```typescript
import { UniCoreClient } from '@bemindlabs/unicore-sdk';

const client = new UniCoreClient({
  baseUrl: 'https://unicore.example.com',
});

// Authenticate
await client.auth.login({
  email: 'admin@example.com',
  password: 'your-password',
});

// Fetch contacts
const { data, total } = await client.erp.contacts.list({ page: 1, limit: 20 });
console.log(`${total} contacts found`);
```

---

## Client Initialization

```typescript
import { UniCoreClient } from '@bemindlabs/unicore-sdk';

const client = new UniCoreClient({
  // Required: base URL of your UniCore instance
  baseUrl: 'https://unicore.example.com',

  // Optional: provide an initial access token
  accessToken: process.env.UNICORE_ACCESS_TOKEN,

  // Optional: provide a refresh token for automatic renewal
  refreshToken: process.env.UNICORE_REFRESH_TOKEN,

  // Optional: hook called when the access token is refreshed
  onTokenRefresh: (tokens) => {
    saveTokensToStorage(tokens.accessToken, tokens.refreshToken);
  },

  // Optional: request timeout in milliseconds (default: 30000)
  timeout: 30_000,

  // Optional: custom fetch implementation
  fetchImpl: globalThis.fetch,
});
```

### Configuration Options

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `baseUrl` | `string` | Yes | Base URL of the UniCore API |
| `accessToken` | `string` | No | Initial JWT access token |
| `refreshToken` | `string` | No | Refresh token for automatic renewal |
| `onTokenRefresh` | `function` | No | Callback invoked after token refresh |
| `timeout` | `number` | No | Request timeout in ms (default: 30000) |
| `fetchImpl` | `function` | No | Custom fetch implementation |

---

## Authentication

### Login

```typescript
const session = await client.auth.login({
  email: 'admin@example.com',
  password: 'your-password',
});

console.log(session.accessToken);  // JWT access token
console.log(session.refreshToken); // Refresh token
console.log(session.expiresIn);    // Seconds until expiry (typically 900)
console.log(session.user.role);    // OWNER | OPERATOR | MARKETER | FINANCE | VIEWER
```

After login, the SDK automatically stores the tokens and includes `Authorization: Bearer <token>` on all subsequent requests.

### Register

```typescript
const session = await client.auth.register({
  email: 'jane@example.com',
  name: 'Jane Doe',
  password: 'SecurePass1',
  confirmPassword: 'SecurePass1',
});
```

### Refresh Token

The SDK automatically refreshes the access token before it expires if a `refreshToken` was provided. You can also refresh manually:

```typescript
const newSession = await client.auth.refresh(refreshToken);
```

### Logout

```typescript
await client.auth.logout(refreshToken);
// Clears stored tokens from the client
```

### Get Current User

```typescript
const me = await client.auth.me();
// { id, email, name, role, createdAt }
```

### Provision Admin (Bootstrap)

Used during initial setup only:

```typescript
await client.auth.provisionAdmin({
  email: 'admin@example.com',
  name: 'Admin',
  password: 'your-secure-password',
  role: 'OWNER',
  bootstrapSecret: process.env.BOOTSTRAP_SECRET,
});
```

### Using a Pre-Existing Token

```typescript
const client = new UniCoreClient({ baseUrl: '...' });
client.setTokens({
  accessToken: 'eyJhbGci...',
  refreshToken: 'a1b2c3...',
});
```

---

## ERP — Contacts

```typescript
// List contacts
const { data, total } = await client.erp.contacts.list({
  type: 'LEAD',
  search: 'acme',
  page: 1,
  limit: 20,
});

// Create a contact
const contact = await client.erp.contacts.create({
  name: 'Acme Corp',
  email: 'contact@acme.com',
  phone: '+1-555-0100',
  type: 'LEAD',
});

// Get a contact
const contact = await client.erp.contacts.get('550e8400-...');

// Update a contact
const updated = await client.erp.contacts.update('550e8400-...', {
  type: 'CUSTOMER',
});

// Update lead score
await client.erp.contacts.updateLeadScore('550e8400-...', 85);

// Get top leads
const leads = await client.erp.contacts.topLeads({ minScore: 60, limit: 10 });

// Delete a contact
await client.erp.contacts.delete('550e8400-...');
```

---

## ERP — Orders

```typescript
// Create an order
const order = await client.erp.orders.create({
  contactId: '550e8400-...',
  items: [
    { productId: '550e8400-...', quantity: 2, unitPrice: 99.99 },
  ],
});

// List orders
const { data } = await client.erp.orders.list({ status: 'DRAFT' });

// Order lifecycle transitions
await client.erp.orders.confirm(order.id);
await client.erp.orders.process(order.id);
await client.erp.orders.ship(order.id, { trackingNumber: '1Z999AA1...', carrier: 'UPS' });
await client.erp.orders.fulfill(order.id, { deliveredAt: new Date().toISOString() });

// Cancel or refund
await client.erp.orders.cancel(order.id, { reason: 'Customer request' });
await client.erp.orders.refund(order.id);
```

---

## ERP — Inventory

```typescript
// Create a product
const product = await client.erp.inventory.create({
  name: 'Wireless Headphones',
  sku: 'WH-001',
  price: 149.99,
  cost: 75.00,
  stockQuantity: 100,
  reorderPoint: 20,
});

// Adjust stock
await client.erp.inventory.adjustStock(product.id, {
  delta: -5,
  reason: 'Damaged goods',
});

// Restock
await client.erp.inventory.restock(product.id, {
  quantity: 50,
  unitCost: 74.00,
  supplier: 'Supplier Ltd',
});

// Low stock alerts
const lowStock = await client.erp.inventory.lowStock();

// Stock movements history
const movements = await client.erp.inventory.movements(product.id);
```

---

## ERP — Invoices

```typescript
// Create an invoice
const invoice = await client.erp.invoices.create({
  contactId: '550e8400-...',
  dueDate: '2026-02-01T00:00:00.000Z',
  lineItems: [
    { description: 'Consulting', quantity: 10, unitPrice: 150.00 },
  ],
});

// Invoice workflow
await client.erp.invoices.send(invoice.id);
await client.erp.invoices.recordPayment(invoice.id, {
  amount: 1500.00,
  method: 'BANK_TRANSFER',
  reference: 'TXN-001',
  paidAt: new Date().toISOString(),
});
await client.erp.invoices.cancel(invoice.id);

// Batch mark overdue
await client.erp.invoices.markOverdue();
```

---

## ERP — Expenses

```typescript
// Submit an expense
const expense = await client.erp.expenses.create({
  title: 'Team lunch',
  amount: 245.00,
  currency: 'USD',
  category: 'MEALS',
  date: '2026-01-10T00:00:00.000Z',
});

// Approve / reject
await client.erp.expenses.approve(expense.id, { notes: 'Approved' });
await client.erp.expenses.reject(expense.id, { reason: 'Exceeds budget' });
```

---

## ERP — Reports

```typescript
// Dashboard KPIs
const dashboard = await client.erp.reports.dashboard();

// Revenue report
const revenue = await client.erp.reports.revenue({
  from: '2026-01-01',
  to: '2026-03-31',
});

// Expense breakdown
const expensesByCategory = await client.erp.reports.expensesByCategory({
  from: '2026-01-01',
  to: '2026-03-31',
});

// Inventory report
const inventory = await client.erp.reports.inventory();

// Top performers
const topProducts = await client.erp.reports.topProducts({ limit: 10 });
const topContacts = await client.erp.reports.topContacts({ limit: 10 });
```

---

## AI Engine

```typescript
// Text completion
const result = await client.ai.complete({
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: 'Summarise Q1 sales.' },
  ],
  model: 'gpt-4o',
  temperature: 0.7,
  maxTokens: 1024,
});

console.log(result.content);
console.log(result.usage.totalTokens);

// Streaming completion
const stream = client.ai.stream({
  messages: [
    { role: 'user', content: 'Write a product description for wireless headphones.' },
  ],
});

for await (const chunk of stream) {
  process.stdout.write(chunk.delta);
  if (chunk.done) break;
}
```

---

## RAG Service

```typescript
// Ingest a document
await client.rag.ingest({
  content: 'Full document text...',
  metadata: {
    workspaceId: 'ws-123',
    documentId: 'doc-001',
    title: 'Product Manual v2',
  },
});

// Batch ingest
await client.rag.ingestBatch([
  { content: '...', metadata: { workspaceId: 'ws-123', documentId: 'doc-002' } },
  { content: '...', metadata: { workspaceId: 'ws-123', documentId: 'doc-003' } },
]);

// Semantic search
const results = await client.rag.query({
  query: 'What is the return policy?',
  workspaceId: 'ws-123',
  topK: 5,
  scoreThreshold: 0.7,
});

for (const result of results.results) {
  console.log(`Score: ${result.score}`, result.content);
}

// Delete document
await client.rag.deleteDocument('doc-001');

// Workspace info
const info = await client.rag.workspaceInfo('ws-123');
console.log(`${info.vectorCount} vectors indexed`);
```

---

## OpenClaw WebSocket Client

The SDK includes a typed WebSocket client for the OpenClaw multi-agent gateway.

```typescript
import { OpenClawClient } from '@bemindlabs/unicore-sdk';

const agent = new OpenClawClient({
  url: 'ws://localhost:18789',
  agentId: 'my-agent-01',
  name: 'My Custom Agent',
  agentType: 'custom',
  version: '1.0.0',
  capabilities: [
    {
      name: 'greeting',
      version: '1.0.0',
      description: 'Greet a user by name',
    },
  ],
  tags: ['custom'],
});

// Connect and register
await agent.connect();

// Listen for direct messages
agent.on('message:direct', (message) => {
  console.log('Received:', message.payload.topic, message.payload.data);

  // Reply
  agent.sendDirect({
    toAgentId: message.payload.fromAgentId,
    topic: 'greeting.response',
    data: { greeting: 'Hello!' },
    correlationId: message.payload.correlationId,
  });
});

// Subscribe to a channel
await agent.subscribe('orders.created');

agent.on('message:publish', (message) => {
  if (message.payload.channel === 'orders.created') {
    console.log('New order:', message.payload.data);
  }
});

// Publish to a channel
await agent.publish('crm.updated', { contactId: '550e8400-...' });

// Report state
await agent.setState('idle');
await agent.setState('running');

// Graceful disconnect
await agent.disconnect();
```

### OpenClawClient Options

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `url` | `string` | Yes | WebSocket server URL |
| `agentId` | `string` | Yes | Unique agent identifier |
| `name` | `string` | Yes | Human-readable agent name |
| `agentType` | `string` | Yes | Agent type category |
| `version` | `string` | Yes | Semver version string |
| `capabilities` | `array` | Yes | Declared capabilities |
| `tags` | `string[]` | No | Categorisation tags |
| `heartbeatInterval` | `number` | No | Heartbeat interval in ms (default: 30000) |
| `reconnect` | `boolean` | No | Auto-reconnect on disconnect (default: true) |
| `reconnectDelay` | `number` | No | Reconnect delay in ms (default: 5000) |

---

## Settings

```typescript
// Wizard status (no auth required)
const status = await client.settings.getWizardStatus();
await client.settings.setWizardStatus({ completed: true });

// Generic settings
const config = await client.settings.get('branding');
await client.settings.set('branding', { primaryColor: '#6366f1', logoUrl: '...' });

// Team size
const { count } = await client.settings.teamCount();
```

---

## Error Handling

The SDK throws typed errors for all failure cases.

```typescript
import {
  UniCoreError,
  AuthenticationError,
  NotFoundError,
  ValidationError,
  RateLimitError,
} from '@bemindlabs/unicore-sdk';

try {
  const contact = await client.erp.contacts.get('non-existent-id');
} catch (err) {
  if (err instanceof NotFoundError) {
    console.error('Contact not found:', err.message);
  } else if (err instanceof AuthenticationError) {
    console.error('Token expired, please log in again');
  } else if (err instanceof ValidationError) {
    console.error('Validation errors:', err.errors);
    // err.errors: Array<{ field: string; message: string }>
  } else if (err instanceof RateLimitError) {
    console.error(`Rate limited. Retry after ${err.retryAfter}s`);
  } else if (err instanceof UniCoreError) {
    console.error(`API error ${err.statusCode}:`, err.message);
  } else {
    throw err; // Re-throw unexpected errors
  }
}
```

### Error Classes

| Class | Status | Description |
|-------|--------|-------------|
| `UniCoreError` | Any | Base error class |
| `AuthenticationError` | 401 | Invalid or expired token |
| `ForbiddenError` | 403 | Insufficient permissions for the role |
| `NotFoundError` | 404 | Resource does not exist |
| `ValidationError` | 400 | Request body failed validation |
| `ConflictError` | 409 | Resource already exists |
| `RateLimitError` | 429 | Too many requests |
| `ServerError` | 500 | Unexpected server error |

---

## TypeScript Types

All request and response types are exported from the package root:

```typescript
import type {
  // Auth
  LoginDto,
  RegisterDto,
  AuthResponse,
  User,

  // ERP
  Contact,
  CreateContactDto,
  Order,
  CreateOrderDto,
  OrderStatus,
  Product,
  Invoice,
  Expense,
  ReportDashboard,

  // AI
  CompleteRequest,
  CompleteResponse,
  Message,

  // RAG
  IngestDocumentDto,
  RetrievalQuery,
  RetrievalResult,

  // WebSocket
  AgentCapability,
  RegisteredAgent,
} from '@bemindlabs/unicore-sdk';
```

---

## Pagination

List endpoints return a standard paginated response:

```typescript
interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}
```

Iterate through all pages:

```typescript
async function* paginate<T>(
  listFn: (opts: { page: number; limit: number }) => Promise<PaginatedResponse<T>>,
  limit = 50,
) {
  let page = 1;
  while (true) {
    const result = await listFn({ page, limit });
    yield* result.data;
    if (result.data.length < limit) break;
    page++;
  }
}

for await (const contact of paginate((opts) => client.erp.contacts.list(opts))) {
  console.log(contact.name);
}
```

---

## Environment Configuration

Configure the client from environment variables:

```typescript
const client = new UniCoreClient({
  baseUrl: process.env.UNICORE_BASE_URL ?? 'http://localhost:4000',
  accessToken: process.env.UNICORE_ACCESS_TOKEN,
  refreshToken: process.env.UNICORE_REFRESH_TOKEN,
});
```

Recommended `.env` entries:

```env
UNICORE_BASE_URL=https://unicore.example.com
UNICORE_ACCESS_TOKEN=eyJhbGci...
UNICORE_REFRESH_TOKEN=a1b2c3...
```

---

## Requirements

| Dependency | Minimum Version |
|------------|----------------|
| Node.js | 20.0.0 |
| TypeScript | 5.5 (for consumers) |
| `@bemindlabs/unicore-shared-types` | workspace version |
