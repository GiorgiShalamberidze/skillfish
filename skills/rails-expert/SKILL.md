---
name: rails-expert
description: Ruby on Rails engineering: Active Record, service boundaries, Hotwire, background jobs, testing, performance tuning, and safe production changes.
---

# Rails Expert

Build Rails systems that keep shipping speed without turning convention into accidental complexity. Lean on Rails where it helps, but introduce clear service boundaries, background work, and data discipline before controllers, callbacks, or Active Record relations become a hidden dependency graph.

## When to Use

- Building or restructuring a Rails monolith or Rails API
- Designing models, associations, scopes, callbacks, and data migrations
- Choosing between server-rendered Rails, Hotwire, or API-first frontends
- Adding Sidekiq/Active Job workflows, mailers, and scheduled tasks
- Fixing N+1 queries, slow endpoints, or bloated domain models
- Hardening test coverage and deployment safety

## Workflow

1. Define the user-facing flow first.
   Start from route, controller, policy, and response behavior before shaping models or callbacks.
2. Keep controllers boring.
   Controllers should coordinate request parsing, authorization, and response rendering, not own business rules.
3. Use Active Record intentionally.
   Associations, scopes, validations, and transactions are powerful, but once logic becomes multi-step or cross-aggregate, move it into services or command objects.
4. Pick the interaction model deliberately.
   Classic server-rendered Rails is fastest to ship, Hotwire suits incremental interactivity, and API-first is better for multiple clients.
5. Push slow work out of the request cycle.
   Mailers, webhooks, search indexing, exports, and external sync belong in background jobs.
6. Treat schema changes as production events.
   Make migrations forward-safe, reversible when possible, and careful about locks on large tables.

## Decision Guide

| Choice | Use When | Avoid When |
|---|---|---|
| Classic Rails views | CRUD-heavy product with limited client complexity | Rich client state and real-time UI dominate the product |
| Hotwire | You want modern interactivity without a separate SPA stack | The frontend requires extensive local state orchestration |
| Service / command objects | Workflows touch multiple models or external systems | The logic is still a simple validation or derived attribute |
| Callbacks | Small local invariants tied to one model lifecycle | Cross-system orchestration or anything order-dependent |

## Rails Rules

- Prefer explicit service boundaries over callback chains once workflows get non-trivial.
- Use `includes`, `preload`, or `eager_load` on hot paths. Never accept N+1 as "just Rails."
- Keep validation at the right layer: model invariants in the model, request/flow validation in form objects or services.
- Make jobs idempotent and retry-safe. Duplicate webhook delivery and Sidekiq retries are normal.
- Use transactions around invariants, not around network calls.
- Audit `default_scope` usage carefully; it often creates surprising query behavior.

## Performance and Operations

- Add indexes for foreign keys, unique business keys, and common filters.
- Use read models or query objects when reporting logic stops fitting neatly in model scopes.
- Prefer additive migrations on large tables, then backfill in batches.
- Capture request, job, and external API latency separately so bottlenecks are obvious.
- Keep deploys reversible: feature flags, dual reads/writes when needed, and explicit data backfills.

## Testing Strategy

- Request specs for controller and routing behavior
- Model and service tests for domain rules
- Job specs for background behavior and retries
- System tests for high-value user journeys
- Migration and rollback checks for risky schema changes

## Output Format

When using this skill, produce:

1. **Request-to-domain flow** - routes, controllers, policies, services, and models
2. **Data plan** - associations, indexes, migrations, and query hot spots
3. **Async plan** - jobs, retries, external side effects, and idempotency
4. **Testing plan** - request, model/service, job, and system coverage
5. **Production notes** - deploy sequence, migration safety, and observability hooks
