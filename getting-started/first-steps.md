# First Steps

This guide walks you through the platform after a successful installation. By the end you will have completed the setup wizard, created your first contact, sent an AI query, explored the ERP, and reviewed platform settings.

> Prerequisite: Complete the [installation](installation.md) and confirm all containers are running (`make ps`).

---

## 1. Open the Dashboard

Navigate to [http://localhost:3000](http://localhost:3000) in your browser.

You will see the UniCore login page. Sign in with the default admin credentials:

- **Email**: `admin@unicore.dev`
- **Password**: `admin123`

> Change this password immediately after your first login. Go to **Settings → Account → Change Password**.

Once logged in, the dashboard loads the main navigation. The sidebar contains:

- **Home** — overview and activity feed
- **CRM** — contacts, leads, companies
- **Inventory** — products, warehouses, stock movements
- **Finance** — invoices, payments, expenses
- **AI Chat** — conversational AI interface
- **Agents** — OpenClaw multi-agent workspace
- **Workflows** — automation builder
- **Settings** — platform configuration

---

## 2. Complete the Bootstrap Wizard

On a fresh installation, the platform redirects to the setup wizard at `/wizard`. The wizard guides you through the initial configuration:

1. **Platform name** — set the display name shown in the UI and emails.
2. **Admin profile** — confirm admin email and optionally set a display name.
3. **AI provider** — enter at least one AI provider key (OpenAI or Anthropic) to enable AI features. You can skip this step and configure it later in Settings.
4. **Finish** — the wizard marks setup as complete and redirects to the dashboard.

The wizard status is stored server-side. Once complete, navigating to `/wizard` redirects to the dashboard. To reset the wizard in development:

```bash
curl -s http://localhost:4000/api/v1/settings/wizard-status \
  -X POST \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <your-jwt-token>' \
  -d '{"completed": false}'
```

---

## 3. Create Your First Contact

UniCore's CRM tracks contacts, leads, and companies with built-in lead scoring.

1. Click **CRM** in the left sidebar.
2. Click **New Contact** (top right).
3. Fill in the contact form:
   - **Name**: e.g., `Jane Doe`
   - **Email**: e.g., `jane@example.com`
   - **Phone**: optional
   - **Company**: optional — creates or links a company record
   - **Status**: set to `Lead` for new prospects
4. Click **Save**.

The contact now appears in the CRM list. The platform automatically calculates a lead score based on profile completeness, activity, and interaction history.

**Import contacts in bulk**: CRM supports CSV import via **CRM → Import**. The expected columns are `name`, `email`, `phone`, `company`, and `status`.

---

## 4. Send an AI Query

UniCore's AI engine supports multi-model conversations with context from your business data (via RAG).

1. Click **AI Chat** in the left sidebar.
2. Type a message in the input field. Start with something basic to confirm connectivity:

   ```
   Hello! What can you help me with?
   ```

3. Press **Enter** or click **Send**. The AI response streams in real time.

If the AI returns an error about missing provider keys, add your API key:

1. Go to **Settings → AI → Providers**.
2. Enter your `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`.
3. Click **Save** and retry the chat.

**OpenClaw agents**: For more advanced interactions, open the **Agents** section. UniCore auto-registers 9 default agents on startup (CRM agent, ERP agent, analytics agent, etc.). Each agent has a specialized system prompt and tool set. You can converse with individual agents or orchestrate multi-agent workflows.

**RAG / Knowledge base**: To ground AI responses in your own documents, go to **AI → Knowledge Base → Upload** and add PDFs, text files, or URLs. The RAG service (port 4300) indexes them into Qdrant. Subsequent AI queries will retrieve relevant chunks before generating a response.

---

## 5. Explore ERP Data

The ERP module covers inventory management and financial operations.

### Inventory

1. Click **Inventory** in the sidebar.
2. Click **New Product** to create a catalog item:
   - Set **Name**, **SKU**, **Unit**, and **Price**.
   - Assign to a **Warehouse** (create one via **Inventory → Warehouses → New**).
3. Click **Stock → Add Movement** to record an incoming stock quantity.

The inventory list shows current stock levels. Items below the reorder threshold appear on the **Low Stock** alert view (powered by the `v_low_stock_alert` database view).

### Finance

1. Click **Finance** in the sidebar.
2. Click **Invoices → New Invoice**:
   - Select a **Contact** from your CRM.
   - Add line items (products from inventory or manual entries).
   - Set **Due Date** and **Status** (`Draft` → `Sent` → `Paid`).
3. Click **Save Invoice**.

The Finance dashboard shows:
- **Monthly P&L** — revenue vs expenses (powered by the `v_pnl_monthly` view)
- **AR Aging** — outstanding invoices by age bucket (powered by `v_ar_aging`)
- **Cash flow** — projected inflows and outflows

---

## 6. Review Platform Settings

Go to **Settings** (gear icon in the sidebar or top right) to explore the configuration options:

| Setting Section | What You Can Configure |
|----------------|----------------------|
| **General** | Platform name, timezone, language, logo |
| **Account** | Admin email, password, profile picture |
| **AI → Providers** | OpenAI / Anthropic / Ollama API keys, primary provider |
| **AI → Agents** | Default agent prompts, tool permissions |
| **Integrations** | Telegram, LINE, and other channel tokens |
| **Security** | JWT expiry, session timeout, 2FA (Pro) |
| **SSO** | SAML / OIDC configuration (Pro edition) |
| **White Label** | Custom domain, colors, fonts, logo (Pro edition) |
| **License** | View current license tier, key, and expiry |

---

## 7. Check Service Health

At any time, you can verify all backend services are healthy:

```bash
# Container status
make ps

# API gateway health
curl -s http://localhost:4000/health

# View logs for a specific service
make logs/api-gateway
make logs/erp
make logs/ai-engine
```

For real-time log tailing across all services:

```bash
make logs
# or: docker compose --profile apps --profile workflows logs -f --tail=100
```

---

## Next Steps

- **Configure AI providers** — see [configuration.md](configuration.md#ai-providers)
- **Set up messaging channels** — see the [Channels guide](../guides/channels/)
- **Build a workflow** — see the [Workflows guide](../guides/workflows/)
- **Deploy to production** — see the [Docker deployment guide](../deployment/docker.md)
- **Upgrade to Pro** — see the [Pro edition guide](../editions/pro.md)
