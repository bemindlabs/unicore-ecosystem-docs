# Coding Standards

Updated: 2026-03-22

This document defines the TypeScript coding conventions, linting rules, formatting standards, and architectural patterns used across the UniCore platform.

## TypeScript

### Compiler Options

All packages extend the shared base tsconfig at `packages/config/src/typescript/base.json`. NestJS services extend `packages/config/src/typescript/nest.json` which adds decorator support. TypeScript 5.5+ with strict mode is required.

**Base config** (`packages/config/src/typescript/base.json`):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "incremental": true
  }
}
```

**NestJS override** (`packages/config/src/typescript/nest.json`) adds:

```json
{
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "declaration": true,
    "removeComments": true,
    "allowSyntheticDefaultImports": true
  }
}
```

Individual services extend `nest.json` and override only `outDir`, `baseUrl`, and `paths`.

### Type Conventions

**Always use explicit return types on public functions:**

```typescript
// Good
function getUser(id: string): Promise<User> { ... }

// Avoid
function getUser(id: string) { ... }
```

**Prefer interfaces over type aliases for object shapes:**

```typescript
// Good
interface CreateContactDto {
  name: string;
  email: string;
  phone?: string;
}

// Acceptable for unions/intersections
type ContactStatus = 'lead' | 'customer' | 'inactive';
type ContactWithMeta = Contact & { meta: Record<string, unknown> };
```

**Use `unknown` instead of `any`:**

```typescript
// Good
function parseJson(raw: string): unknown {
  return JSON.parse(raw);
}

// Avoid
function parseJson(raw: string): any {
  return JSON.parse(raw);
}
```

**Avoid non-null assertions (`!`) — use guards instead:**

```typescript
// Good
if (!user) throw new NotFoundException('User not found');
return user.email;

// Avoid
return user!.email;
```

**Use `readonly` for immutable data:**

```typescript
interface Config {
  readonly jwtSecret: string;
  readonly dbUrl: string;
}
```

## ESLint

All packages extend the shared ESLint config (`@bemindlabs/unicore-eslint-config`):

```json
// .eslintrc.json
{
  "extends": ["@bemindlabs/unicore-eslint-config"],
  "parserOptions": {
    "project": "./tsconfig.json"
  }
}
```

### Key Rules

```javascript
// Enforced rules (not exhaustive)
{
  "@typescript-eslint/no-explicit-any": "error",
  "@typescript-eslint/explicit-function-return-type": "warn",
  "@typescript-eslint/no-floating-promises": "error",
  "@typescript-eslint/await-thenable": "error",
  "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
  "no-console": ["warn", { "allow": ["warn", "error"] }],
  "prefer-const": "error",
  "no-var": "error",
  "eqeqeq": ["error", "always"]
}
```

### Running ESLint

```bash
# Lint a specific package
pnpm --filter @bemindlabs/unicore-api-gateway lint

# Auto-fix
pnpm --filter @bemindlabs/unicore-api-gateway lint --fix

# Lint all
pnpm lint
```

## Prettier

All code is formatted with Prettier. Configuration in `.prettierrc`:

```json
{
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

Format on save is configured for VS Code. To format manually:

```bash
# Format all files
pnpm format

# Check formatting without modifying
pnpm format:check
```

## NestJS Patterns

### Module Structure

Each NestJS service follows this directory layout:

```
services/api-gateway/src/
├── main.ts                    ← Bootstrap entry point
├── app.module.ts              ← Root module
├── prisma/
│   ├── schema.prisma          ← Prisma schema
│   └── prisma.service.ts      ← PrismaClient wrapper
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.guard.ts          ← JWT guard
│   ├── dto/
│   │   ├── login.dto.ts
│   │   └── register.dto.ts
│   └── strategies/
│       ├── jwt.strategy.ts
│       └── local.strategy.ts
└── users/
    ├── users.module.ts
    ├── users.controller.ts
    ├── users.service.ts
    └── dto/
        ├── create-user.dto.ts
        └── update-user.dto.ts
```

### Controllers

```typescript
import { Controller, Get, Post, Body, Param, UseGuards, HttpCode, HttpStatus } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/auth.guard';
import { CreateContactDto } from './dto/create-contact.dto';
import { ContactsService } from './contacts.service';

@Controller('contacts')
@UseGuards(JwtAuthGuard)
export class ContactsController {
  constructor(private readonly contactsService: ContactsService) {}

  @Get()
  findAll(): Promise<Contact[]> {
    return this.contactsService.findAll();
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() dto: CreateContactDto): Promise<Contact> {
    return this.contactsService.create(dto);
  }

  @Get(':id')
  findOne(@Param('id') id: string): Promise<Contact> {
    return this.contactsService.findOne(id);
  }
}
```

### Services

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateContactDto } from './dto/create-contact.dto';
import { Contact } from '@prisma/client';

@Injectable()
export class ContactsService {
  constructor(private readonly prisma: PrismaService) {}

  async findAll(): Promise<Contact[]> {
    return this.prisma.contact.findMany({
      orderBy: { createdAt: 'desc' },
    });
  }

  async findOne(id: string): Promise<Contact> {
    const contact = await this.prisma.contact.findUnique({ where: { id } });
    if (!contact) throw new NotFoundException(`Contact ${id} not found`);
    return contact;
  }

  async create(dto: CreateContactDto): Promise<Contact> {
    return this.prisma.contact.create({ data: dto });
  }
}
```

### DTOs with class-validator

```typescript
import { IsEmail, IsString, IsOptional, MinLength, IsEnum } from 'class-validator';

export class CreateContactDto {
  @IsString()
  @MinLength(1)
  name: string;

  @IsEmail()
  email: string;

  @IsOptional()
  @IsString()
  phone?: string;

  @IsOptional()
  @IsEnum(ContactStatus)
  status?: ContactStatus;
}
```

### Exception Handling

Use NestJS built-in exceptions. Do not throw raw `Error` objects from controllers or services:

```typescript
import {
  NotFoundException,
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  ConflictException,
  InternalServerErrorException,
} from '@nestjs/common';

// Good
throw new NotFoundException('User not found');
throw new ConflictException('Email already in use');

// Avoid
throw new Error('User not found');
```

### Global Exception Filter

The API Gateway uses a global exception filter that:
- Formats all errors as `{ statusCode, message, error }` JSON
- Logs 5xx errors to the audit log
- Sanitizes stack traces in production (`NODE_ENV=production`)

## Next.js / React Patterns

### Component Structure

```typescript
// apps/dashboard/src/components/contacts/ContactCard.tsx
import { Contact } from '@bemindlabs/unicore-shared-types';

interface ContactCardProps {
  contact: Contact;
  onEdit?: (id: string) => void;
}

export function ContactCard({ contact, onEdit }: ContactCardProps): JSX.Element {
  return (
    <div className="rounded-lg border p-4">
      <h3 className="font-semibold">{contact.name}</h3>
      <p className="text-muted-foreground text-sm">{contact.email}</p>
    </div>
  );
}
```

### Data Fetching

Use SWR for client-side data fetching in the dashboard:

```typescript
import useSWR from 'swr';
import { fetcher } from '@/lib/fetcher';
import { Contact } from '@bemindlabs/unicore-shared-types';

function useContacts() {
  const { data, error, isLoading, mutate } = useSWR<Contact[]>(
    '/api/v1/contacts',
    fetcher,
  );

  return { contacts: data, error, isLoading, mutate };
}
```

Server-side data fetching uses Next.js Server Components where possible:

```typescript
// app/contacts/page.tsx (Server Component)
import { getContacts } from '@/lib/api';

export default async function ContactsPage() {
  const contacts = await getContacts();
  return <ContactList contacts={contacts} />;
}
```

### Tailwind CSS

- Use Tailwind utility classes directly; avoid inline `style` attributes
- Use `cn()` from `@bemindlabs/unicore-ui` for conditional classes:

```typescript
import { cn } from '@bemindlabs/unicore-ui/lib/utils';

<div className={cn('rounded-lg p-4', isActive && 'bg-primary text-primary-foreground')}>
```

- Use shadcn/ui components from `@bemindlabs/unicore-ui` — do not install shadcn/ui directly in the app

## Naming Conventions

| Entity | Convention | Example |
|--------|-----------|---------|
| Files | `kebab-case.ts` | `create-contact.dto.ts` |
| Classes | `PascalCase` | `ContactsService` |
| Interfaces | `PascalCase` | `CreateContactDto` |
| Functions / methods | `camelCase` | `findOneById()` |
| Variables | `camelCase` | `contactList` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_CONTACTS_PER_PAGE` |
| Enum members | `SCREAMING_SNAKE_CASE` | `ContactStatus.ACTIVE` |
| Database columns | `snake_case` (Prisma maps) | `created_at` |
| REST endpoints | `kebab-case` | `/api/v1/contact-groups` |
| Environment vars | `SCREAMING_SNAKE_CASE` | `DATABASE_URL` |

## Import Order

Use the following import order (enforced by ESLint `import/order`):

```typescript
// 1. Node.js built-ins
import { join } from 'path';

// 2. External packages
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';

// 3. Internal workspace packages
import { ContactStatus } from '@bemindlabs/unicore-shared-types';

// 4. Local imports (absolute paths via tsconfig paths)
import { PrismaService } from '../prisma/prisma.service';
import { CreateContactDto } from './dto/create-contact.dto';
```

## Comments and Documentation

- Write self-documenting code — prefer clear names over comments
- Use JSDoc for public API methods in shared packages:

```typescript
/**
 * Calculates the lead score for a contact based on engagement metrics.
 *
 * @param contact - The contact to score
 * @param weights - Optional custom weight configuration
 * @returns A score between 0 and 100
 */
export function calculateLeadScore(
  contact: Contact,
  weights?: LeadScoringWeights,
): number { ... }
```

- Use `// TODO:` comments for known pending work (tracked in Jira)
- Do not commit commented-out code

## Related Documentation

- [Local setup guide](./local-setup.md)
- [Testing guide](./testing.md)
- [Contributing guide](./contributing.md)

---

© 2026 BeMind Technology
