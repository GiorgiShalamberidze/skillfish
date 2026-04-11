---
name: flutter-expert
description: Flutter app engineering: widget architecture, Riverpod/Bloc state, platform channels, performance tuning, testing, and release workflows.
---

# Flutter Expert

Build Flutter apps that stay maintainable as product complexity grows. Favor clear state boundaries, small composable widgets, measurable performance work, and release pipelines that treat Android and iOS as first-class targets.

## When to Use

- Designing a new Flutter application or restructuring an existing one
- Choosing between local state, Riverpod, Bloc, and other app-wide patterns
- Integrating native capabilities such as camera, notifications, payments, or deep links
- Fixing jank, slow startup, excessive rebuilds, or large binary size
- Defining widget, integration, and golden-test coverage
- Preparing Play Store and App Store releases

## Workflow

1. Define product and platform constraints first.
   Capture target devices, offline requirements, push/deep link needs, accessibility expectations, and store-release constraints before picking packages.
2. Split the app by responsibility.
   Keep presentation, application, domain, and data concerns separate so widgets stay declarative and business logic stays testable.
3. Choose the smallest state tool that fits.
   Use local widget state for ephemeral UI, Riverpod for most shared async state, and Bloc when event/state auditability matters.
4. Treat plugins as boundaries.
   Wrap platform plugins and SDKs behind interfaces so camera, auth, secure storage, and analytics integrations are mockable.
5. Optimize from measurements, not intuition.
   Profile frame time, rebuild counts, image decode cost, startup work, and binary size before rewriting architecture.
6. Lock down the release path.
   Automate signing, environment config, crash reporting, and store metadata so shipping is repeatable.

## Decision Guide

| Choice | Use When | Avoid When |
|---|---|---|
| `setState` | Small local state inside one widget subtree | Shared async data or cross-screen workflows |
| Riverpod | Default for most apps; testable and async-friendly | The team needs explicit event logs and state transitions |
| Bloc | Complex workflows, regulated flows, or event/state traceability matter | The app mostly needs straightforward fetch-and-render state |
| Platform channels / plugin wrappers | Native SDK access is unavoidable | A pure Dart package already covers the need well |

## Architecture Rules

- Keep widgets declarative. Networking, mapping, and orchestration belong in providers, controllers, or use cases.
- Prefer many small widgets over giant screen files with mixed layout and business rules.
- Normalize API and storage models before they reach the UI layer.
- Centralize theme tokens, spacing, typography, and semantic colors. Do not scatter raw values across screens.
- Lazy-load non-critical SDKs so analytics, chat, and experiments do not slow first paint.
- Treat navigation as application state. Deep links, onboarding gates, and auth redirects need explicit rules.

## Performance Checklist

- Mark stable widgets `const` wherever possible.
- Use `ListView.builder` or slivers for long collections; never render large lists with `Column`.
- Move expensive derived data out of `build()`.
- Resize and cache images intentionally; oversized decoded bitmaps are a common memory spike.
- Profile in profile or release mode, not debug mode.
- Watch for rebuild storms caused by broad provider subscriptions.

## Testing Strategy

- Unit tests: domain rules, formatters, validators, repositories, and mappers
- Widget tests: critical screen states, forms, empty/loading/error branches
- Integration tests: login, payment, push/deep-link entry points, offline recovery
- Golden tests: reusable design-system components, not every screen
- Device checks: at least one lower-end Android device and one current iPhone class before release

## Output Format

When using this skill, produce:

1. **Architecture** - package layout, state approach, and integration boundaries
2. **Implementation plan** - screens, services, native hooks, and migration steps
3. **Performance risks** - likely rebuild, memory, startup, or binary-size issues
4. **Testing plan** - unit, widget, integration, and release verification
5. **Release checklist** - signing, config, observability, and store-submission steps
