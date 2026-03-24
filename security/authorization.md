# Authorization

Updated: 2026-03-22

UniCore enforces authorization at every layer: NestJS route guards on individual endpoints, Prisma query scoping in service repositories, and (for Pro/Enterprise) the dedicated `@bemindlabs/unicore-rbac` package for fine-grained permission management.

---

## Role System

All users are assigned exactly one role. Roles follow a hierarchy from broadest to most restricted access.

| Role | Description | Typical User |
|------|-------------|-------------|
| `OWNER` | Full system access — all settings, billing, user management | Business owner, platform admin |
| `OPERATOR` | Operational access — CRM, inventory, workflows, agents | Operations manager, team lead |
| `MARKETER` | Marketing scope — contacts, campaigns, messaging channels | Marketing staff |
| `FINANCE` | Finance scope — invoices, payments, expenses, reports | Accountant, finance team |
| `VIEWER` | Read-only access to all permitted resources | Stakeholders, auditors |

Roles are stored in the `User` model in the API Gateway's Prisma schema and encoded in the JWT `role` claim.

---

## Permission Model

### Community Edition (Flat RBAC)

Community uses a simple role-based check: each endpoint declares the minimum role required. The check is purely hierarchical — a higher role always satisfies a lower role requirement.

```
OWNER ⊇ OPERATOR ⊇ MARKETER / FINANCE ⊇ VIEWER
```

> `MARKETER` and `FINANCE` are at the same level; each has distinct allowed resource sets rather than a strict superset relationship.

**Default resource access matrix**

| Resource | OWNER | OPERATOR | MARKETER | FINANCE | VIEWER |
|----------|-------|----------|----------|---------|--------|
| Users & Roles | RW | — | — | — | R |
| Settings | RW | R | — | — | R |
| CRM / Contacts | RW | RW | RW | R | R |
| Inventory | RW | RW | R | R | R |
| Orders | RW | RW | — | R | R |
| Invoices / Payments | RW | R | — | RW | R |
| Expenses | RW | R | — | RW | R |
| Reports | RW | R | — | R | R |
| AI Engine / RAG | RW | RW | R | — | R |
| Workflows | RW | RW | — | — | R |
| Agents (OpenClaw) | RW | RW | — | — | R |
| Channels | RW | RW | RW | — | R |
| API Keys | RW | RW | — | — | — |

`RW` = read + write, `R` = read only, `—` = no access.

---

## Route Guards (NestJS)

Authorization is enforced via NestJS guards applied at the controller or handler level.

### `@Roles()` Decorator

```typescript
import { Roles } from '@bemindlabs/unicore-shared-types';

@Controller('contacts')
export class ContactsController {
  @Get()
  @Roles('VIEWER') // any role at VIEWER or above
  findAll() { ... }

  @Post()
  @Roles('MARKETER') // MARKETER, OPERATOR, or OWNER
  create(@Body() dto: CreateContactDto) { ... }

  @Delete(':id')
  @Roles('OWNER') // OWNER only
  remove(@Param('id') id: string) { ... }
}
```

### `RolesGuard` Implementation

The `RolesGuard` reads the `role` claim from the JWT payload (injected by `JwtAuthGuard`) and compares it against the roles declared via `@Roles()`.

```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true; // unguarded route

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => ROLE_HIERARCHY[user.role] >= ROLE_HIERARCHY[role]);
  }
}
```

Guards are registered globally in `AppModule` so every route is protected by default. Public routes are marked with `@Public()`.

### Global Guard Registration

```typescript
// app.module.ts
providers: [
  { provide: APP_GUARD, useClass: JwtAuthGuard },   // auth first
  { provide: APP_GUARD, useClass: RolesGuard },      // then RBAC
],
```

---

## Pro RBAC Package

`unicore-pro/packages/rbac` replaces the flat role model with a fully configurable permission system.

### Concepts

| Concept | Description |
|---------|-------------|
| **Permission** | A named action on a resource (e.g. `contacts:delete`, `reports:export`) |
| **Role** | A named group of permissions (custom, not limited to the 5 built-in roles) |
| **Policy** | A rule that grants or denies permissions to roles or individual users |
| **Scope** | Limits a permission to specific resources (e.g. own records, specific warehouse) |

### Prisma Schema (Pro)

```prisma
model RbacRole {
  id          String       @id @default(cuid())
  name        String       @unique
  permissions String[]     // e.g. ["contacts:read", "contacts:write"]
  users       User[]
  createdAt   DateTime     @default(now())
}

model RbacPolicy {
  id         String   @id @default(cuid())
  roleId     String?
  userId     String?
  resource   String   // e.g. "contacts"
  action     String   // e.g. "delete"
  effect     String   // "allow" | "deny"
  conditions Json?    // optional attribute-based conditions
}
```

### Permission Check

```typescript
// In a service
const allowed = await this.rbacService.can(user, 'contacts:delete', contactId);
if (!allowed) throw new ForbiddenException();
```

`RbacService` evaluates policies in priority order: explicit **deny** always wins over **allow**.

### Configuring Roles via Dashboard

Pro users manage roles and permissions in **Settings → Roles & Permissions**:

1. Create a custom role (e.g. `Regional Manager`).
2. Assign permissions from the permission catalogue.
3. Optionally scope permissions to specific resources.
4. Assign the role to users.

---

## Enterprise Tenant-Level Permissions

Enterprise Edition adds multi-tenancy with isolated permission namespaces per tenant.

### Tenant Isolation

Each tenant has its own set of roles, policies, and user memberships. The `tenantId` claim in the JWT is used to scope all database queries and permission checks.

```
tenantId: ten_01HABC...
  └── Roles: [Admin, Agent, Viewer, ...]
       └── Policies per role
            └── Users assigned to roles within this tenant
```

### Cross-Tenant Access

Only platform-level `OWNER` accounts (with no `tenantId`) can access multiple tenants. Service accounts used for infrastructure never carry tenant-scoped tokens.

### Tenant-Level SSO Mapping

Enterprise SSO can map IdP groups to UniCore roles:

```json
{
  "ssoGroupMappings": [
    { "idpGroup": "finance-team", "unicoreRole": "FINANCE" },
    { "idpGroup": "managers",     "unicoreRole": "OPERATOR" }
  ]
}
```

---

## Related

- [Authentication](authentication.md)
- [Data Privacy](data-privacy.md)
- [Best Practices](best-practices.md)
- Pro RBAC package (available in UniCore Pro edition)
