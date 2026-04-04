---
name: cursor-rules-engineer
description: Create optimized .cursorrules and AI editor configuration files from codebase analysis. Boost AI coding assistant accuracy and consistency.
---

# Cursor Rules Engineer

Well-crafted AI editor rules are the single highest-leverage investment you can make in AI-assisted development. A good rules file turns a generic coding assistant into a team member that knows your stack, follows your conventions, and produces code that passes review on the first try. This skill analyzes any codebase and generates production-grade rule files for Cursor, Claude Code, GitHub Copilot, Windsurf, and other AI editors.

## Table of Contents

- [When to Use](#when-to-use)
- [Rule File Formats](#rule-file-formats)
- [Codebase Analysis Workflow](#codebase-analysis-workflow)
- [Rule Categories and Templates](#rule-categories-and-templates)
- [Advanced Techniques](#advanced-techniques)
- [Quality Validation](#quality-validation)
- [Example Rules Library](#example-rules-library)
- [Output Format](#output-format)

---

## When to Use

Trigger this skill when any of these conditions apply:

- **New project setup** — Bootstrapping a repo and want AI assistants to follow your conventions from day one
- **Onboarding an AI assistant** — First time using Cursor, Claude Code, Copilot, or Windsurf on an existing codebase
- **Inconsistent AI output** — The AI keeps generating code that doesn't match your patterns (wrong imports, different naming conventions, missing error handling)
- **Framework migration** — Moving from Pages Router to App Router, from REST to GraphQL, or from JavaScript to TypeScript and need rules to reflect the new patterns
- **Team scaling** — Sharing AI configuration across developers so everyone gets consistent suggestions
- **Post-code-review friction** — Repeated review comments about the same issues that rules could prevent
- **Monorepo onboarding** — Different packages need different rules and the AI keeps mixing conventions

---

## Rule File Formats

AI editors each use their own configuration format. Generate rules for whichever editors your team uses.

### Cursor

| File / Path | Scope | Notes |
|---|---|---|
| `.cursorrules` | Project root, applies globally | Legacy format, still widely supported |
| `.cursor/rules/*.md` | Per-directory or per-topic rules | Modern format, supports granular rules with frontmatter |
| `.cursor/rules/*.mdc` | Same as above | MDC format with YAML frontmatter for metadata |

**Modern .cursor/rules/ format with frontmatter:**

```markdown
---
description: React component conventions for this project
globs: ["src/components/**/*.tsx", "src/features/**/*.tsx"]
alwaysApply: false
---

# React Component Rules

- Use functional components with arrow function syntax
- Props interface named `{ComponentName}Props` and exported
- One component per file, filename matches component name
```

### Claude Code

| File / Path | Scope | Notes |
|---|---|---|
| `CLAUDE.md` | Project root, loaded automatically | Primary instruction file for Claude Code |
| `CLAUDE.md` in subdirectories | Scoped to that directory | Claude Code merges all CLAUDE.md files in the path |

### GitHub Copilot

| File / Path | Scope | Notes |
|---|---|---|
| `.github/copilot-instructions.md` | Repository-wide | Loaded automatically by Copilot |
| `.github/copilot/` directory | Per-topic instruction files | Supports multiple files for organization |

### Windsurf

| File / Path | Scope | Notes |
|---|---|---|
| `.windsurfrules` | Project root | Similar format to .cursorrules |

### Aider

| File / Path | Scope | Notes |
|---|---|---|
| `.aider.conf.yml` | Project root | YAML configuration with conventions section |
| `CONVENTIONS.md` | Project root | Referenced from aider config |

### Universal Strategy

For teams using multiple editors, maintain a single source of truth and generate editor-specific files:

```
ai-rules/
  base-rules.md          # Shared rules (editor-agnostic)
  cursor-rules.md        # Cursor-specific additions
  claude-rules.md        # Claude Code-specific additions
  copilot-rules.md       # Copilot-specific additions
scripts/
  generate-ai-rules.sh   # Compiles base + editor-specific into each format
```

---

## Codebase Analysis Workflow

Follow these steps to analyze a codebase and generate accurate rules.

### Step 1: Identify Tech Stack

```bash
# Detect package manager and dependencies
cat package.json 2>/dev/null | jq '{name, scripts: .scripts, deps: (.dependencies // {} | keys), devDeps: (.devDependencies // {} | keys)}'

# Python projects
cat pyproject.toml 2>/dev/null || cat requirements.txt 2>/dev/null | head -30

# Go projects
cat go.mod 2>/dev/null | head -20

# Check for framework markers
ls -la next.config.* nuxt.config.* vite.config.* webpack.config.* tsconfig.json .eslintrc* .prettierrc* tailwind.config.* 2>/dev/null
```

### Step 2: Detect Project Structure

```bash
# Directory layout (top 3 levels, ignoring noise)
find . -maxdepth 3 \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/.next/*' \
  -not -path '*/dist/*' \
  -not -path '*/__pycache__/*' \
  -not -path '*/venv/*' \
  | sort | head -80

# Identify source roots
ls -d src/ app/ lib/ pkg/ internal/ cmd/ pages/ components/ 2>/dev/null
```

### Step 3: Extract Naming Conventions

```bash
# File naming patterns
find src/ -name "*.ts" -o -name "*.tsx" | head -30 | xargs -I{} basename {}

# Component naming
grep -rn "export default function\|export const\|export function" src/components/ --include="*.tsx" | head -20

# Variable and function naming style
grep -rn "const [a-z]" src/ --include="*.ts" | head -10
grep -rn "function [a-z]" src/ --include="*.ts" | head -10
```

### Step 4: Identify Patterns and Conventions

```bash
# Import style (relative vs alias)
grep -rn "^import" src/ --include="*.ts" --include="*.tsx" | head -20

# Error handling patterns
grep -rn "try {" src/ --include="*.ts" | head -10
grep -rn "catch\|\.catch\|throw new" src/ --include="*.ts" | head -10

# API patterns
grep -rn "fetch\|axios\|api\." src/ --include="*.ts" | head -15

# State management
grep -rn "useState\|useReducer\|zustand\|redux\|jotai\|recoil" src/ --include="*.tsx" | head -10
```

### Step 5: Extract Testing Conventions

```bash
# Test file location and naming
find . -name "*.test.*" -o -name "*.spec.*" -o -name "*_test.*" | head -20

# Test framework and patterns
grep -rn "describe\|it(\|test(\|expect(" --include="*.test.*" | head -15

# Test utilities
grep -rn "render\|screen\|fireEvent\|userEvent\|waitFor" --include="*.test.*" | head -10
```

### Step 6: Compile and Generate Rules

Synthesize findings from Steps 1-5 into a structured rules file. Each finding maps to a rule category. Prioritize rules that address the most common patterns -- these have the highest impact on AI output quality.

---

## Rule Categories and Templates

### Code Style Rules

```markdown
## Code Style

### Naming Conventions
- Variables and functions: camelCase
- React components: PascalCase
- Constants: UPPER_SNAKE_CASE
- Types and interfaces: PascalCase, prefix interfaces with nothing (no "I" prefix)
- Files: kebab-case for utilities, PascalCase for components
- Test files: `{filename}.test.ts` colocated with source

### Formatting
- Use single quotes for strings
- Trailing commas in multiline structures
- 2-space indentation
- Max line length: 100 characters
- Semicolons: always

### Imports
- Group imports: 1) external packages, 2) internal aliases (@/), 3) relative imports
- Sort alphabetically within each group
- Use path aliases: `@/components`, `@/lib`, `@/hooks`
- Never use relative paths that traverse more than one level (no `../../`)
- Prefer named exports over default exports
```

### Architecture Rules

```markdown
## Architecture

### File Structure
- Feature-based organization: `src/features/{feature-name}/`
- Each feature contains: components/, hooks/, utils/, types.ts, index.ts
- Shared code lives in `src/shared/` with the same subdirectory structure
- API route handlers in `src/app/api/` follow RESTful resource naming

### Component Patterns
- Smart/container components in `features/{name}/` — handle data fetching and state
- Presentational components in `features/{name}/components/` — receive props, no side effects
- Page components are thin wrappers that compose feature components
- Maximum component size: ~150 lines; extract subcomponents beyond that

### API Conventions
- All API responses follow: `{ data: T, error: null } | { data: null, error: { code, message } }`
- Use Zod schemas for request validation at the API boundary
- API clients return typed responses, never raw fetch results
- Server actions for mutations, route handlers for GET endpoints
```

### Framework-Specific Rules

**React / Next.js (App Router):**

```markdown
## Next.js App Router Rules

### Server vs Client Components
- Components are Server Components by default — do NOT add "use client" unless required
- Add "use client" only when using: useState, useEffect, useRef, event handlers, browser APIs
- Data fetching happens in Server Components or Server Actions, never in client components
- Keep "use client" boundaries as low in the tree as possible

### Data Fetching
- Use async Server Components for data fetching, not useEffect
- Server Actions for mutations (forms, writes)
- Revalidation: use `revalidatePath()` or `revalidateTag()` after mutations
- Never call Route Handlers from Server Components — access the data source directly

### Routing
- Use file-based routing: `app/{route}/page.tsx`
- Dynamic routes: `app/{route}/[id]/page.tsx`
- Loading states: `loading.tsx` in each route segment
- Error boundaries: `error.tsx` in each route segment
- Layouts: `layout.tsx` for shared UI, do not re-render on navigation
```

**Python / FastAPI:**

```markdown
## FastAPI Rules

### Project Structure
- Routers in `app/routers/` — one file per resource
- Schemas in `app/schemas/` — Pydantic models for request/response
- Models in `app/models/` — SQLAlchemy/SQLModel ORM models
- Services in `app/services/` — business logic, never in routers
- Dependencies in `app/deps.py` — database sessions, auth, common deps

### Patterns
- All endpoints return Pydantic response models, never raw dicts
- Use `Depends()` for dependency injection (auth, db session, pagination)
- Async endpoints for I/O-bound operations
- Background tasks via `BackgroundTasks` parameter, not inline
- HTTPException for error responses with appropriate status codes

### Naming
- Router files: plural resource name (`users.py`, `orders.py`)
- Schema classes: `{Resource}Create`, `{Resource}Update`, `{Resource}Response`
- Endpoint functions: `get_{resource}`, `list_{resources}`, `create_{resource}`
```

**Go Microservice:**

```markdown
## Go Rules

### Project Layout (Standard)
- `cmd/{service-name}/main.go` — entrypoint
- `internal/` — private application code not importable by other projects
- `internal/handler/` — HTTP/gRPC handlers
- `internal/service/` — business logic
- `internal/repository/` — data access layer
- `pkg/` — library code safe for external use

### Patterns
- Accept interfaces, return structs
- Errors are values: always check and propagate errors, never ignore
- Use `fmt.Errorf("operation: %w", err)` for error wrapping
- Context as first parameter: `func DoThing(ctx context.Context, ...) error`
- Table-driven tests with `t.Run()` subtests
- No init() functions — explicit initialization in main

### Naming
- Package names: short, lowercase, no underscores (`handler` not `http_handler`)
- Interfaces: verb-er pattern (`Reader`, `Validator`, `Store`)
- Unexported by default — only export what is part of the public API
```

### Testing Rules

```markdown
## Testing

### Structure
- Test files colocated with source: `feature.test.ts` next to `feature.ts`
- Integration tests in `tests/integration/`
- E2E tests in `tests/e2e/` or `e2e/`

### Patterns
- Arrange-Act-Assert pattern for all unit tests
- Descriptive test names: `it("should return 404 when user does not exist")`
- One assertion per test when practical
- Use factories/builders for test data, not inline object literals
- Mock external services at the boundary (API clients, database), not internal modules
- Never test implementation details — test behavior and outputs

### Coverage
- New features require tests — do not submit without them
- Bug fixes require a regression test that fails before the fix
```

### Security Rules

```markdown
## Security

- NEVER hardcode secrets, API keys, tokens, or passwords in source code
- Use environment variables via `.env.local` (never committed) and a validation layer
- All user input must be validated with Zod/Pydantic before use
- SQL queries use parameterized queries or ORM methods — never string concatenation
- API endpoints require authentication unless explicitly marked as public
- File uploads: validate MIME type, enforce size limits, sanitize filenames
- CORS: explicit origin allowlist, never wildcard in production
- Dependencies: run `npm audit` / `pip audit` before merging dependency updates
```

### Documentation Rules

```markdown
## Documentation

### Code Comments
- Do not comment obvious code — comments explain "why", not "what"
- Use JSDoc/TSDoc for all exported functions and types:
  ```typescript
  /** Calculates the pro-rated subscription amount for a mid-cycle upgrade. */
  export function calculateProration(current: Plan, next: Plan, daysRemaining: number): number
  ```
- Python: Google-style docstrings for all public functions
- TODOs must include a ticket reference: `// TODO(PROJ-123): migrate to new API`

### API Documentation
- All API endpoints documented with request/response examples
- Error responses documented with status codes and error codes
- Breaking changes noted in CHANGELOG.md
```

---

## Advanced Techniques

### Conditional Rules with Globs

In Cursor's modern `.cursor/rules/` format, use frontmatter globs to apply rules only to matching files:

```markdown
---
description: Database migration conventions
globs: ["**/migrations/**", "**/db/migrate/**"]
alwaysApply: false
---

# Migration Rules

- One migration per schema change
- Migrations must be reversible (include down/rollback)
- Never modify a migration that has been deployed to production
- Use descriptive names: `20240115_add_stripe_customer_id_to_users`
```

```markdown
---
description: API route handler conventions
globs: ["src/app/api/**/*.ts"]
alwaysApply: false
---

# API Handler Rules

- Validate request body with Zod schema at the top of the handler
- Return consistent response shape: { data, error, meta }
- Log all errors before returning error responses
- Include request ID in error responses for debugging
```

### Project-Specific Overrides

Layer rules for different contexts within the same repo:

```markdown
---
description: Overrides for legacy modules being migrated
globs: ["src/legacy/**"]
alwaysApply: false
---

# Legacy Code Rules

- Do NOT refactor legacy code while fixing bugs — create a separate PR
- Legacy modules use CommonJS require() — match existing style
- Add TypeScript types gradually via JSDoc annotations, not .ts conversion
- New tests for legacy code go in `tests/legacy/` using the existing test helpers
```

### Monorepo Rules

For monorepos, place rules at multiple levels:

```
.cursorrules                          # Shared rules for the entire repo
packages/web/.cursor/rules/           # Frontend-specific rules
packages/api/.cursor/rules/           # Backend-specific rules
packages/shared/.cursor/rules/        # Shared library rules
```

**Root-level monorepo rules:**

```markdown
## Monorepo Rules

- Import shared packages via workspace alias: `@acme/shared`, `@acme/ui`
- Never import directly from another package's `src/` — use the package's public API
- Changes to `packages/shared/` require running tests in all dependent packages
- Each package manages its own dependencies in its own package.json
- TypeScript project references for cross-package type checking
```

### Team-Shared Rules

Store rules in the repository so the entire team benefits:

```bash
# Commit AI rules to the repo
git add .cursorrules .cursor/rules/ CLAUDE.md .github/copilot-instructions.md
git commit -m "chore: add AI editor rules for project conventions"
```

Add a section to your CONTRIBUTING.md:

```markdown
## AI Editor Rules

This repo includes configuration files for AI coding assistants.
When you update a convention or pattern, update the corresponding rule file:

- `.cursor/rules/` — Cursor rules (glob-scoped)
- `CLAUDE.md` — Claude Code instructions
- `.github/copilot-instructions.md` — GitHub Copilot instructions
```

### Rule Inheritance and Composition

Build complex rule sets from composable pieces:

```
.cursor/rules/
  01-code-style.md        # Naming, formatting, imports
  02-architecture.md      # File structure, patterns
  03-react.md             # React/Next.js specific
  04-testing.md           # Test conventions
  05-security.md          # Security requirements
  06-api.md               # API design rules
  07-database.md          # Database and migration rules
```

Number-prefix files to control loading order. Keep each file focused on one concern for easy maintenance.

---

## Quality Validation

### Testing Rules Against Real Prompts

After generating rules, validate them with representative prompts:

1. **Component generation test** — Ask the AI to create a new component and verify it follows naming, file structure, and pattern rules
2. **API endpoint test** — Ask the AI to add a new endpoint and check response format, validation, and error handling
3. **Refactoring test** — Ask the AI to refactor existing code and confirm it maintains conventions
4. **Bug fix test** — Give the AI a bug and verify the fix follows testing rules (adds regression test)
5. **New feature test** — Request a complete feature to test end-to-end convention adherence

### Measuring AI Output Consistency

Track these signals to evaluate rule effectiveness:

| Signal | How to Measure | Target |
|---|---|---|
| First-pass acceptance | % of AI code accepted without edits | > 80% |
| Convention violations | Manual tally per PR | < 2 per PR |
| Rule-covered patterns | % of team conventions captured in rules | > 90% |
| Lint/format pass rate | % of AI code that passes linting without fixes | > 95% |

### Iterative Refinement Loop

```
1. Generate initial rules from codebase analysis
2. Use AI assistant for 1-2 days of normal work
3. Track every correction you make to AI output
4. Categorize corrections:
   a. Missing rule — add it
   b. Ambiguous rule — clarify it
   c. Conflicting rules — resolve and simplify
   d. Edge case — add a specific example
5. Update rules and repeat from step 2
```

The best rule files are living documents refined over 3-4 iterations. Initial rules get you 70% accuracy; two rounds of refinement push it past 90%.

---

## Example Rules Library

### Complete React / Next.js Project

```markdown
# Project Rules — Acme Web App

## Tech Stack
- Next.js 14 (App Router), React 18, TypeScript 5
- Styling: Tailwind CSS + shadcn/ui components
- State: Zustand for client state, React Query for server state
- Database: Prisma ORM with PostgreSQL
- Auth: NextAuth.js v5
- Testing: Vitest + React Testing Library + Playwright

## Code Style
- TypeScript strict mode — no `any` types, no `@ts-ignore`
- Prefer `type` over `interface` unless declaration merging is needed
- Use path alias `@/` for all imports from `src/`
- Named exports only — no default exports except for Next.js pages
- Destructure props in function signature: `function Button({ label, onClick }: ButtonProps)`

## Components
- Server Components by default — add "use client" only when required
- Props type: `type {Name}Props = { ... }` exported above the component
- Colocate styles: component-specific Tailwind classes, no separate CSS files
- Use shadcn/ui primitives for forms, dialogs, dropdowns — do not build from scratch
- Loading states: use Suspense boundaries with skeleton components

## Data Fetching
- Server Components fetch data directly via Prisma or service functions
- Client-side data: React Query with typed query keys
- Mutations: Server Actions in `actions.ts` files colocated with the feature
- Revalidate after mutation: `revalidatePath()` at the end of every Server Action

## API Routes
- Path: `app/api/v1/{resource}/route.ts`
- Validate with Zod: `const body = schema.parse(await req.json())`
- Response shape: `NextResponse.json({ data } | { error: { code, message } })`
- Auth: check session via `auth()` at the top of every handler

## Testing
- Unit tests: `*.test.ts` colocated, run with `vitest`
- Component tests: render with Testing Library, assert on visible text/roles
- E2E: Playwright tests in `e2e/`, one spec per user flow
- Test data: use factories in `tests/factories/`

## Git
- Branch naming: `feat/`, `fix/`, `chore/`, `refactor/` prefixes
- Commit messages: conventional commits (`feat:`, `fix:`, `chore:`)
- PRs require passing CI and one approval
```

### Complete Python FastAPI Project

```markdown
# Project Rules — Acme API

## Tech Stack
- Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2
- Database: PostgreSQL with Alembic migrations
- Auth: JWT with refresh token rotation
- Testing: pytest + httpx (async test client)
- Linting: ruff, type checking: mypy (strict)

## Project Structure
```
app/
  main.py              # FastAPI app factory
  config.py            # Settings via pydantic-settings
  deps.py              # Dependency providers
  routers/             # One file per resource
  schemas/             # Pydantic request/response models
  models/              # SQLAlchemy ORM models
  services/            # Business logic
  repositories/        # Data access (queries)
```

## Naming
- Files: snake_case (`user_router.py`, `order_schema.py`)
- Classes: PascalCase (`UserCreate`, `OrderRepository`)
- Functions: snake_case (`get_user_by_email`)
- Constants: UPPER_SNAKE_CASE (`MAX_RETRY_COUNT`)
- Router prefixes: plural nouns (`/api/v1/users`, `/api/v1/orders`)

## Patterns
- Thin routers: validate input, call service, return response — no business logic
- Services receive repository via dependency injection
- All database access through repository layer — never import models in routers
- Async all the way: async def for endpoints, async session, async queries
- Use `Annotated[Dep, Depends(provider)]` syntax for dependency injection

## Error Handling
- Raise `HTTPException` only in routers, never in services
- Services raise custom domain exceptions (`UserNotFoundError`, `InsufficientFundsError`)
- Router catches domain exceptions and maps to HTTP status codes
- Global exception handler for unhandled errors returns 500 with request ID

## Testing
- Test files: `tests/test_{module}.py`
- Use `httpx.AsyncClient` with `app` for integration tests
- Factory functions for test data in `tests/factories.py`
- Each test function is independent — no shared mutable state
- Use `pytest.mark.asyncio` for async tests

## Security
- Validate all input with Pydantic schemas — never use raw dict from request
- Rate limiting on auth endpoints
- CORS: explicit origin allowlist from environment variable
- SQL: use ORM or parameterized queries — never f-strings for SQL
```

### Complete Go Microservice

```markdown
# Project Rules — Acme Order Service

## Tech Stack
- Go 1.22, standard library net/http + chi router
- Database: PostgreSQL with pgx driver
- Config: envconfig for environment variable parsing
- Testing: standard testing package + testify assertions
- Observability: OpenTelemetry traces, structured logging with slog

## Project Layout
```
cmd/orders/main.go           # Entrypoint
internal/
  handler/                   # HTTP handlers
  service/                   # Business logic
  repository/                # Data access
  model/                     # Domain types
  middleware/                 # HTTP middleware
pkg/
  httputil/                  # Response helpers
  validate/                  # Input validation
```

## Patterns
- Accept interfaces, return structs
- Constructor functions: `NewOrderService(repo OrderRepository) *OrderService`
- Error wrapping: `fmt.Errorf("create order: %w", err)`
- Context propagation: every function that does I/O takes `ctx context.Context` as first param
- No global state — all dependencies injected via constructor

## Error Handling
- Check every error immediately — never use `_` to ignore errors
- Wrap errors with context at each layer
- Handlers map domain errors to HTTP status codes via errors.Is/As
- Use sentinel errors for expected conditions: `var ErrOrderNotFound = errors.New("order not found")`
- Log at the handler layer only — services return errors, do not log

## HTTP
- Handlers: `func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request)`
- Decode request: `json.NewDecoder(r.Body).Decode(&req)` with limit reader
- Encode response: use `httputil.JSON(w, statusCode, data)` helper
- Middleware chain: logging -> tracing -> auth -> rate-limit -> handler

## Testing
- Table-driven tests: `tests := []struct{ name string; ... }{ ... }`
- Use `t.Run(tt.name, func(t *testing.T) { ... })` for subtests
- Repository tests use a real test database (Docker via testcontainers)
- Handler tests use `httptest.NewRecorder()` and real handler (no mocks)
- Aim for behavior tests, not implementation tests

## Naming
- Packages: short, single word (`handler`, `order`, `auth`)
- Interfaces: `-er` suffix (`OrderStore`, `TokenValidator`)
- Exported only if used outside the package
- No stuttering: `order.Service` not `order.OrderService`
```

### Full-Stack Monorepo

```markdown
# Project Rules — Acme Platform Monorepo

## Structure
```
packages/
  web/        # Next.js frontend (see React/Next.js rules)
  api/        # FastAPI backend (see FastAPI rules)
  shared/     # Shared TypeScript types and utilities
  ui/         # Shared React component library
  config/     # Shared ESLint, TypeScript, Tailwind configs
```

## Monorepo Rules
- Package manager: pnpm with workspace protocol
- Import shared packages via `@acme/shared`, `@acme/ui`, `@acme/config`
- Never reference another package by relative path — use the workspace alias
- Each package has its own `tsconfig.json` extending `@acme/config/tsconfig.base`
- Changes to `@acme/shared` require running tests in both `web` and `api`

## Cross-Package Conventions
- API types defined once in `@acme/shared/types/api.ts` and imported everywhere
- Error codes are string enums shared between frontend and backend
- Date handling: store as ISO 8601 strings, parse with date-fns on the frontend
- IDs: UUIDs as strings, generated server-side

## CI/CD
- Turborepo for build orchestration — respect the dependency graph
- Each package has: `build`, `test`, `lint`, `typecheck` scripts
- CI runs only affected packages on PR (turbo --filter=...[origin/main])
- Deploy: web to Vercel, api to Cloud Run, triggered by changes to respective packages
```

---

## Output Format

When this skill runs, produce the following deliverables:

### Primary Output

1. **Analysis Summary** — Bullet list of detected tech stack, patterns, and conventions
2. **Rule Files** — Complete, ready-to-commit files for the requested editors:
   - `.cursorrules` or `.cursor/rules/*.md` (Cursor)
   - `CLAUDE.md` (Claude Code)
   - `.github/copilot-instructions.md` (GitHub Copilot)
   - `.windsurfrules` (Windsurf)
3. **Validation Prompts** — 5 test prompts to verify the rules produce correct output

### File Structure

```
# Generated files (commit these to your repo)
.cursorrules                           # or .cursor/rules/ directory
CLAUDE.md                              # if using Claude Code
.github/copilot-instructions.md        # if using Copilot
.windsurfrules                         # if using Windsurf
```

### Formatting Requirements

- Rules files must be valid Markdown
- Each rule is a single imperative sentence (do this, never do that)
- Group rules under clear headings
- Include concrete examples for any rule that could be ambiguous
- Keep total file size under 4,000 tokens for optimal AI context usage
- Prioritize rules by impact: conventions violated most often go first
