---
name: chaos-engineer
description: Chaos engineering: steady-state hypotheses, blast-radius control, fault injection experiments, rollback guards, and resilience validation.
---

# Chaos Engineer

Use controlled experiments to prove whether systems are actually resilient under failure. This skill favors hypothesis-driven experiments, small blast radius, explicit abort criteria, and follow-through that turns findings into better automation, runbooks, and architecture.

## When to Use

- Planning resilience experiments for production or staging systems
- Validating failover, retries, circuit breakers, bulkheads, or graceful degradation
- Stress-testing critical dependencies such as databases, queues, caches, or third-party APIs
- Building a safe chaos program with approvals, rollback, and observability
- Turning incident learnings into repeatable experiments
- Prioritizing reliability work based on measured failure behavior

## Workflow

1. Define the steady-state metric first.
   Pick a user-facing success signal such as checkout success rate, p95 latency, message delivery, or queue drain time.
2. Write a falsifiable hypothesis.
   Example: "If one cache node is lost, API latency stays under 300 ms and error rate stays under 1% because the service falls back to the primary store."
3. Minimize blast radius.
   Start with one service, one zone, one dependency, one customer segment, or one canary environment.
4. Add guardrails and abort criteria.
   Set stop conditions, owners, communications, and rollback steps before any fault injection begins.
5. Run one failure mode at a time.
   Isolate network loss, latency, packet corruption, CPU pressure, disk pressure, pod eviction, or dependency unavailability so results stay interpretable.
6. Turn findings into changes.
   Update retries, timeouts, capacity rules, runbooks, alerting, or architecture based on what the experiment exposed.

## Experiment Template

| Field | What Good Looks Like |
|---|---|
| Steady-state metric | User-facing and measurable in real time |
| Hypothesis | Specific enough to be proven wrong |
| Blast radius | Small, explicit, and reversible |
| Fault | One clear injected condition |
| Guardrail | Abort threshold tied to customer risk |
| Rollback | Immediate, owned, and rehearsed |

## Failure Modes to Test

- Dependency timeout or hard failure
- Partial packet loss or added latency between services
- Sudden traffic spike or queue backlog
- Pod, node, or zone failure
- Expired credentials or rotated secrets
- Cache eviction, replica lag, or stale reads

## Program Rules

- Never start with customer-wide experiments if you have not validated observability and rollback in a safer environment.
- Every experiment needs a clear owner, communication channel, and stop button.
- Do not run chaos only for spectacle. If no follow-up work will happen, the experiment is waste.
- Fold successful experiments into recurring game days and pre-release validation.
- Use incident data to choose experiments, not random curiosity.

## Output Format

When using this skill, produce:

1. **Experiment brief** - system, failure mode, hypothesis, and steady-state metric
2. **Safety controls** - blast radius, approvals, guardrails, and rollback plan
3. **Execution steps** - exact inject/observe/abort sequence
4. **Expected signals** - dashboards, alerts, logs, and traces to watch
5. **Follow-up actions** - architecture, runbook, alerting, or capacity changes
