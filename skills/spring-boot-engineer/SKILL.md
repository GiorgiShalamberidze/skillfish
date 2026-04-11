---
name: spring-boot-engineer
description: Spring Boot engineering: REST services, configuration, validation, security, data access, testing, observability, and production deployment patterns.
---

# Spring Boot Engineer

Build Spring Boot services that stay readable in production. Favor explicit module boundaries, typed configuration, predictable transactions, well-defined security rules, and observability that makes operational failures obvious instead of mysterious.

## When to Use

- Building or refactoring Spring Boot applications and APIs
- Designing controller, service, repository, and domain boundaries
- Choosing between MVC, WebFlux, JPA, jOOQ, messaging, and async work
- Adding validation, authentication, authorization, and configuration profiles
- Improving test coverage, startup behavior, and production observability
- Planning containerized or cloud deployments for Java services

## Workflow

1. Start from the domain and contracts.
   Define aggregates, commands, events, and API boundaries before wiring framework details.
2. Keep HTTP adapters thin.
   Controllers translate requests and responses; orchestration belongs in services or use cases.
3. Make configuration typed and explicit.
   Use `@ConfigurationProperties`, environment-specific profiles, and validated settings instead of scattered `@Value` strings.
4. Choose the data-access style deliberately.
   JPA is fine for standard CRUD and aggregate persistence; use jOOQ or JDBC for query-heavy reporting and SQL-first workflows.
5. Treat security and transactions as first-class design concerns.
   Validate input, scope authorization rules clearly, and keep transaction boundaries tight.
6. Ship with production visibility.
   Expose health/readiness, metrics, structured logs, and safe rollout controls from day one.

## Decision Guide

| Choice | Use When | Avoid When |
|---|---|---|
| Spring MVC | Standard request/response APIs and blocking I/O dominate | You truly need reactive end-to-end throughput |
| WebFlux | High fan-out async I/O and reactive dependencies are already in place | The team mainly needs normal CRUD services |
| JPA | Aggregate-centric CRUD with moderate query complexity | The workload is dominated by complex joins and reporting SQL |
| jOOQ / JDBC | Query shape and SQL control matter more than ORM convenience | The team needs rapid CRUD scaffolding with minimal SQL |

## Engineering Rules

- Keep controllers, services, and repositories focused on one responsibility each.
- Prefer constructor injection everywhere.
- Put business invariants in the domain/service layer, not in controllers.
- Avoid long-running remote calls inside transactions.
- Validate all incoming data with Bean Validation or explicit parsers before it touches the domain.
- Treat retries, message consumption, and scheduled work as idempotent by default.

## Production Checklist

- Typed config with sane defaults and validation
- `/actuator/health`, readiness, metrics, and log correlation IDs
- Clear security rules for public, authenticated, and admin paths
- Database migration tooling such as Flyway or Liquibase
- Timeouts, circuit breakers, and retry policy for outbound calls
- Container and JVM settings aligned to actual memory limits

## Testing Strategy

- Unit tests for services, validators, and domain rules
- Slice tests for MVC, repository, and security boundaries
- Integration tests with real infrastructure where behavior matters
- Contract or API tests for public endpoints
- Smoke tests for startup, health checks, and migration safety

## Output Format

When using this skill, produce:

1. **Service architecture** - modules, adapters, services, repositories, and configuration
2. **Data strategy** - persistence choice, transaction boundaries, and migration plan
3. **Security plan** - authn/authz rules, secrets handling, and public/private paths
4. **Testing plan** - unit, slice, integration, and rollout checks
5. **Operational plan** - observability, scaling assumptions, and deployment notes
