---
name: laravel-specialist
description: Laravel development: Eloquent, queues, events, policies, Livewire/Inertia tradeoffs, testing, and production deployment patterns.
---

# Laravel Specialist

Build Laravel applications that remain clear under growth. Favor thin controllers, explicit validation, predictable domain boundaries, queue-backed side effects, and database access patterns that do not hide performance problems behind Eloquent convenience.

## When to Use

- Building or refactoring a Laravel application or API
- Designing Eloquent models, relationships, scopes, and query patterns
- Choosing between Blade, Livewire, Inertia, or API-first delivery
- Adding queues, events, scheduled jobs, notifications, or webhooks
- Implementing authorization with policies, gates, and multi-tenant rules
- Tightening test coverage, deployment safety, and production operations

## Workflow

1. Start with the request boundary.
   Define routes, request validation, authorization rules, and response shape before touching models or jobs.
2. Keep controllers thin.
   Use `FormRequest` classes for validation and dedicated actions/services for business workflows that span multiple models.
3. Model relationships carefully.
   Be explicit about eager loading, indexes, uniqueness rules, and soft-delete behavior so Eloquent stays predictable.
4. Move side effects out of the request cycle.
   Queue emails, webhooks, document generation, third-party syncs, and non-critical recalculations.
5. Choose the rendering stack deliberately.
   Blade is simplest, Livewire suits server-driven interactions, Inertia fits SPA-like flows, and API-first works best for multi-client systems.
6. Ship with operational safety.
   Add migrations, horizon/queue visibility, scheduler checks, structured logs, and rollback-aware deployment steps.

## Decision Guide

| Choice | Use When | Avoid When |
|---|---|---|
| Blade | Mostly server-rendered pages with simple interactions | The team expects SPA-like client state everywhere |
| Livewire | Form-heavy back-office UX with small JS footprint | Highly interactive dashboards with complex client-side state |
| Inertia | Laravel backend with modern frontend ergonomics | You need multiple public clients beyond the web app |
| API-first | Mobile apps, partner APIs, or multiple frontends | The product only needs a fast server-rendered web UI |

## Application Rules

- Validate with `FormRequest`, not ad hoc controller logic.
- Keep authorization close to the action with policies and gates.
- Prefer explicit query scopes and repository-style reads when query complexity grows.
- Treat jobs as retryable and idempotent. External side effects must survive retries without duplicating work.
- Use database transactions around multi-row invariants, not around long-running third-party calls.
- Watch for N+1 queries. Eloquent convenience does not excuse hidden query explosions.

## Data and Async Checklist

- Add indexes for every frequent filter, join, and unique business key.
- Use `with()` and `loadMissing()` intentionally; do not eager load the world.
- Keep queue payloads small. Pass IDs, not huge serialized models.
- Separate domain events from infrastructure notifications so workflows stay composable.
- Use Horizon or equivalent queue visibility in production.

## Testing Strategy

- Feature tests for routes, policies, validation, and major workflows
- Unit tests for pure domain logic, formatters, and custom rules
- Database tests around scopes, constraints, and transaction boundaries
- Queue and notification assertions for side effects
- Smoke tests for migration + deploy-critical paths

## Output Format

When using this skill, produce:

1. **Application shape** - routes, requests, actions/services, and model boundaries
2. **Data plan** - relationships, indexes, eager-loading strategy, and migrations
3. **Async plan** - jobs, events, notifications, retry/idempotency behavior
4. **Testing plan** - feature, unit, queue, and deployment checks
5. **Operational notes** - scheduler, Horizon, observability, and rollout risks
