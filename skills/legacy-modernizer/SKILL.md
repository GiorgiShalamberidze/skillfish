---
name: legacy-modernizer
description: Legacy code modernization: strangler fig pattern, incremental migration strategies, dependency upgrades, framework migration paths, and rewrite risk assessment.
---

# Legacy Modernizer

Systematic approach to modernizing legacy codebases without big-bang rewrites. Covers assessment, incremental migration strategies, dependency management, framework transitions, database evolution, API versioning, testing legacy code, and the rewrite-vs-refactor decision. Every technique prioritizes production stability and reversibility.

## Table of Contents

1. [Assessment](#1-assessment)
2. [Strangler Fig Pattern](#2-strangler-fig-pattern)
3. [Dependency Upgrades](#3-dependency-upgrades)
4. [Framework Migrations](#4-framework-migrations)
5. [Database Migrations](#5-database-migrations)
6. [API Evolution](#6-api-evolution)
7. [Testing Legacy Code](#7-testing-legacy-code)
8. [Decision Framework](#8-decision-framework)

---

## 1. Assessment

Before changing anything, measure the current state. Assessment produces the evidence that drives prioritization and stakeholder buy-in.

### When to Use

- Inheriting a codebase with unknown tech debt
- Proposing a modernization initiative to leadership
- Prioritizing which modules to modernize first
- Estimating timelines and risk for migration projects

### Codebase Health Metrics

Collect these quantitative signals before forming a plan:

| Metric | Tool / Method | Red Flag Threshold |
|---|---|---|
| Cyclomatic complexity | `radon`, `lizard`, SonarQube | Avg > 15 per function |
| Dependency freshness | `npm outdated`, `pip list --outdated` | > 50% packages behind major |
| Test coverage | `coverage`, `nyc`, `jest --coverage` | < 30% line coverage |
| Dead code ratio | `vulture` (Python), `ts-prune` (TS) | > 15% of codebase |
| Build time | CI pipeline metrics | > 10 min for incremental |
| Mean time to recovery | Incident logs | > 4 hours |
| Deployment frequency | CI/CD logs | < 1/week when desired daily |
| Security vulnerabilities | `npm audit`, `safety`, Snyk | Any critical/high unpatched |

### Dependency Audit

```bash
# Node.js — list outdated with severity
npm outdated --long
npx npm-check-updates --target greatest

# Python — list outdated and check vulnerabilities
pip list --outdated --format=columns
pip-audit

# Ruby
bundle outdated --strict
bundle audit check --update

# General — SBOM generation for compliance
syft packages dir:. -o cyclonedx-json > sbom.json
```

Classify each dependency:

- **Active**: maintained, security patches within 30 days
- **Stale**: last release > 12 months, open CVEs unpatched
- **Abandoned**: archived repo, no maintainer response > 6 months
- **Forked**: team maintains a private fork (high maintenance burden)

### Risk Scoring

Assign each module a modernization risk score (1-5):

```
Risk Score = (Complexity * 0.3) + (Coupling * 0.25) + (Test Gap * 0.25) + (Business Criticality * 0.2)

Where each factor is rated 1-5:
  Complexity:          1 = simple CRUD, 5 = deep algorithmic / state machine
  Coupling:            1 = isolated module, 5 = touches every other module
  Test Gap:            1 = >90% coverage, 5 = no tests at all
  Business Criticality: 1 = internal tool, 5 = revenue-critical path
```

Modules scoring 4-5 need the most caution. Start modernization with modules scoring 2-3 to build confidence and tooling before tackling the riskiest areas.

### Tech Debt Quantification

Frame debt in terms leadership understands:

```markdown
## Tech Debt Impact Report

### Developer Velocity
- New feature delivery: 3.2x slower than industry benchmark
- Average PR cycle time: 4.7 days (target: < 1 day)
- Onboarding time for new devs: 6 weeks (target: 2 weeks)

### Operational Cost
- Monthly incident hours from legacy systems: 47 hrs
- Infrastructure premium from outdated runtime: $2,400/mo
- Security patching (manual backports): 15 hrs/sprint

### Risk Exposure
- 3 unpatched critical CVEs in unmaintained dependencies
- No disaster recovery tested in 18 months
- Single point of failure: monolith deployment
```

### Anti-Patterns

- **Boiling the ocean**: trying to assess everything at once instead of focusing on the area you plan to modernize first
- **Analysis paralysis**: spending months on assessment without starting any actual migration work
- **Vanity metrics**: tracking lines of code changed instead of measurable outcomes (deploy frequency, MTTR)
- **Skipping stakeholder alignment**: producing a technical report that leadership cannot act on

---

## 2. Strangler Fig Pattern

Named after the strangler fig tree that grows around a host tree and gradually replaces it. New functionality is built alongside the legacy system, and traffic is incrementally shifted.

### When to Use

- The legacy system is too large or risky for a one-shot rewrite
- You need to keep the system running in production throughout the migration
- Different modules can be migrated independently
- You want reversibility at every step

### Incremental Replacement

```
Phase 1: Identify a bounded context (e.g., user authentication)
Phase 2: Build the replacement service alongside the legacy system
Phase 3: Route a percentage of traffic to the new service
Phase 4: Validate correctness with shadow traffic / dual writes
Phase 5: Cut over fully and decommission the legacy module
Phase 6: Repeat for the next bounded context
```

Each phase is a safe resting point. If anything goes wrong, revert the routing.

### Facade Routing

Place a routing layer (reverse proxy, API gateway, or in-app router) in front of both systems:

```nginx
# Nginx example — route by path prefix
upstream legacy_app {
    server legacy.internal:8080;
}
upstream modern_app {
    server modern.internal:3000;
}

server {
    listen 80;

    # Migrated routes go to the new service
    location /api/v2/users {
        proxy_pass http://modern_app;
    }
    location /api/v2/billing {
        proxy_pass http://modern_app;
    }

    # Everything else stays on legacy
    location / {
        proxy_pass http://legacy_app;
    }
}
```

Application-level routing for monolith extraction:

```python
# Django middleware — strangler fig router
class StranglerFigMiddleware:
    MIGRATED_PATHS = {
        "/api/users/": "http://users-service.internal",
        "/api/billing/": "http://billing-service.internal",
    }

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        for prefix, target in self.MIGRATED_PATHS.items():
            if request.path.startswith(prefix):
                return self._proxy_to(request, target)
        return self.get_response(request)

    def _proxy_to(self, request, target):
        # Forward the request to the modern service
        import httpx
        resp = httpx.request(
            method=request.method,
            url=f"{target}{request.path}",
            headers=dict(request.headers),
            content=request.body,
        )
        return HttpResponse(
            content=resp.content,
            status=resp.status_code,
            content_type=resp.headers.get("content-type"),
        )
```

### Feature Flags for Migration

Use feature flags to control the rollout percentage:

```javascript
// LaunchDarkly / Unleash / custom flag example
const useModernAuth = featureFlags.isEnabled('modern-auth-service', {
  userId: req.user.id,
  percentage: 25,  // 25% of traffic to new service
});

if (useModernAuth) {
  return modernAuthService.authenticate(req);
} else {
  return legacyAuthModule.authenticate(req);
}
```

Ramp the percentage: 5% -> 25% -> 50% -> 100%. Monitor error rates and latency at each step.

### Parallel Running (Shadow Mode)

Run both systems simultaneously and compare outputs without affecting users:

```python
import asyncio
import logging

async def parallel_run(request):
    """Send request to both old and new systems, return old result, log differences."""
    legacy_task = asyncio.create_task(legacy_service.handle(request))
    modern_task = asyncio.create_task(modern_service.handle(request))

    legacy_result = await legacy_task
    modern_result = await modern_task

    if legacy_result != modern_result:
        logging.warning(
            "Mismatch detected",
            extra={
                "request_id": request.id,
                "legacy": legacy_result,
                "modern": modern_result,
            },
        )

    # Always return legacy result during validation phase
    return legacy_result
```

### Anti-Patterns

- **No routing layer**: directly replacing code in place removes the ability to roll back
- **Migrating too many modules at once**: lose the incremental safety that makes the pattern valuable
- **Forgetting to decommission**: leaving dead legacy code running alongside new services indefinitely
- **Shared mutable state**: both systems writing to the same database without coordination causes data corruption

---

## 3. Dependency Upgrades

Outdated dependencies are the most common source of security vulnerabilities and the easiest modernization win.

### When to Use

- Security audit flagged vulnerable dependencies
- A dependency is end-of-life (Node 14, Python 2, Angular 1)
- A needed feature requires a newer version of a library
- Build tooling is incompatible with modern environments

### Upgrade Path Planning

```
Step 1: Generate dependency graph
        npm ls --all / pipdeptree / bundle viz

Step 2: Identify upgrade order (leaf dependencies first)
        Upgrade transitive deps before direct deps that consume them

Step 3: Check each upgrade for breaking changes
        Read changelogs, migration guides, and GitHub issues

Step 4: Group related upgrades into atomic PRs
        e.g., upgrade all testing libs together

Step 5: Run full test suite + manual smoke test per group

Step 6: Deploy to staging, soak for 24-48 hours, then production
```

### Handling Breaking Changes

Common breaking change categories and strategies:

```markdown
| Breaking Change Type        | Detection                | Strategy                          |
|-----------------------------|--------------------------|-----------------------------------|
| Removed API                 | TypeScript / lint errors | Find replacement in migration guide |
| Changed default behavior    | Test failures            | Explicitly set the old default     |
| Renamed export              | Import errors            | Codemod or find-and-replace        |
| Dropped Node/Python version | CI failure               | Upgrade runtime first              |
| Peer dependency conflict    | Install failure          | Upgrade peer dep or use overrides  |
```

### Codemods

Automated code transformations for large-scale changes:

```bash
# React — upgrade prop types
npx react-codemod rename-unsafe-lifecycles ./src

# JavaScript — convert var to const/let
npx jscodeshift -t ./transforms/var-to-const.js ./src

# Python — modernize syntax (2to3 successor)
python -m pyupgrade --py310-plus **/*.py

# TypeScript — custom codemod with ts-morph
npx ts-node ./codemods/update-imports.ts
```

### Renovate / Dependabot Configuration

```json
// renovate.json — grouped updates with auto-merge for patches
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "matchUpdateTypes": ["minor"],
      "groupName": "minor updates",
      "schedule": ["before 6am on Monday"]
    },
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboardApproval": true,
      "labels": ["breaking-change"]
    }
  ]
}
```

### Anti-Patterns

- **Upgrading everything at once**: one failing upgrade blocks all others; keep PRs focused
- **Ignoring peer dependency warnings**: leads to subtle runtime bugs that are hard to trace
- **Pinning exact versions forever**: avoids breakage but accumulates massive upgrade debt
- **Skipping changelogs**: relying only on "does it build?" misses changed default behaviors

---

## 4. Framework Migrations

Framework migrations are the highest-effort modernization task. The key principle: never stop shipping features during the migration.

### When to Use

- Current framework is end-of-life or unmaintained
- Hiring is difficult because developers avoid the old framework
- Performance or security requirements cannot be met
- The ecosystem (plugins, tooling, community) has moved on

### jQuery to React

```javascript
// Phase 1: Mount React components inside jQuery pages
// legacy-page.html still uses jQuery; React islands render inside it

// mount-react-island.js
import { createRoot } from 'react-dom/client';
import UserProfile from './components/UserProfile';

// jQuery triggers the mount
$(document).ready(function() {
  const container = document.getElementById('react-user-profile');
  if (container) {
    const userId = container.dataset.userId;
    const root = createRoot(container);
    root.render(<UserProfile userId={userId} />);
  }
});

// Phase 2: Shared state bridge (jQuery events <-> React context)
// Allow React components and jQuery code to communicate
window.appEventBus = {
  emit(event, data) {
    $(document).trigger(event, data);         // jQuery listeners
    window.dispatchEvent(new CustomEvent(event, { detail: data })); // React listeners
  }
};

// Phase 3: New pages are pure React with React Router
// Phase 4: Convert remaining jQuery pages one at a time
// Phase 5: Remove jQuery dependency entirely
```

### AngularJS to Angular

```typescript
// Use @angular/upgrade module for hybrid app

// main.ts — bootstrap hybrid application
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { UpgradeModule } from '@angular/upgrade/static';
import { AppModule } from './app/app.module';

platformBrowserDynamic().bootstrapModule(AppModule).then(platformRef => {
  const upgrade = platformRef.injector.get(UpgradeModule) as UpgradeModule;
  upgrade.bootstrap(document.body, ['legacyAngularJsApp'], { strictDi: true });
});

// Downgrade Angular component for use in AngularJS templates
import { downgradeComponent } from '@angular/upgrade/static';
import { ModernHeaderComponent } from './modern-header.component';

angular.module('legacyAngularJsApp')
  .directive('modernHeader',
    downgradeComponent({ component: ModernHeaderComponent })
  );

// Upgrade AngularJS service for use in Angular components
import { upgradeInjector } from '@angular/upgrade/static';

export function legacyAuthServiceFactory(i: any) {
  return i.get('AuthService');
}

@NgModule({
  providers: [{
    provide: 'LegacyAuthService',
    useFactory: legacyAuthServiceFactory,
    deps: ['$injector']
  }]
})
export class AppModule {}
```

### Express to Fastify

```javascript
// Phase 1: Run Fastify alongside Express with shared middleware logic
// Use @fastify/express plugin for gradual migration

const fastify = require('fastify')({ logger: true });

// Register Express compatibility layer
await fastify.register(require('@fastify/express'));

// Existing Express middleware works inside Fastify
fastify.use(require('cors')());
fastify.use(require('helmet')());

// New routes use native Fastify (faster validation, serialization)
fastify.get('/api/v2/users/:id', {
  schema: {
    params: {
      type: 'object',
      properties: {
        id: { type: 'string', format: 'uuid' }
      }
    },
    response: {
      200: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          name: { type: 'string' },
          email: { type: 'string' }
        }
      }
    }
  }
}, async (request, reply) => {
  const user = await userService.findById(request.params.id);
  return user;
});

// Legacy Express routes still work
fastify.use('/api/v1', legacyExpressRouter);

await fastify.listen({ port: 3000 });
```

### Django 2 to Django 4

```python
# Step-by-step upgrade path: 2.2 -> 3.0 -> 3.2 LTS -> 4.0 -> 4.2 LTS
# Never skip a major version — follow the official upgrade path

# 1. Run deprecation warnings on current version
python -Wd manage.py test  # -Wd shows deprecation warnings

# 2. Key breaking changes per major version:

# Django 3.0: django.utils.encoding.force_text -> force_str
# Codemod:
# find . -name "*.py" -exec sed -i 's/force_text/force_str/g' {} +

# Django 3.1: JSONField available on all backends (no longer postgres-only)
# from django.contrib.postgres.fields import JSONField  # OLD
# from django.db.models import JSONField                 # NEW

# Django 4.0: USE_L10N defaults to True, timezone handling changes
# settings.py — explicitly set for backward compat during migration
USE_L10N = True
USE_TZ = True

# Django 4.0: url() removed, use path() / re_path()
# OLD: url(r'^users/(?P<pk>[0-9]+)/$', user_detail)
# NEW: path('users/<int:pk>/', user_detail)

# 3. Update one major version at a time
# After each upgrade: run full test suite, fix deprecation warnings,
# deploy to staging, soak, then proceed to next version
```

### Anti-Patterns

- **Big bang rewrite**: "We'll rewrite it all and launch in 6 months" — this almost always fails or ships late
- **Framework-of-the-month**: migrating to something trendy rather than something stable and well-supported
- **Hybrid limbo**: running two frameworks indefinitely without a plan to complete the migration
- **Ignoring the data layer**: migrating the framework but leaving legacy data access patterns unchanged

---

## 5. Database Migrations

Schema changes in production databases require extra caution. A bad migration can cause downtime, data loss, or corrupted state.

### When to Use

- Adding or removing columns in production tables
- Changing data types or constraints
- Splitting or merging tables during service extraction
- Moving from one database technology to another

### Schema Evolution Principles

```
1. Every migration must be reversible (include rollback SQL)
2. Never rename a column in one step — add new, migrate data, drop old
3. Never drop a column that the current running code still reads
4. Migrations must be backward-compatible with the previous app version
5. Large data migrations run as background jobs, not in deploy scripts
```

### Expand-Contract Pattern

The safest way to make breaking schema changes in zero-downtime deployments:

```sql
-- EXPAND PHASE (deploy 1): add new column, keep old column
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- DATA MIGRATION (background job): populate new column from old
UPDATE users SET full_name = first_name || ' ' || last_name
WHERE full_name IS NULL;
-- Run in batches of 1000 to avoid locking:
-- UPDATE users SET full_name = first_name || ' ' || last_name
-- WHERE full_name IS NULL LIMIT 1000;

-- APP UPDATE (deploy 2): code reads from full_name, writes to both
-- This deploy is backward-compatible with the old schema

-- CONTRACT PHASE (deploy 3): drop old columns after verification
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

Three separate deployments. Each one is safe to roll back independently.

### Online Schema Changes

For large tables where ALTER TABLE would lock for minutes:

```bash
# MySQL — use pt-online-schema-change (Percona Toolkit)
pt-online-schema-change \
  --alter "ADD COLUMN full_name VARCHAR(255)" \
  --execute \
  --chunk-size=1000 \
  --max-lag=1s \
  D=mydb,t=users

# MySQL 8+ — use gh-ost (GitHub Online Schema Tool)
gh-ost \
  --alter="ADD COLUMN full_name VARCHAR(255)" \
  --database=mydb \
  --table=users \
  --execute \
  --allow-on-master

# PostgreSQL — most ALTER TABLEs are non-locking, but adding
# a NOT NULL column with a default requires care:
-- Safe in PG 11+: default is metadata-only, no table rewrite
ALTER TABLE users ADD COLUMN full_name VARCHAR(255) NOT NULL DEFAULT '';

-- For older PG: add nullable, backfill, then add constraint
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
-- backfill in batches
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
```

### Backward Compatibility Checklist

Before deploying any migration:

```markdown
- [ ] Current running code works with the NEW schema (expand phase)
- [ ] New code works with the OLD schema (in case of rollback)
- [ ] Data migration handles NULL values and edge cases
- [ ] Migration runs within acceptable time on production-sized data
- [ ] Rollback SQL is tested and documented
- [ ] Indexes exist for any new columns used in WHERE/JOIN clauses
- [ ] Foreign key constraints added only after data is consistent
- [ ] Application-level validation matches new database constraints
```

### Anti-Patterns

- **Deploying code and schema in one step**: if the deploy fails mid-way, you have a mismatched code/schema state
- **Running data migrations in transactions**: a 2-hour UPDATE inside a transaction locks the table the entire time
- **Dropping columns before removing code references**: causes runtime errors during rollback or rolling deploys
- **No rollback plan**: every migration needs a tested reverse operation

---

## 6. API Evolution

APIs outlive the code that implements them. Evolving APIs without breaking consumers is critical during modernization.

### When to Use

- Changing response shapes during backend modernization
- Deprecating old endpoints after service extraction
- Supporting multiple client versions during mobile app transitions
- Introducing a new API style (REST to GraphQL, monolith to microservices)

### Versioning Strategies

```markdown
| Strategy         | Example                        | Pros                        | Cons                         |
|------------------|--------------------------------|-----------------------------|------------------------------|
| URL path         | /api/v1/users, /api/v2/users   | Explicit, easy to route     | URL proliferation            |
| Header           | Accept: application/vnd.app.v2 | Clean URLs                  | Harder to test in browser    |
| Query parameter  | /api/users?version=2           | Easy to switch              | Pollutes query string        |
| Content negotiation | Accept: application/json;v=2| Standards-based             | Complex to implement         |
```

Recommendation: use URL path versioning (`/api/v1/`) for public APIs. It is the most explicit and debuggable approach.

### Deprecation Policy

```yaml
# API Deprecation Timeline
Phase 1 - Announce (Month 0):
  - Add Sunset header to deprecated endpoints
  - Add deprecation notice to API docs
  - Notify consumers via email/webhook
  - Log usage of deprecated endpoints

Phase 2 - Warn (Month 3):
  - Return Warning header: "299 - Endpoint deprecated, use /api/v2/users"
  - Dashboard shows consumer migration progress
  - Offer migration support to high-usage consumers

Phase 3 - Restrict (Month 6):
  - Rate limit deprecated endpoints (50% reduction)
  - Return 410 Gone for consumers who have migrated

Phase 4 - Remove (Month 9):
  - Return 410 Gone with migration instructions
  - Remove endpoint code
```

Example headers for deprecated endpoints:

```
HTTP/1.1 200 OK
Sunset: Sat, 01 Mar 2027 00:00:00 GMT
Deprecation: true
Link: </api/v2/users>; rel="successor-version"
Warning: 299 - "This endpoint is deprecated. Migrate to /api/v2/users by 2027-03-01."
```

### API Gateway Routing

Use an API gateway to route between legacy and modern backends:

```yaml
# Kong / AWS API Gateway style configuration
routes:
  # Migrated endpoints -> new service
  - path: /api/v1/users
    upstream: http://users-service.internal:3000
    plugins:
      - name: rate-limiting
        config: { minute: 100 }
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Migrated: true"

  # Legacy endpoints -> monolith
  - path: /api/v1/orders
    upstream: http://legacy-monolith.internal:8080

  # New version -> new service directly
  - path: /api/v2/users
    upstream: http://users-service.internal:3000
```

### Consumer-Driven Contracts

Ensure API changes do not break any consumer:

```javascript
// Pact — consumer-driven contract testing
// Consumer side (frontend or downstream service)
const { Pact } = require('@pact-foundation/pact');

const provider = new Pact({
  consumer: 'WebApp',
  provider: 'UserService',
});

describe('User API Contract', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());

  it('returns user by ID', async () => {
    await provider.addInteraction({
      state: 'user 123 exists',
      uponReceiving: 'a request for user 123',
      withRequest: {
        method: 'GET',
        path: '/api/v1/users/123',
      },
      willRespondWith: {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          id: '123',
          name: Matchers.string('Alice'),
          email: Matchers.email(),
        },
      },
    });

    const response = await fetch(`${provider.mockService.baseUrl}/api/v1/users/123`);
    const user = await response.json();
    expect(user.id).toBe('123');
  });
});

// Provider side — verify against all consumer contracts
// pact-provider-verifier runs during CI for the UserService
```

### Anti-Patterns

- **Breaking changes without versioning**: changing response shapes on existing endpoints breaks all consumers
- **Too many live versions**: supporting v1 through v7 simultaneously is unsustainable; limit to 2-3
- **No usage tracking**: cannot deprecate endpoints safely without knowing who calls them
- **Internal APIs without contracts**: "it's just us" until another team starts consuming your API

---

## 7. Testing Legacy Code

Legacy code is defined as code without tests (Michael Feathers). Before you can safely modernize, you need a safety net.

### When to Use

- Preparing to refactor a module that has no tests
- Validating that migrated code behaves identically to the original
- Building confidence before a major dependency upgrade
- Onboarding to an unfamiliar codebase

### Characterization Tests

Characterization tests capture what the code actually does, not what it should do:

```python
# Step 1: Call the function with known inputs
# Step 2: Assert whatever the current output is
# Step 3: Now you have a safety net for refactoring

import pytest
from legacy_module import calculate_pricing

class TestCalculatePricingCharacterization:
    """These tests document CURRENT behavior, not desired behavior.
    If a test fails after refactoring, investigate before updating the assertion."""

    def test_basic_order(self):
        result = calculate_pricing(items=[{"sku": "A1", "qty": 2, "price": 10.00}])
        # Captured from actual execution — do not change without investigation
        assert result == {"subtotal": 20.00, "tax": 1.60, "total": 21.60}

    def test_empty_order(self):
        result = calculate_pricing(items=[])
        assert result == {"subtotal": 0, "tax": 0, "total": 0}

    def test_discount_edge_case(self):
        # Legacy code has a known quirk: 100% discount still charges $0.01
        result = calculate_pricing(
            items=[{"sku": "B2", "qty": 1, "price": 50.00}],
            discount_pct=100
        )
        assert result == {"subtotal": 50.00, "tax": 0, "total": 0.01}
        # TODO: Decide if this is a bug to fix or behavior to preserve
```

### Approval Testing (Snapshot Testing)

Capture entire output and approve it as the baseline:

```javascript
// Jest snapshot testing for legacy API responses
const request = require('supertest');
const app = require('../legacy-app');

describe('Legacy API Snapshots', () => {
  test('GET /api/v1/products returns expected shape', async () => {
    const response = await request(app).get('/api/v1/products');
    expect(response.body).toMatchSnapshot();
    // First run: creates __snapshots__/legacy-api.test.js.snap
    // Subsequent runs: fails if output changes
  });

  test('POST /api/v1/orders processes correctly', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .send({ items: [{ sku: 'WIDGET-1', qty: 3 }] });
    expect(response.status).toBe(201);
    expect(response.body).toMatchSnapshot();
  });
});

// When you intentionally change behavior:
// npx jest --updateSnapshot
```

### Golden Master Testing

For complex outputs (reports, rendered HTML, file generation):

```python
import hashlib
import json
from pathlib import Path

GOLDEN_DIR = Path("tests/golden_masters")

def golden_master_test(test_name, actual_output):
    """Compare output against a stored golden master file."""
    golden_file = GOLDEN_DIR / f"{test_name}.golden"

    if not golden_file.exists():
        # First run: create the golden master
        golden_file.parent.mkdir(parents=True, exist_ok=True)
        golden_file.write_text(json.dumps(actual_output, indent=2, default=str))
        pytest.skip(f"Golden master created: {golden_file}. Review and re-run.")

    expected = json.loads(golden_file.read_text())
    assert actual_output == expected, (
        f"Output differs from golden master.\n"
        f"To update: delete {golden_file} and re-run."
    )

# Usage
def test_monthly_report_generation():
    report = generate_monthly_report(month=3, year=2026)
    golden_master_test("monthly_report_march_2026", report)
```

### Strategy for Adding Tests to Legacy Code

```
Priority order for adding tests:

1. Public API endpoints — highest value, tests the full stack
2. Business-critical calculations — revenue, billing, permissions
3. Code you are about to change — immediate safety net needed
4. Integration boundaries — database queries, external API calls
5. Frequently-buggy modules — check git blame for hotspot files

Techniques for untestable code:

- Extract pure functions from methods with side effects
- Introduce seams (interfaces/protocols) at dependency boundaries
- Use dependency injection to replace real services with test doubles
- Subclass-and-override for hard-to-test classes (temporary technique)
- Wrap global state access behind injectable services
```

### Anti-Patterns

- **Testing implementation details**: tests that break when you refactor internals but behavior is unchanged
- **Updating snapshots without review**: `--updateSnapshot` should trigger a careful diff review, not a rubber stamp
- **Skipping tests for "simple" code**: the simplest-looking functions often hide surprising edge cases
- **100% coverage as a goal**: coverage does not equal correctness; focus on high-value paths first

---

## 8. Decision Framework

The hardest question in legacy modernization: should we rewrite, refactor, or leave it alone?

### When to Use

- Evaluating whether to invest in modernization at all
- Choosing between incremental refactoring and a ground-up rewrite
- Presenting options to stakeholders with clear trade-offs
- Setting realistic timelines and milestones

### Rewrite vs Refactor Decision Matrix

```
                    Low Business Risk    High Business Risk
                  ┌─────────────────────┬─────────────────────┐
 High Code        │                     │                     │
 Quality          │   LEAVE ALONE       │   REFACTOR          │
 (maintainable,   │   (it works, focus  │   (improve the      │
  tested)         │   effort elsewhere) │   critical path)    │
                  ├─────────────────────┼─────────────────────┤
 Low Code         │                     │                     │
 Quality          │   REPLACE / RETIRE  │   STRANGLER FIG     │
 (unmaintained,   │   (not worth the    │   (incrementally    │
  no tests)       │   investment)       │   replace — never   │
                  │                     │   big-bang rewrite)  │
                  └─────────────────────┴─────────────────────┘
```

Decision criteria:

| Factor | Favors Refactor | Favors Rewrite |
|---|---|---|
| Domain knowledge | Team understands the domain well | Domain is poorly understood |
| Test coverage | > 40% coverage exists | < 10% coverage, no specs |
| Architecture | Modular, clear boundaries | Monolithic, circular deps |
| Team skill | Team knows the current stack | Team only knows the target stack |
| Timeline | Continuous delivery needed | Can pause features for 1-2 quarters |
| Data model | Data model is sound | Data model is fundamentally broken |
| Dependencies | Core deps are maintainable | Core deps are abandoned |

### Risk Assessment

```markdown
## Risk Register for Modernization Project

### Technical Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Data migration corruption | Medium | Critical | Dual-write validation, rollback plan |
| Performance regression | High | High | Load testing at each phase, feature flags |
| Integration breakage | Medium | High | Contract testing, shadow traffic |
| Scope creep into rewrite | High | Critical | Strict phase boundaries, weekly reviews |

### Organizational Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Key person dependency | Medium | High | Pair programming, documented decisions |
| Stakeholder impatience | High | Medium | Visible progress every 2 weeks |
| Feature freeze resistance | High | High | Strangler fig — no freeze needed |
| Budget overrun | Medium | Critical | Phase-gated funding, kill criteria |
```

### Stakeholder Communication

```markdown
## Modernization Proposal Template

### Current State
- System age: 8 years, originally built for 100 users, now serving 50,000
- Incident frequency: 3.2 production incidents per month (up from 0.5 last year)
- Feature velocity: 2.1 features/sprint (down from 5.3 two years ago)

### Proposed Approach
Strangler fig migration over 4 phases (12 months total):
- Phase 1 (Q1): Authentication & user management — reduce incident rate by 40%
- Phase 2 (Q2): Billing & payments — enable 3 blocked revenue features
- Phase 3 (Q3): Reporting & analytics — 10x query performance
- Phase 4 (Q4): Remaining modules & legacy decommission

### Investment
- Team: 3 engineers dedicated, 2 engineers 50% time
- Feature impact: 15% velocity reduction in Q1, net positive by Q3
- Infrastructure: $3,200/mo additional during parallel running

### Success Criteria (per phase)
- Zero data loss during migration
- Latency within 10% of current baseline
- All consumer contracts passing
- Legacy module fully decommissioned within 30 days of cutover

### Kill Criteria
- If Phase 1 exceeds 5 months, re-evaluate approach
- If production incidents increase during migration, pause and stabilize
```

### Timeline Planning

```
Realistic modernization timeline for a medium-sized system:

Month 1-2:   Assessment, risk scoring, team alignment
Month 3:     Tooling setup (CI/CD, feature flags, contract tests)
Month 4-6:   First module migration (authentication — chosen for
             clear boundaries and high incident rate)
Month 7:     Retrospective, adjust approach based on learnings
Month 8-10:  Second module migration (billing — highest business value)
Month 11-12: Third module, begin legacy decommissioning
Month 13+:   Remaining modules, ongoing maintenance

Key milestones:
  ✓ First request served by new system (Month 4)
  ✓ First legacy module fully decommissioned (Month 6)
  ✓ Feature velocity returns to baseline (Month 9)
  ✓ Legacy system powered off (Month 14-18)

Budget buffer: add 30-50% to all estimates. Modernization
always surfaces hidden complexity.
```

### Anti-Patterns

- **Second system effect**: the rewrite becomes overengineered because the team tries to fix every shortcoming at once
- **Invisible progress**: modernization work that does not produce visible improvements loses stakeholder support fast
- **No kill criteria**: every modernization project needs defined conditions under which you stop and change approach
- **Comparing against ideal**: comparing the legacy system to a perfect future system instead of realistic alternatives
- **Modernization as vanity project**: choosing exciting technology over boring technology that solves the actual problem

---

## Output Format

When producing a modernization plan, structure the output as:

```markdown
## Modernization Plan: [System Name]

### Assessment Summary
- Overall risk score: [1-5]
- Recommended approach: [Refactor / Strangler Fig / Replace / Leave Alone]
- Estimated timeline: [X months]
- Team allocation: [N engineers at Y% capacity]

### Phase Breakdown
For each phase:
- Scope: [which modules/components]
- Duration: [weeks]
- Dependencies: [what must be done first]
- Success criteria: [measurable outcomes]
- Rollback plan: [how to revert if needed]

### Risk Register
[Top 5 risks with likelihood, impact, and mitigation]

### Decision Log
[Key decisions made and their rationale]
```
