# Contributing to UniCore

Updated: 2026-03-22

Thank you for your interest in contributing to the UniCore platform. This guide covers the contribution process, branch conventions, commit standards, and pull request workflow.

## Code of Conduct

All contributors are expected to follow our Code of Conduct. Be respectful, constructive, and collaborative. See `CODE_OF_CONDUCT.md` in the root repository for the full policy.

## Repository Overview

UniCore uses a multi-repo structure organized as git submodules:

| Repository | Path | License |
|-----------|------|---------|
| `unicore-ecosystem` | Root workspace | BSL 1.1 |
| `unicore` | `unicore/` | BSL 1.1 |
| `unicore-pro` | `unicore-pro/` | BSL 1.1 |
| `unicore-license` | `unicore-license/` | Proprietary |
| `unicore-ecosystem-docs` | `unicore-ecosystem-docs/` | CC BY 4.0 |

Most community contributions will be to the `unicore` repository.

## Getting Started

### 1. Fork and Clone

```bash
# Fork the repo on GitHub, then clone your fork
git clone --recurse-submodules git@github.com:YOUR_USERNAME/unicore.git
cd unicore

# Add the upstream remote
git remote add upstream git@github.com:bemindlabs/unicore.git
```

### 2. Set Up Your Development Environment

See the [local setup guide](./local-setup.md) for complete instructions.

### 3. Create a Branch

Always work on a feature branch — never commit directly to `main`.

**Branch naming convention:**

```
<type>/<scope>-<short-description>
```

| Type | When to Use |
|------|------------|
| `feature/` | New feature or functionality |
| `fix/` | Bug fix |
| `docs/` | Documentation only |
| `refactor/` | Code restructuring (no behavior change) |
| `test/` | Adding or improving tests |
| `chore/` | Build, tooling, dependency updates |
| `hotfix/` | Critical production fix |

**Examples:**

```bash
git checkout -b feature/erp-bulk-import
git checkout -b fix/api-gateway-jwt-expiry
git checkout -b docs/rag-knowledge-base-guide
git checkout -b refactor/workflow-engine-kafka-consumer
```

## Commit Conventions

UniCore uses **Conventional Commits** format.

### Syntax

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation |
| `refactor` | Refactoring (no feature/fix) |
| `test` | Tests only |
| `chore` | Build, CI, tooling |
| `perf` | Performance improvement |
| `style` | Formatting, whitespace (no logic) |
| `ci` | CI/CD configuration |

### Scopes

Use the affected service or package as the scope:

```
dashboard, api-gateway, erp, ai-engine, rag, bootstrap, openclaw,
workflow, shared-types, config, ui, integrations
```

### Examples

```
feat(erp): add bulk CSV import for contacts
fix(api-gateway): resolve JWT refresh token expiry edge case
docs(rag): add knowledge base indexing guide
refactor(workflow): migrate Kafka consumer to KafkaJS 2.x API
test(erp): add integration tests for invoice creation
chore(deps): upgrade Prisma to 6.2.0
```

### Breaking Changes

Add `BREAKING CHANGE:` in the commit footer:

```
feat(api-gateway)!: change auth endpoint from /login to /auth/login

BREAKING CHANGE: The /login endpoint has been moved to /auth/login.
Update all clients to use the new path.
```

## Pull Request Process

### Before Opening a PR

1. Ensure all tests pass locally:
   ```bash
   pnpm test
   pnpm test:e2e  # if applicable
   ```

2. Ensure linting passes:
   ```bash
   pnpm lint
   pnpm typecheck
   ```

3. Update or add documentation for any new features

4. Rebase on the latest `main`:
   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

### PR Title and Description

Use the same Conventional Commits format for the PR title:

```
feat(erp): add bulk CSV import for contacts
```

In the description, include:
- **What** the change does
- **Why** it was needed
- **How** to test it
- Screenshots for UI changes
- Any breaking changes or migration steps

### Review Process

1. Submit the PR — GitHub Actions will run CI checks
2. At least 1 maintainer review is required for merge
3. Address all review comments in new commits (do not force-push during review)
4. Once approved, a maintainer will squash-merge into `main`

### CI Checks

All PRs must pass:
- TypeScript compilation (`pnpm typecheck`)
- ESLint (`pnpm lint`)
- Unit tests (`pnpm test`)
- Build (`pnpm build`)

## Code Review Guidelines

### For Authors

- Keep PRs focused — one feature or fix per PR
- Avoid large PRs over 500 lines of diff when possible
- Respond to review comments promptly
- Explain non-obvious decisions in comments

### For Reviewers

- Review for correctness, security, performance, and maintainability
- Be constructive — suggest alternatives, don't just reject
- Approve only when you are confident the change is ready
- Use GitHub's suggestion feature for small fixes

## Submodule Workflow

When contributing to a submodule (e.g., `unicore`):

```bash
# Work inside the submodule
cd unicore
git checkout -b feature/my-feature
# ... make changes ...
git add src/some-file.ts
git commit -m "feat(erp): add my feature"
git push origin feature/my-feature

# Open PR on github.com/bemindlabs/unicore

# After merge: update the root repo to track the new submodule commit
cd ..
git add unicore
git commit -m "chore: update submodule refs (unicore)"
```

## Reporting Issues

Use GitHub Issues for:
- Bug reports
- Feature requests
- Documentation improvements
- Security vulnerabilities (use the private security advisory for sensitive issues)

### Bug Report Template

```markdown
**Describe the bug**
A clear description of what the bug is.

**Steps to reproduce**
1. Go to '...'
2. Click on '...'
3. See error

**Expected behavior**
What you expected to happen.

**Environment**
- UniCore version:
- Edition: Community / Pro / Enterprise
- OS: Ubuntu 22.04 / macOS / Windows
- Docker version:
- Browser (if applicable):

**Logs**
```docker logs unicores-unicore-api-gateway-1```
```

## License

By contributing to UniCore, you agree that your contributions will be licensed under the Business Source License 1.1 (BSL 1.1) for code, and Creative Commons Attribution 4.0 International (CC BY 4.0) for documentation.

---

© 2026 BeMind Technology
