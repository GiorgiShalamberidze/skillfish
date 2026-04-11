---
name: api-rate-limiter-designer
description: API rate limiting design: token bucket, leaky bucket, fixed and sliding windows, quota policies, 429 behavior, and multi-tenant enforcement.
---

# API Rate Limiter Designer

Design rate limiting that is fair, observable, and hard to bypass. This skill helps choose the right limiting algorithm, scope limits to the right actors and resources, define 429 semantics clearly, and implement distributed enforcement without surprising legitimate users.

## When to Use

- Designing or revising API rate limiting and quota policies
- Choosing between token bucket, leaky bucket, fixed window, and sliding window approaches
- Enforcing limits by IP, user, API key, tenant, endpoint, or cost unit
- Handling burst traffic, abuse prevention, and fairness across tenants
- Standardizing `429` responses, rate-limit headers, and retry guidance
- Reviewing distributed rate-limit storage and failure behavior

## Workflow

1. Define the actor and resource.
   Decide who is being limited and what scarce resource is being protected: requests, tokens, compute cost, writes, or downstream capacity.
2. Pick the fairness model.
   Separate anonymous, authenticated, internal, and premium traffic so one class does not starve another.
3. Choose the algorithm based on behavior.
   Burst-friendly limits usually want token bucket; strict sustained throughput often wants leaky bucket; fixed windows are simple but coarse.
4. Design the client contract.
   Define headers, retry semantics, and whether clients should back off, queue, or degrade gracefully.
5. Plan distributed enforcement.
   Choose storage, sharding, clock assumptions, and failure behavior for multi-instance environments.
6. Monitor and tune.
   Track false positives, top violators, hot keys, and user-impact metrics, not just total blocked requests.

## Algorithm Guide

| Algorithm | Best For | Tradeoff |
|---|---|---|
| Token bucket | APIs that allow short bursts but need steady protection | Slightly more stateful than fixed windows |
| Leaky bucket | Smoothing outbound or downstream throughput | Less burst-friendly for legitimate spikes |
| Fixed window | Simple coarse limits and low implementation complexity | Boundary effects allow burst abuse at window edges |
| Sliding window | Fairer request accounting over time | More storage and compute cost than fixed windows |

## Policy Rules

- Limit by the most meaningful identity available: tenant or API key beats raw IP for authenticated traffic.
- Separate read-heavy and write-heavy endpoints when their cost profiles differ.
- Make high-cost endpoints consume more budget than cheap endpoints when appropriate.
- Return clear `429` responses with machine-readable retry information.
- Decide whether enforcement should fail open or fail closed when the limiter store is degraded.
- Keep exemptions explicit and audited; "temporary" bypasses tend to become permanent.

## Client Contract

- Publish limit scope and units clearly
- Return remaining budget when it helps clients self-throttle
- Include `Retry-After` or equivalent guidance for backoff
- Document burst vs sustained behavior, not only "100 requests per minute"
- Tell clients whether limits are per key, per user, per tenant, or per endpoint

## Output Format

When using this skill, produce:

1. **Rate-limit model** - actor, protected resource, units, and fairness classes
2. **Algorithm choice** - selected approach and why it fits the traffic pattern
3. **Enforcement design** - storage, sharding, failure behavior, and bypass rules
4. **Client contract** - headers, `429` payload, retry guidance, and docs language
5. **Tuning plan** - metrics, abuse signals, false positives, and rollout strategy
