---
name: claude-code-config
description: Generate optimized CLAUDE.md files, memory configs, and project instructions for Claude Code. Analyzes codebase to create tailored AI context.
---

# Claude Code Config

> Your codebase has conventions. Claude Code should know them before it writes a single line.

CLAUDE.md is the instruction file that tells Claude Code how to work in your project. Without it, Claude operates on general knowledge alone -- it guesses your test runner, invents commit message formats, and picks naming conventions that clash with your existing code. A well-crafted CLAUDE.md eliminates that guesswork entirely. It loads at the start of every session and acts as persistent, authoritative context that shapes every response.

This skill analyzes your codebase -- tech stack, file structure, git history, testing patterns, and existing documentation -- then generates a complete CLAUDE.md tailored to how your team actually works.

## Table of Contents

- [When to Use](#when-to-use)
- [CLAUDE.md Anatomy](#claudemd-anatomy)
- [Codebase Analysis Workflow](#codebase-analysis-workflow)
- [Essential Sections to Include](#essential-sections-to-include)
- [Memory System Configuration](#memory-system-configuration)
- [Advanced Patterns](#advanced-patterns)
- [Templates](#templates)
- [Output Format](#output-format)

---

## When to Use

- **New project setup** -- You just initialized a repo and want Claude Code productive from day one
- **Onboarding Claude Code to an existing project** -- You have a codebase with established conventions that Claude keeps getting wrong
- **Inconsistent AI output** -- Claude Code produces code that doesn't match your patterns (wrong test framework, inconsistent naming, bad import order)
- **Team standardization** -- Multiple developers use Claude Code and you need everyone's AI assistant to follow the same rules
- **Project migration** -- Moving from Cursor, Copilot, or another AI tool and need to translate `.cursorrules` or `copilot-instructions.md` into CLAUDE.md format
- **After major refactors** -- Your architecture changed significantly and the existing CLAUDE.md (or lack of one) no longer reflects reality
- **Monorepo expansion** -- Adding a new package or service to a monorepo that needs its own scoped instructions

---

## CLAUDE.md Anatomy

### File Location Hierarchy

Claude Code loads instruction files from multiple locations. They cascade from general to specific, with more specific files taking priority.

```
~/.claude/CLAUDE.md                          # Global (all projects)
  └── /project-root/CLAUDE.md                # Project-wide
        └── /project-root/packages/api/CLAUDE.md   # Package-specific
```

### What Goes Where

| Location | Scope | Use For |
|----------|-------|---------|
| `~/.claude/CLAUDE.md` | Every project on this machine | Personal preferences: editor style, communication tone, commit conventions you always follow |
| `./CLAUDE.md` (project root) | This project, all contributors | Tech stack, architecture, test commands, code conventions, project-specific rules |
| `./packages/*/CLAUDE.md` | One package in a monorepo | Package-specific tooling, patterns that differ from the root config |
| `.claude/rules/*.md` | Triggered by file path globs | Scoped rules that load only when working with matching files |

### How Cascading Works

All matching CLAUDE.md files load simultaneously. They don't replace each other -- they stack. If your global config says "use conventional commits" and your project config says "prefix commits with ticket numbers," both rules apply.

**Priority when rules conflict:** More specific files win. A project CLAUDE.md overrides `~/.claude/CLAUDE.md`. A nested directory CLAUDE.md overrides the project root.

### Global CLAUDE.md Example

```markdown
# Global Preferences

- When I say "ship it," create a commit and push to the current branch
- Always explain trade-offs when suggesting architectural decisions
- Use concise commit messages, imperative mood, under 72 characters
- Never add AI co-author attribution to commits
- When writing tests, always include at least one edge case
```

### Scoped Rules Example

```yaml
# .claude/rules/api-routes.md
---
paths:
  - "src/api/**/*.ts"
  - "src/routes/**/*.ts"
---
- All API handlers must validate input with zod schemas
- Return consistent error format: { error: string, code: string, status: number }
- Always include rate limiting middleware on public endpoints
- Log request/response pairs at debug level
```

---

## Codebase Analysis Workflow

When generating a CLAUDE.md, follow these steps in order. Each step builds on the previous one.

### Step 1: Detect Tech Stack

Read package manifests and config files to identify the core stack.

```bash
# Node/JS/TS projects
cat package.json | jq '{name, scripts, dependencies: (.dependencies | keys), devDependencies: (.devDependencies | keys)}'

# Python projects
cat pyproject.toml 2>/dev/null || cat setup.py 2>/dev/null || cat requirements.txt

# Go projects
cat go.mod

# Rust projects
cat Cargo.toml
```

Record: language, framework, package manager, build tool, test runner, linter, formatter.

### Step 2: Identify Project Structure

```bash
# Top-level directory layout (excludes noise)
find . -maxdepth 2 -type d \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/.next/*' \
  -not -path '*/dist/*' \
  -not -path '*/__pycache__/*' \
  -not -path '*/venv/*' | sort

# Largest source files (often core modules)
find src/ -name "*.ts" -o -name "*.py" -o -name "*.go" | \
  xargs wc -l 2>/dev/null | sort -rn | head -20
```

Record: directory purpose (src/, lib/, tests/, scripts/), entry points, config file locations.

### Step 3: Extract Code Conventions

```bash
# Detect formatting config
ls .prettierrc* .eslintrc* biome.json .editorconfig ruff.toml pyproject.toml 2>/dev/null

# Check for import patterns (first 5 source files)
head -30 src/**/*.ts 2>/dev/null | grep -E "^import"

# Naming conventions -- look at exports
grep -rn "export.*function\|export.*const\|export.*class" src/ --include="*.ts" | head -20
```

Record: naming style (camelCase, snake_case), import order, formatting tools, lint rules.

### Step 4: Analyze Git History

```bash
# Recent commit message style
git log --oneline -20

# Branch naming pattern
git branch -r | head -10

# Commit message format (conventional commits? ticket prefixes?)
git log --format="%s" -50 | head -20
```

Record: commit format, branch naming convention, whether conventional commits are used.

### Step 5: Review Existing Docs

```bash
# Check what documentation already exists
ls README.md CONTRIBUTING.md docs/ .github/ 2>/dev/null

# Check for existing AI config files
ls CLAUDE.md .cursorrules .github/copilot-instructions.md AGENTS.md 2>/dev/null
```

Record: any existing conventions documented, migration candidates from other AI tools.

### Step 6: Generate CLAUDE.md

Combine all findings into a structured CLAUDE.md following the Essential Sections format below.

---

## Essential Sections to Include

Every generated CLAUDE.md should contain these sections. Skip a section only if it genuinely does not apply.

### Project Overview

One paragraph. What this project does, who uses it, and what matters most.

```markdown
## Project Overview

Skillfish is a marketplace and CLI for AI coding agent skills. The Next.js frontend
serves aicoach.pw, the API handles skill publishing and OAuth, and the CLI (`skillfish`)
lets developers install, bundle, and submit skills from their terminal. User trust and
API stability are the top priorities.
```

### Tech Stack

List every major technology. Be specific about versions when they matter.

```markdown
## Tech Stack

- **Runtime:** Node.js 20 LTS
- **Language:** TypeScript 5.4 (strict mode)
- **Framework:** Next.js 14 (App Router)
- **Database:** PostgreSQL 16 via Drizzle ORM
- **Auth:** Custom OAuth 2.0 with PKCE flow
- **Testing:** Vitest (unit), Playwright (e2e)
- **Package Manager:** pnpm 9
- **Deployment:** Vercel (frontend), Railway (API)
```

### Architecture

Explain the directory structure. Developers (and Claude) need to know where things live.

```markdown
## Architecture

```
src/
  app/           # Next.js App Router pages and layouts
  components/    # React components (colocated with pages when possible)
  lib/           # Shared utilities, database client, auth helpers
  api/           # API route handlers (REST, /api/v1/ prefix)
  hooks/         # Custom React hooks
  types/         # Shared TypeScript types and interfaces
scripts/         # Build and deployment scripts
tests/
  unit/          # Vitest unit tests (mirror src/ structure)
  e2e/           # Playwright end-to-end tests
```

- API routes follow `/api/v1/[resource]` pattern
- Components use named exports, one component per file
- Database queries live in `src/lib/db/queries/` -- never inline SQL in route handlers
```

### Code Conventions

Be prescriptive. Claude follows explicit rules better than vague suggestions.

```markdown
## Code Conventions

- **Naming:** camelCase for variables/functions, PascalCase for components/types, UPPER_SNAKE for constants
- **Files:** kebab-case for all filenames (`user-profile.ts`, not `userProfile.ts`)
- **Imports:** External packages first, then `@/lib`, then `@/components`, then relative. Blank line between groups.
- **Functions:** Prefer named function declarations over arrow functions for top-level exports
- **Types:** Define types in the same file unless shared across 3+ files, then move to `src/types/`
- **Error handling:** Always use custom AppError class, never throw raw strings
- **Comments:** Only for "why," never for "what." If code needs a "what" comment, refactor it.
```

### Testing

Tell Claude how to run tests and what patterns to follow.

```markdown
## Testing

### Commands
- `pnpm test` -- Run all unit tests
- `pnpm test:watch` -- Run in watch mode
- `pnpm test:coverage` -- Generate coverage report (minimum 80% required)
- `pnpm test:e2e` -- Run Playwright e2e tests
- `pnpm test:e2e --ui` -- Open Playwright UI mode

### Conventions
- Test files live next to source: `user-service.ts` -> `user-service.test.ts`
- Use `describe` blocks grouped by method/behavior, not by test type
- Factory functions for test data, never raw object literals
- Mock external services at the HTTP boundary with msw, not at the function level
- Every bug fix must include a regression test
```

### Git Conventions

Prevent Claude from inventing its own commit style.

```markdown
## Git Conventions

- **Commit format:** `type: short description` (conventional commits)
  - Types: feat, fix, chore, docs, refactor, test, ci
  - Example: `feat: add PKCE flow to OAuth login`
- **Branch naming:** `type/short-description` (e.g., `feat/oauth-pkce`, `fix/memory-leak`)
- **PR process:** One feature per PR. Include description and test plan.
- **Never** force push to `main`. Never amend shared commits.
- **Never** add AI co-author attribution to commits.
```

### Common Commands

The commands Claude will actually need to run.

```markdown
## Common Commands

```bash
pnpm dev              # Start dev server (port 3000)
pnpm build            # Production build
pnpm lint             # ESLint check
pnpm lint:fix         # Auto-fix lint issues
pnpm format           # Prettier format
pnpm db:migrate       # Run database migrations
pnpm db:seed          # Seed development data
pnpm db:studio        # Open Drizzle Studio
```
```

### Do's and Don'ts

Explicit guardrails prevent the most common AI mistakes.

```markdown
## Do's and Don'ts

### Do
- Use the existing `AppError` class for all error handling
- Use `@/lib/db` for database access, never import `pg` directly
- Run `pnpm lint` before suggesting a commit
- Use existing UI components from `src/components/ui/` before creating new ones
- Keep API responses under the existing envelope format: `{ data, meta, error }`

### Don't
- Don't use `any` type -- use `unknown` and narrow, or define a proper type
- Don't add new dependencies without confirming with the user first
- Don't modify migration files that have already been applied
- Don't use `console.log` in production code -- use the `logger` utility
- Don't create barrel files (`index.ts` that re-exports) -- import directly
- Don't use default exports -- always use named exports
```

---

## Memory System Configuration

### Directory Structure

Claude Code stores auto-memory in a per-project directory derived from the project path:

```
~/.claude/
  CLAUDE.md                              # Global instructions
  projects/
    -Users-you-projects-myapp/           # Path-encoded project directory
      memory/
        MEMORY.md                        # Auto-memory index (first 200 lines loaded)
        feedback_api_patterns.md         # Topic-specific overflow file
        feedback_testing.md              # Another topic file
        project_architecture.md          # Project-specific knowledge
      settings.json                      # Project-specific settings
```

### MEMORY.md Index Pattern

The MEMORY.md file serves as an index. Each entry is a brief note with an optional link to a deeper topic file.

```markdown
# Memory Index

- [feedback_api_patterns.md](feedback_api_patterns.md) -- Always validate request bodies with zod before processing
- [feedback_testing.md](feedback_testing.md) -- Use msw for HTTP mocking, not jest.mock
- [project_architecture.md](project_architecture.md) -- API routes are in src/api/, not src/app/api/
- User prefers concise explanations over verbose walkthroughs
- This project uses pnpm workspaces; never suggest npm or yarn commands
```

### Memory Types

| Type | Prefix | Purpose | Example |
|------|--------|---------|---------|
| **Feedback** | `feedback_` | Corrections from the user that should persist | `feedback_naming.md` -- "Use kebab-case for filenames, not camelCase" |
| **Project** | `project_` | Structural knowledge about the codebase | `project_architecture.md` -- "The auth module is in src/lib/auth/" |
| **Reference** | `reference_` | External information Claude looked up | `reference_drizzle_migrations.md` -- "Drizzle migration syntax notes" |
| **User** | (no prefix) | Personal preferences in MEMORY.md directly | "User prefers short explanations" |

### Setting Up Memory for a New Project

1. Let Claude Code create auto-memory naturally during your first few sessions
2. After 3-5 sessions, review what Claude has learned:
   ```
   cat ~/.claude/projects/-Users-you-projects-myapp/memory/MEMORY.md
   ```
3. Promote important patterns to CLAUDE.md (where they load in full, not truncated)
4. Remove stale or one-off entries to keep MEMORY.md under 200 lines
5. Use topic files for detailed reference material that doesn't need to load every session

---

## Advanced Patterns

### Monorepo Configuration

In a monorepo, use the root CLAUDE.md for shared conventions and per-package CLAUDE.md files for package-specific rules.

```
monorepo/
  CLAUDE.md                    # Shared: formatting, commit style, PR process
  packages/
    api/
      CLAUDE.md                # API-specific: route patterns, middleware, db access
    web/
      CLAUDE.md                # Frontend-specific: component patterns, state management
    shared/
      CLAUDE.md                # Shared lib: export rules, versioning policy
```

**Root CLAUDE.md (monorepo):**
```markdown
# Monorepo Root

## Structure
- `packages/api` -- Express API server
- `packages/web` -- Next.js frontend
- `packages/shared` -- Shared types and utilities

## Rules (apply everywhere)
- Use pnpm. Never npm or yarn.
- Import shared code as `@myapp/shared`, never with relative paths across packages.
- Run `pnpm -w lint` to lint everything before committing.
- Commit messages must reference the affected package: `feat(api): add rate limiting`
```

**Package CLAUDE.md (packages/api/CLAUDE.md):**
```markdown
# API Package

- Framework: Express 4 with TypeScript
- All routes registered in `src/routes/index.ts`
- Middleware order matters: auth -> validate -> rate-limit -> handler
- Database: Prisma ORM. Run `pnpm prisma migrate dev` for schema changes.
- Tests: `pnpm --filter api test`
```

### Environment-Specific Instructions

Use `.claude/rules/` for instructions that only apply in certain contexts.

```yaml
# .claude/rules/database.md
---
paths:
  - "src/db/**/*"
  - "prisma/**/*"
  - "drizzle/**/*"
---
- Never modify existing migration files. Always create new ones.
- Use transactions for operations that touch multiple tables.
- Add indexes for any column used in WHERE clauses with large tables.
- Test migrations against a local database before committing.
```

```yaml
# .claude/rules/react-components.md
---
paths:
  - "src/components/**/*.tsx"
  - "src/app/**/*.tsx"
---
- Use the existing design system components from `src/components/ui/`
- All components must accept a `className` prop for style overrides
- Use `cn()` utility from `src/lib/utils` for conditional class merging
- No inline styles. Use Tailwind classes exclusively.
- Client components must have "use client" directive at the top.
```

### Hook Configurations

Claude Code supports hooks that run before or after tool use. Configure them in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(npm install|npm add|yarn add)",
        "hook": "echo 'Use pnpm, not npm/yarn' && exit 1"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hook": "npx eslint --fix $CLAUDE_FILE_PATH 2>/dev/null || true"
      }
    ]
  }
}
```

Common hook use cases:
- **Block wrong package manager** -- Reject npm/yarn commands in a pnpm project
- **Auto-format on save** -- Run prettier/eslint after every file edit
- **Validate schemas** -- Run type checks after TypeScript file changes
- **Prevent secret leaks** -- Block writes to `.env` files

### MCP Server Integration Notes

If your project uses MCP (Model Context Protocol) servers, document them in CLAUDE.md so Claude knows what tools are available:

```markdown
## MCP Servers

This project connects to the following MCP servers (configured in .claude/settings.json):

- **database** -- Read-only access to production DB replica. Use `mcp__database__query` for lookups.
- **sentry** -- Error tracking. Use `mcp__sentry__get_issues` to check for related bugs before fixing.
- **linear** -- Project management. Check `mcp__linear__get_issue` for ticket context.

Never use MCP database access to modify data. Read-only queries only.
```

---

## Templates

### React / Next.js Project

```markdown
# Project Name

## Overview
[One paragraph: what this app does, who uses it, deployed where]

## Tech Stack
- **Framework:** Next.js 14 (App Router)
- **Language:** TypeScript (strict mode)
- **Styling:** Tailwind CSS 3.4
- **UI Components:** shadcn/ui
- **State:** React Server Components + Zustand for client state
- **Database:** PostgreSQL via Prisma ORM
- **Auth:** NextAuth.js v5
- **Testing:** Vitest (unit), Playwright (e2e)
- **Package Manager:** pnpm

## Architecture
```
app/                # Pages and layouts (App Router)
  (auth)/           # Auth-related pages (grouped route)
  (dashboard)/      # Dashboard pages
  api/              # API routes
components/
  ui/               # shadcn/ui primitives
  features/         # Feature-specific components
lib/
  db/               # Prisma client and queries
  auth/             # Auth configuration and helpers
  utils/            # Shared utilities
```

## Code Conventions
- Named exports only, no default exports
- File naming: kebab-case (`user-profile.tsx`)
- Component naming: PascalCase (`UserProfile`)
- Server Components by default; add `"use client"` only when needed
- Colocate components with their page when used in one place only
- Use `cn()` from `lib/utils` for Tailwind class merging

## Testing
- `pnpm test` -- unit tests
- `pnpm test:e2e` -- Playwright tests
- Test files: `*.test.ts(x)` next to source files
- Minimum 80% coverage on `lib/` directory

## Common Commands
```bash
pnpm dev            # Start dev server
pnpm build          # Production build
pnpm lint           # Run ESLint
pnpm db:push        # Push schema changes
pnpm db:studio      # Open Prisma Studio
```

## Don'ts
- Don't use `any` -- use `unknown` or define types
- Don't import from `@prisma/client` directly -- use `lib/db`
- Don't add dependencies without asking first
- Don't use CSS modules or styled-components -- Tailwind only
- Don't create API routes for data that Server Components can fetch directly
```

### Python / FastAPI Project

```markdown
# Project Name

## Overview
[One paragraph: what this API does, who consumes it]

## Tech Stack
- **Framework:** FastAPI 0.110
- **Language:** Python 3.12
- **Database:** PostgreSQL via SQLAlchemy 2.0 (async)
- **Migrations:** Alembic
- **Testing:** pytest + httpx (async test client)
- **Package Manager:** uv
- **Linting:** ruff
- **Type Checking:** mypy (strict)

## Architecture
```
src/
  api/
    v1/              # Versioned route modules
    deps.py          # Dependency injection (auth, db session)
  core/
    config.py        # Settings via pydantic-settings
    security.py      # JWT and auth utilities
  models/            # SQLAlchemy ORM models
  schemas/           # Pydantic request/response schemas
  services/          # Business logic (one service per domain)
  db/
    session.py       # Async engine and session factory
    migrations/      # Alembic migration scripts
tests/
  api/               # Route-level tests
  services/          # Service-level unit tests
  conftest.py        # Shared fixtures
```

## Code Conventions
- All functions must have type annotations (enforced by mypy)
- Naming: snake_case everywhere, PascalCase for classes
- Pydantic schemas for all request/response models -- never return raw dicts
- Services layer between routes and database -- routes must not contain business logic
- Use `Depends()` for all shared dependencies (auth, db, config)

## Testing
```bash
uv run pytest                    # Run all tests
uv run pytest -x                 # Stop on first failure
uv run pytest --cov=src          # Coverage report
uv run pytest tests/api/         # API tests only
```
- Use `async def test_*` with httpx.AsyncClient
- Fixtures in conftest.py for db sessions, auth tokens, test data
- Always test both success and error paths

## Common Commands
```bash
uv run fastapi dev               # Start dev server with reload
uv run alembic upgrade head      # Run migrations
uv run alembic revision --autogenerate -m "description"  # New migration
uv run ruff check src/           # Lint
uv run ruff format src/          # Format
uv run mypy src/                 # Type check
```

## Don'ts
- Don't use `from sqlalchemy import *` -- import specific types
- Don't put business logic in route handlers -- use services
- Don't use synchronous database calls -- always async
- Don't modify existing Alembic migrations -- create new ones
- Don't hardcode config values -- use `core/config.py` settings
```

### Go Microservice

```markdown
# Service Name

## Overview
[One paragraph: what this service does, what it communicates with]

## Tech Stack
- **Language:** Go 1.22
- **HTTP:** Standard library net/http + chi router
- **Database:** PostgreSQL via pgx
- **Config:** envconfig
- **Testing:** Standard library testing + testify assertions
- **Containerization:** Docker multi-stage build

## Architecture
```
cmd/
  server/           # Application entry point (main.go)
internal/
  handler/          # HTTP handlers (one file per resource)
  service/          # Business logic layer
  repository/       # Database access layer
  model/            # Domain types
  middleware/        # HTTP middleware (auth, logging, recovery)
  config/           # Environment configuration struct
pkg/
  httperr/          # Shared HTTP error types
  validate/         # Input validation helpers
migrations/         # SQL migration files (golang-migrate)
```

## Code Conventions
- Follow standard Go project layout: `cmd/`, `internal/`, `pkg/`
- Handler -> Service -> Repository layering (no skipping layers)
- Errors: wrap with `fmt.Errorf("operation: %w", err)`, never discard
- Context: pass `context.Context` as first parameter to all service/repo methods
- Interfaces: define at the consumer, not the producer
- Naming: short variable names in small scopes (`r` for request, `w` for writer)

## Testing
```bash
go test ./...                    # All tests
go test ./internal/service/...   # Service tests only
go test -race ./...              # Race condition detection
go test -cover ./...             # Coverage
```
- Table-driven tests for handlers and services
- Use testify `assert` and `require`, not manual if-checks
- Test database with dockertest or testcontainers-go

## Common Commands
```bash
go run cmd/server/main.go        # Run locally
go build -o bin/server cmd/server/main.go  # Build binary
migrate -path migrations/ -database $DB_URL up  # Run migrations
golangci-lint run                # Lint
```

## Don'ts
- Don't use global variables for state -- pass dependencies via constructor
- Don't use `init()` functions -- explicit initialization in main
- Don't return concrete types from constructors -- return interfaces
- Don't use `panic` for error handling -- return errors
- Don't log and return an error -- do one or the other
```

### Monorepo

```markdown
# Monorepo Name

## Overview
[One paragraph: what products this monorepo contains]

## Tech Stack
- **Monorepo Tool:** Turborepo
- **Package Manager:** pnpm workspaces
- **Shared Language:** TypeScript (strict, project references)

## Structure
```
apps/
  web/              # Next.js customer-facing app
  admin/            # Next.js internal admin dashboard
  api/              # Express API server
packages/
  ui/               # Shared React component library
  config/           # Shared ESLint, TypeScript, Tailwind configs
  db/               # Database client, schema, migrations
  types/            # Shared TypeScript types
```

## Rules (Global)
- Use pnpm exclusively. Never npm or yarn.
- Import shared packages by name (`@myapp/ui`), never relative paths across packages
- Commit messages: `type(scope): description` where scope is the package name
- Run `pnpm turbo lint` before committing
- Changes to `packages/db` schema require migration review from backend team

## Per-Package Commands
```bash
pnpm --filter web dev            # Start web app
pnpm --filter api dev            # Start API server
pnpm --filter ui storybook       # Component docs
pnpm turbo build                 # Build everything
pnpm turbo test                  # Test everything
pnpm turbo lint                  # Lint everything
```

## Don'ts
- Don't install dependencies in the root -- always scope to a package
- Don't create circular dependencies between packages
- Don't bypass the `packages/db` layer -- no direct SQL in apps
- Don't duplicate types -- if it's shared, it belongs in `packages/types`
```

---

## Output Format

A generated CLAUDE.md should:

1. **Start with a project overview** -- one paragraph, no preamble
2. **Use H2 (`##`) sections** -- Claude parses these as distinct instruction blocks
3. **Be prescriptive, not descriptive** -- "Use pnpm" not "This project uses pnpm"
4. **Include runnable commands** -- exact commands, not descriptions of what to run
5. **Stay under 500 lines** -- Claude loads the full file every session; bloat wastes context
6. **Avoid redundancy** -- don't repeat what's in `package.json` unless it needs emphasis
7. **Use bullet points for rules** -- easier for Claude to parse than prose paragraphs
8. **Include concrete examples** -- show the right import order, the right test structure, the right commit format

### Quality Checklist

Before finalizing a generated CLAUDE.md, verify:

- [ ] Project overview is accurate and concise
- [ ] Tech stack lists all major technologies with versions
- [ ] Architecture section matches actual directory structure
- [ ] All listed commands actually work (`pnpm test`, `pnpm build`, etc.)
- [ ] Code conventions match what's in existing source files
- [ ] Git conventions match the actual commit history
- [ ] Do's and Don'ts address real mistakes (check git log for reverted commits)
- [ ] No sensitive information (API keys, internal URLs, credentials)
- [ ] File is under 500 lines
- [ ] Sections are ordered from most-referenced to least-referenced
