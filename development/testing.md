# Testing Guide

This guide covers the testing strategy, tooling, and conventions for the UniCore platform. Tests are organized into three tiers: unit tests, integration tests, and end-to-end (E2E) tests.

## Testing Stack

| Tool | Purpose | Config |
|------|---------|--------|
| Jest 29 | Unit and integration tests (all services) | `jest.config.ts` |
| Playwright 1.58 | E2E browser tests (dashboard) | `playwright.config.ts` |
| SuperTest | HTTP integration tests (NestJS APIs) | Used with Jest |
| `@testcontainers/postgresql` | Isolated Postgres for integration tests | — |
| `@faker-js/faker` | Test data generation | — |

## Coverage Targets

| Tier | Target |
|------|--------|
| Unit tests | ≥ 80% line coverage per service |
| Integration tests | All public API endpoints covered |
| E2E tests | All critical user flows covered |

## Unit Tests (Jest)

### Configuration

Each service has a `jest.config.ts` at its root:

```typescript
// services/erp/jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: ['**/*.(t|j)s'],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  coverageThreshold: {
    global: {
      lines: 80,
      functions: 80,
      branches: 70,
    },
  },
};

export default config;
```

### Running Unit Tests

```bash
# Run all unit tests (all packages via Turborepo)
pnpm test

# Run unit tests for a specific service
pnpm --filter @unicore/erp test
pnpm --filter @unicore/api-gateway test

# Watch mode (rerun on file changes)
pnpm --filter @unicore/erp test --watch

# Coverage report
pnpm --filter @unicore/erp test --coverage

# Run a specific test file
pnpm --filter @unicore/erp test -- --testPathPattern=contacts.service
```

### Writing Unit Tests

Unit tests live alongside source files as `*.spec.ts`:

```
services/erp/src/
├── contacts/
│   ├── contacts.service.ts
│   ├── contacts.service.spec.ts   ← Unit test
│   ├── contacts.controller.ts
│   └── contacts.controller.spec.ts
```

#### Service Unit Test

```typescript
// services/erp/src/contacts/contacts.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException } from '@nestjs/common';
import { ContactsService } from './contacts.service';
import { PrismaService } from '../prisma/prisma.service';
import { createMockPrismaService, mockContact } from '../test/helpers';

describe('ContactsService', () => {
  let service: ContactsService;
  let prisma: ReturnType<typeof createMockPrismaService>;

  beforeEach(async () => {
    prisma = createMockPrismaService();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ContactsService,
        { provide: PrismaService, useValue: prisma },
      ],
    }).compile();

    service = module.get<ContactsService>(ContactsService);
  });

  describe('findOne', () => {
    it('returns a contact when found', async () => {
      const contact = mockContact();
      prisma.contact.findUnique.mockResolvedValue(contact);

      const result = await service.findOne(contact.id);

      expect(result).toEqual(contact);
      expect(prisma.contact.findUnique).toHaveBeenCalledWith({
        where: { id: contact.id },
      });
    });

    it('throws NotFoundException when contact does not exist', async () => {
      prisma.contact.findUnique.mockResolvedValue(null);

      await expect(service.findOne('nonexistent')).rejects.toThrow(
        NotFoundException,
      );
    });
  });

  describe('create', () => {
    it('creates and returns a new contact', async () => {
      const dto = { name: 'Acme Corp', email: 'hello@acme.com' };
      const contact = mockContact(dto);
      prisma.contact.create.mockResolvedValue(contact);

      const result = await service.create(dto);

      expect(result).toEqual(contact);
      expect(prisma.contact.create).toHaveBeenCalledWith({ data: dto });
    });
  });
});
```

#### Test Helpers

Shared test utilities live in `src/test/helpers.ts`:

```typescript
// services/erp/src/test/helpers.ts
import { faker } from '@faker-js/faker';
import { Contact } from '@prisma/client';

export function mockContact(overrides: Partial<Contact> = {}): Contact {
  return {
    id: faker.string.uuid(),
    name: faker.company.name(),
    email: faker.internet.email(),
    phone: faker.phone.number(),
    status: 'lead',
    leadScore: faker.number.int({ min: 0, max: 100 }),
    createdAt: new Date(),
    updatedAt: new Date(),
    ...overrides,
  };
}

export function createMockPrismaService() {
  return {
    contact: {
      findMany: jest.fn(),
      findUnique: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    },
    invoice: {
      findMany: jest.fn(),
      findUnique: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
    },
    // ... other models
  };
}
```

## Integration Tests (SuperTest + Jest)

Integration tests test full HTTP request/response cycles against a running NestJS application, using a real (test) database.

### Configuration

Integration tests use a `jest-e2e.config.ts` at the service root:

```typescript
// services/erp/jest-e2e.config.ts
import type { Config } from 'jest';

const config: Config = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: '.',
  testRegex: '.e2e-spec.ts$',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  testEnvironment: 'node',
  globalSetup: './test/global-setup.ts',
  globalTeardown: './test/global-teardown.ts',
};

export default config;
```

### Running Integration Tests

```bash
# Run integration tests for a service
pnpm --filter @unicore/erp test:e2e

# Run all integration tests
pnpm test:e2e
```

### Writing Integration Tests (SuperTest)

```typescript
// services/erp/test/contacts.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { PrismaService } from '../src/prisma/prisma.service';

describe('ContactsController (e2e)', () => {
  let app: INestApplication;
  let prisma: PrismaService;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();

    prisma = app.get(PrismaService);

    // Get auth token
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    authToken = loginRes.body.accessToken;
  });

  afterAll(async () => {
    await prisma.contact.deleteMany();
    await app.close();
  });

  describe('GET /contacts', () => {
    it('returns 200 with contact list', async () => {
      const res = await request(app.getHttpServer())
        .get('/contacts')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(Array.isArray(res.body)).toBe(true);
    });

    it('returns 401 without auth token', async () => {
      await request(app.getHttpServer()).get('/contacts').expect(401);
    });
  });

  describe('POST /contacts', () => {
    it('creates a contact and returns 201', async () => {
      const dto = { name: 'Test Corp', email: 'test@testcorp.com' };

      const res = await request(app.getHttpServer())
        .post('/contacts')
        .set('Authorization', `Bearer ${authToken}`)
        .send(dto)
        .expect(201);

      expect(res.body.name).toBe(dto.name);
      expect(res.body.email).toBe(dto.email);
      expect(res.body.id).toBeDefined();
    });

    it('returns 400 for invalid email', async () => {
      await request(app.getHttpServer())
        .post('/contacts')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Test', email: 'not-an-email' })
        .expect(400);
    });
  });
});
```

## E2E Tests (Playwright)

Playwright E2E tests run against a fully deployed stack (or local full-stack) and test real browser interactions.

### Configuration

```typescript
// apps/dashboard/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: process.env.PLAYWRIGHT_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Running E2E Tests

```bash
# Install Playwright browsers (first time only)
pnpm --filter @unicore/dashboard exec playwright install chromium

# Run E2E tests
pnpm --filter @unicore/dashboard test:e2e

# Run in headed mode (see the browser)
pnpm --filter @unicore/dashboard test:e2e --headed

# Run a specific test file
pnpm --filter @unicore/dashboard test:e2e -- e2e/contacts.spec.ts

# Open Playwright report
pnpm --filter @unicore/dashboard exec playwright show-report
```

### Writing E2E Tests

```typescript
// apps/dashboard/e2e/contacts.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Contacts module', () => {
  test.beforeEach(async ({ page }) => {
    // Login
    await page.goto('/login');
    await page.getByLabel('Email').fill('admin@unicore.dev');
    await page.getByLabel('Password').fill('admin123');
    await page.getByRole('button', { name: 'Login' }).click();
    await expect(page).toHaveURL('/dashboard');
  });

  test('displays the contacts list', async ({ page }) => {
    await page.goto('/contacts');
    await expect(page.getByRole('heading', { name: 'Contacts' })).toBeVisible();
    await expect(page.getByRole('table')).toBeVisible();
  });

  test('creates a new contact', async ({ page }) => {
    await page.goto('/contacts');
    await page.getByRole('button', { name: 'Add Contact' }).click();

    await page.getByLabel('Name').fill('Playwright Corp');
    await page.getByLabel('Email').fill('test@playwright.dev');
    await page.getByRole('button', { name: 'Create' }).click();

    await expect(page.getByText('Playwright Corp')).toBeVisible();
  });

  test('shows validation error for invalid email', async ({ page }) => {
    await page.goto('/contacts');
    await page.getByRole('button', { name: 'Add Contact' }).click();

    await page.getByLabel('Name').fill('Test');
    await page.getByLabel('Email').fill('not-an-email');
    await page.getByRole('button', { name: 'Create' }).click();

    await expect(page.getByText('Invalid email')).toBeVisible();
  });
});
```

### Page Object Model

For complex flows, use the Page Object Model (POM):

```typescript
// apps/dashboard/e2e/pages/LoginPage.ts
import { Page } from '@playwright/test';

export class LoginPage {
  constructor(private readonly page: Page) {}

  async goto(): Promise<void> {
    await this.page.goto('/login');
  }

  async login(email: string, password: string): Promise<void> {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Login' }).click();
  }

  async loginAsAdmin(): Promise<void> {
    await this.login('admin@unicore.dev', 'admin123');
  }
}
```

## CI/CD Integration

### GitHub Actions

Tests run automatically on all PRs and pushes to `main`:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: unicore
          POSTGRES_PASSWORD: unicore
          POSTGRES_DB: unicore
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 10.30
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm build
      - run: pnpm test --coverage
      - run: pnpm test:e2e
```

## Test Data Management

- Use `@faker-js/faker` for generating realistic test data
- Clean up test data in `afterAll` / `afterEach` hooks
- Use database transactions in integration tests for rollback-based cleanup:

```typescript
beforeEach(async () => {
  await prisma.$executeRaw`BEGIN`;
});

afterEach(async () => {
  await prisma.$executeRaw`ROLLBACK`;
});
```

## Related Documentation

- [Local setup guide](./local-setup.md)
- [Coding standards](./coding-standards.md)
- [Contributing guide](./contributing.md)

---

© 2026 BeMind Technology
