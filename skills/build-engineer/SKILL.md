---
name: build-engineer
description: Build engineering: reproducible builds, dependency caching, monorepo build graphs, artifact pipelines, release automation, and CI performance tuning.
---

# Build Engineer

Make builds fast, reproducible, and boring. This skill focuses on deterministic inputs, cache efficiency, artifact integrity, release automation, and CI pipelines that scale with team and repository size instead of collapsing under their own complexity.

## When to Use

- Designing or fixing CI/CD build pipelines
- Improving slow local builds, test runs, or monorepo task graphs
- Standardizing artifact packaging, publishing, and provenance
- Debugging flaky builds, dependency drift, or environment mismatch
- Introducing remote caches, build matrices, or reusable workflows
- Planning release automation for libraries, apps, or containers

## Workflow

1. Map the build graph.
   Identify sources, generated code, dependency restore, compile, test, package, sign, and publish steps before tuning performance.
2. Make builds reproducible first.
   Pin toolchains, lock dependencies, isolate environment variables, and remove hidden machine-local assumptions.
3. Add caching only after correctness.
   Cache dependency downloads, compiler outputs, and remote task artifacts once cache keys and invalidation rules are clear.
4. Separate build stages by responsibility.
   Restore, build, test, package, and publish should be independently inspectable so failures are obvious.
5. Treat artifacts as products.
   Sign, version, store, and retain binaries, images, and packages with the same rigor as source code.
6. Measure continuously.
   Track p50 and p95 build time, cache hit rate, flaky-step rate, and queue time, not just the best-case run.

## Build Rules

- Hermetic beats convenient. If a build depends on undeclared system state, it will fail in CI at the worst time.
- Prefer reusable workflows and shared tooling over copy-pasted pipeline YAML.
- Fail fast on formatting, lint, type, or unit-test errors before expensive packaging work.
- Keep generated files deterministic so diffs and cache keys stay stable.
- Review cache invalidation strategy whenever compiler version, lockfile, or environment changes.
- Publish immutable artifacts; never overwrite a released version.

## Optimization Guide

| Bottleneck | First Lever |
|---|---|
| Slow dependency restore | Lockfiles, mirror/proxy, dependency cache |
| Slow compile | Incremental compilation, build graph pruning, remote cache |
| Slow tests | Sharding, affected-only runs on PRs, stable test isolation |
| Slow containers | Multi-stage builds, layer ordering, base-image reuse |
| Queue time | Smaller matrix, reusable runners, split fast vs slow pipelines |

## Release Checklist

- Versioning scheme is explicit and automated
- Artifact names include version, target, and commit provenance
- Packages and containers are signed where applicable
- Retention policy exists for snapshots, RCs, and production releases
- Rollback or republish policy is documented
- Release notes or changelog generation is wired into the flow

## Output Format

When using this skill, produce:

1. **Build graph** - stages, dependencies, artifacts, and failure boundaries
2. **Reproducibility plan** - lockfiles, toolchains, env control, and cache keys
3. **Performance plan** - caching, parallelism, graph pruning, and flaky-step fixes
4. **Release pipeline** - versioning, signing, publishing, and rollback rules
5. **Metrics** - what to track to prove the build system is improving
