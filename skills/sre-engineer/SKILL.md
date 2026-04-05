---
name: sre-engineer
description: Site Reliability Engineering: SLOs/SLIs/error budgets, toil reduction, capacity planning, incident management, postmortem culture, and reliability automation.
---

# SRE Engineer

Comprehensive Site Reliability Engineering skill covering the full SRE discipline -- from defining and defending service reliability targets to building the operational culture and automation that keeps production systems healthy at scale.

This skill codifies practices from the Google SRE books, industry postmortem databases, and real-world operational patterns into actionable guidance for AI coding agents working on production infrastructure.

---

## Table of Contents

1. [SLOs, SLIs, and Error Budgets](#1-slos-slis-and-error-budgets)
2. [Toil Reduction](#2-toil-reduction)
3. [Capacity Planning](#3-capacity-planning)
4. [Incident Management](#4-incident-management)
5. [Postmortems](#5-postmortems)
6. [Monitoring and Alerting](#6-monitoring-and-alerting)
7. [Reliability Patterns](#7-reliability-patterns)
8. [On-Call](#8-on-call)

---

## When to Use

Use this skill when:
- Defining or revising SLOs/SLIs for a service
- Designing error budget policies and burn rate alerts
- Identifying and eliminating toil in operational workflows
- Building or reviewing capacity plans and load test strategies
- Setting up or improving incident response processes
- Writing or facilitating blameless postmortems
- Designing monitoring, alerting, and dashboards for production systems
- Implementing reliability patterns (circuit breakers, retries, load shedding)
- Building or refining on-call rotations and runbooks

Skip this skill when:
- The system is a prototype with no production traffic
- You need deep application-level debugging (use a language-specific skill)
- The task is purely CI/CD pipeline construction (use a DevOps skill)

---

## 1. SLOs, SLIs, and Error Budgets

### 1.1 Service Level Indicators (SLIs)

SLIs are the quantitative measurements of user-perceived service quality. Choose SLIs that directly reflect the user experience.

**Common SLI Types:**

| SLI Category | Metric | Measurement Method |
|---|---|---|
| Availability | Proportion of successful requests | `good_requests / total_requests` |
| Latency | Proportion of requests faster than threshold | `requests_below_threshold / total_requests` |
| Throughput | Data processing rate | `bytes_processed / time_window` |
| Correctness | Proportion of correct responses | `correct_responses / total_responses` |
| Freshness | Proportion of data updated within deadline | `fresh_records / total_records` |
| Durability | Proportion of data retained over time | `retained_objects / total_objects` |

**Example -- defining an SLI in Prometheus:**

```yaml
# SLI: Request availability for the payments API
- record: sli:payment_api:availability
  expr: |
    sum(rate(http_requests_total{service="payments", code!~"5.."}[5m]))
    /
    sum(rate(http_requests_total{service="payments"}[5m]))

# SLI: Latency -- proportion of requests under 300ms
- record: sli:payment_api:latency_good
  expr: |
    sum(rate(http_request_duration_seconds_bucket{service="payments", le="0.3"}[5m]))
    /
    sum(rate(http_request_duration_seconds_count{service="payments"}[5m]))
```

### 1.2 Setting SLOs

SLOs are the target values for your SLIs over a rolling time window. They represent the reliability contract between the service team and its users.

**SLO Design Principles:**
- Start with user expectations, not engineering aspirations
- Use rolling windows (typically 28 or 30 days) rather than calendar months
- Set achievable targets -- 99.9% is dramatically different from 99.99%
- Document SLOs alongside the service, not in a separate wiki

**Example SLO Specification:**

```yaml
service: payments-api
slos:
  - name: availability
    description: "Proportion of non-5xx responses"
    sli: sli:payment_api:availability
    target: 0.999          # 99.9%
    window: 30d
    
  - name: latency_p99
    description: "99th percentile response time under 500ms"
    sli: sli:payment_api:latency_good
    target: 0.99           # 99%
    window: 30d

  - name: latency_p50
    description: "50th percentile response time under 100ms"
    sli: sli:payment_api:latency_p50_good
    target: 0.995          # 99.5%
    window: 30d
```

**Downtime Budget Table:**

| SLO Target | Monthly Error Budget | Downtime Equivalent |
|---|---|---|
| 99% | 1% | ~7h 18m/month |
| 99.5% | 0.5% | ~3h 39m/month |
| 99.9% | 0.1% | ~43m/month |
| 99.95% | 0.05% | ~21m/month |
| 99.99% | 0.01% | ~4m 23s/month |

### 1.3 Error Budget Policies

Error budgets transform SLOs from aspirational targets into actionable operational controls.

**Policy Template:**

```markdown
## Error Budget Policy: [Service Name]

**When error budget is healthy (> 50% remaining):**
- Normal development velocity
- Feature work proceeds as planned
- Standard deployment cadence

**When error budget is draining (25-50% remaining):**
- Increase code review rigor for reliability-sensitive changes
- Prioritize reliability fixes in the current sprint
- Conduct targeted load testing before deployments

**When error budget is critical (< 25% remaining):**
- Freeze non-critical feature deployments
- Dedicate 50% of engineering time to reliability
- Require SRE approval for all production changes

**When error budget is exhausted (0% remaining):**
- Full feature freeze until budget recovers
- 100% of engineering effort on reliability
- Mandatory postmortem for budget-consuming incidents
- Escalate to engineering leadership
```

### 1.4 Burn Rate Alerts

Burn rate alerts detect when you are consuming your error budget faster than sustainable.

**Multi-window burn rate strategy:**

```yaml
# Fast burn -- consuming 30-day budget in 1 hour
# Short window catches acute outages
- alert: SLOHighBurnRate_PaymentsAPI
  expr: |
    (
      1 - sli:payment_api:availability:rate5m
    ) > (14.4 * (1 - 0.999))
  for: 2m
  labels:
    severity: critical
    window: "5m/1h"
  annotations:
    summary: "Payments API burning error budget 14.4x faster than allowed"

# Slow burn -- consuming 30-day budget in 3 days
# Longer window catches sustained degradation
- alert: SLOSlowBurnRate_PaymentsAPI
  expr: |
    (
      1 - sli:payment_api:availability:rate1h
    ) > (1 * (1 - 0.999))
  for: 30m
  labels:
    severity: warning
    window: "1h/6h"
  annotations:
    summary: "Payments API sustained elevated error rate eroding budget"
```

### Anti-Patterns

- **Vanity SLOs:** Setting 99.999% when the downstream dependencies only offer 99.9%
- **SLO without teeth:** Defining SLOs but never enforcing the error budget policy
- **Too many SLIs:** Tracking 30 SLIs per service -- focus on 2-5 that reflect user experience
- **Internal-only SLIs:** Measuring CPU utilization as an SLI instead of user-facing latency
- **Calendar windows:** Using calendar months causes reset-day gaming and end-of-month panic

---

## 2. Toil Reduction

### 2.1 Identifying Toil

Toil is manual, repetitive, automatable work that scales linearly with service growth and has no lasting value.

**Toil Characteristics Checklist:**

| Characteristic | Question | Toil? |
|---|---|---|
| Manual | Does a human perform the steps? | Yes |
| Repetitive | Does this recur on a schedule or trigger? | Yes |
| Automatable | Could software perform this? | Yes |
| Tactical | Is this reactive, not strategic? | Yes |
| No lasting value | Does the service return to the same state after? | Yes |
| O(n) with growth | Does effort scale with service/user count? | Yes |

**Example Toil Audit:**

```markdown
## Toil Audit: Q1 2026

| Task | Frequency | Time/Occurrence | Monthly Hours | Automatable? | Priority |
|---|---|---|---|---|---|
| Certificate rotation | Quarterly | 4h | 1.3h | Yes - cert-manager | High |
| DB user provisioning | 5x/week | 30m | 10h | Yes - Terraform | High |
| Log volume cleanup | Daily | 15m | 7.5h | Yes - retention policy | Medium |
| Deploy approval clicks | 10x/week | 5m | 3.3h | Yes - policy engine | Medium |
| Capacity report gen | Monthly | 3h | 3h | Yes - script | Low |
```

### 2.2 Automation Priorities

Prioritize toil elimination by impact and effort.

**Priority Framework:**

```
Priority Score = (Monthly Hours Saved * Frequency Score * Risk Reduction) / Automation Effort

Where:
  Frequency Score: Daily=5, Weekly=3, Monthly=1
  Risk Reduction:  Eliminates outage risk=3, Reduces errors=2, Convenience=1
  Automation Effort: Days to automate (lower is better)
```

**Example -- automating certificate rotation:**

```bash
#!/bin/bash
# Before: manual certificate rotation (4h quarterly, error-prone)
# After: cert-manager handles it automatically

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Define a ClusterIssuer for Let's Encrypt
cat <<'YAML' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: sre-team@company.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
YAML

# Annotate ingress for automatic cert provisioning
kubectl annotate ingress payments-api \
  cert-manager.io/cluster-issuer=letsencrypt-prod
```

### 2.3 Toil Budgets

The Google SRE model targets a maximum of 50% toil per SRE. Track it.

```markdown
## SRE Toil Budget: Team Alpha

Target: <= 50% toil, >= 50% engineering work

| Engineer | Toil Hours/Week | Engineering Hours/Week | Toil % | Status |
|---|---|---|---|---|
| Alice | 12 | 28 | 30% | Healthy |
| Bob | 22 | 18 | 55% | OVER BUDGET |
| Carol | 18 | 22 | 45% | At risk |

Action: Bob's certificate rotation and DB provisioning toil must be automated this quarter.
```

### 2.4 Measuring Impact

Track toil reduction over time to demonstrate value and maintain momentum.

```yaml
# Toil reduction metrics to track
metrics:
  - name: toil_hours_per_engineer_weekly
    description: "Hours spent on toil per engineer per week"
    target: "< 20 hours (50% of 40-hour week)"
    
  - name: toil_ratio
    description: "Percentage of total time spent on toil"
    target: "< 50%"
    
  - name: automation_coverage
    description: "Percentage of identified toil tasks that are automated"
    target: "> 80% within 6 months of identification"
    
  - name: mean_time_to_automate
    description: "Average days from toil identification to automation"
    target: "< 30 days for high-priority items"
```

### Anti-Patterns

- **Automating the wrong thing:** Building elaborate automation for a task done once a year
- **Toil blindness:** Accepting toil as "just how things work" without measuring it
- **Hero culture:** Praising engineers who manually handle repetitive firefighting
- **Partial automation:** Automating 80% of a task but leaving manual steps that still require human intervention
- **Not tracking toil:** If you don't measure it, you cannot reduce it systematically

---

## 3. Capacity Planning

### 3.1 Demand Forecasting

Predict future resource needs based on historical trends, business plans, and organic growth.

**Forecasting Approach:**

```python
# Simple linear regression forecast for capacity planning
import numpy as np
from datetime import datetime, timedelta

def forecast_demand(historical_data, forecast_days=90):
    """
    historical_data: list of (date, value) tuples
    Returns: projected demand at forecast horizon
    """
    days = np.array([(d - historical_data[0][0]).days for d, _ in historical_data])
    values = np.array([v for _, v in historical_data])
    
    # Fit linear trend
    slope, intercept = np.polyfit(days, values, 1)
    
    # Add safety margin (20% headroom)
    forecast_day = days[-1] + forecast_days
    projected = slope * forecast_day + intercept
    with_headroom = projected * 1.20
    
    return {
        "current": values[-1],
        "projected_raw": round(projected, 2),
        "projected_with_headroom": round(with_headroom, 2),
        "daily_growth_rate": round(slope, 4),
        "forecast_date": (historical_data[-1][0] + timedelta(days=forecast_days)).isoformat()
    }
```

**Business Event Calendar:**

```markdown
## Capacity Calendar: 2026

| Event | Date | Expected Traffic Multiplier | Pre-scaling Required |
|---|---|---|---|
| Product launch v3.0 | March 15 | 3x baseline | 1 week before |
| Marketing campaign | April 1-15 | 2x baseline | 2 days before |
| Black Friday | November 27 | 10x baseline | 2 weeks before |
| Year-end processing | Dec 28-31 | 5x baseline | 1 week before |
```

### 3.2 Load Testing

Validate capacity assumptions with realistic load tests before they matter.

**Load Test Strategy:**

```yaml
load_test_plan:
  name: "Payments API capacity validation"
  
  baseline:
    description: "Normal day traffic pattern"
    target_rps: 500
    duration: 30m
    ramp_up: 5m
    
  peak:
    description: "Expected peak (2x baseline)"
    target_rps: 1000
    duration: 15m
    ramp_up: 3m
    
  stress:
    description: "Beyond expected peak (3x baseline)"
    target_rps: 1500
    duration: 10m
    ramp_up: 2m
    
  soak:
    description: "Sustained load for memory leak detection"
    target_rps: 500
    duration: 4h
    ramp_up: 5m

  success_criteria:
    - p99_latency_ms: "< 500"
    - error_rate: "< 0.1%"
    - cpu_utilization: "< 70%"
    - memory_utilization: "< 80%"
    - no_oom_kills: true
```

**Example k6 load test script:**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '5m', target: 500 },   // ramp to baseline
    { duration: '30m', target: 500 },   // hold baseline
    { duration: '3m', target: 1000 },   // ramp to peak
    { duration: '15m', target: 1000 },  // hold peak
    { duration: '5m', target: 0 },      // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],
    errors: ['rate<0.001'],
  },
};

export default function () {
  const res = http.post('https://api.example.com/v1/payments', JSON.stringify({
    amount: Math.floor(Math.random() * 10000),
    currency: 'USD',
  }), { headers: { 'Content-Type': 'application/json' } });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'latency < 500ms': (r) => r.timings.duration < 500,
  });

  errorRate.add(res.status >= 400);
  sleep(1);
}
```

### 3.3 Resource Modeling

Map service demand to infrastructure resources.

```markdown
## Resource Model: Payments API

### Per-Instance Capacity
- Max concurrent connections: 200
- Throughput: ~100 RPS per pod (at p99 < 300ms)
- Memory: 512Mi baseline + ~1Mi per 10 concurrent connections
- CPU: 250m baseline + ~50m per 100 RPS

### Scaling Formula
Required pods = ceil(target_RPS / 100) + 2 (headroom)

### Current vs Required

| Scenario | Target RPS | Pods Needed | Pods Deployed | Status |
|---|---|---|---|---|
| Normal | 500 | 7 | 8 | OK |
| Peak (2x) | 1000 | 12 | 8 | UNDER-PROVISIONED |
| Black Friday (10x) | 5000 | 52 | 8 | NEEDS PRE-SCALING |
```

### 3.4 Scaling Strategies

```yaml
horizontal_pod_autoscaler:
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: payments-api
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: payments-api
    minReplicas: 4
    maxReplicas: 60
    behavior:
      scaleUp:
        stabilizationWindowSeconds: 60
        policies:
          - type: Percent
            value: 50          # scale up by 50% of current
            periodSeconds: 60
      scaleDown:
        stabilizationWindowSeconds: 300
        policies:
          - type: Percent
            value: 10          # scale down slowly
            periodSeconds: 120
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 60
      - type: Pods
        pods:
          metric:
            name: http_requests_per_second
          target:
            type: AverageValue
            averageValue: "80"
```

### Anti-Patterns

- **Scaling on CPU alone:** CPU is a lagging indicator; scale on request rate or queue depth instead
- **No headroom:** Running at 90% utilization leaves no room for spikes
- **Annual capacity reviews:** Review quarterly at minimum; monthly for fast-growing services
- **Ignoring dependencies:** Your service scales, but the database behind it does not
- **Load testing in isolation:** Test the full request path including downstream services

---

## 4. Incident Management

### 4.1 Response Process

A structured incident response ensures consistent, efficient handling regardless of who is on-call.

**Incident Lifecycle:**

```
Detection -> Triage -> Mitigation -> Resolution -> Postmortem
    |           |           |             |            |
  Alert      Severity    Stop the     Fix root      Learn and
  fires      assigned    bleeding     cause         improve
```

**Initial Response Checklist:**

```markdown
## Incident Response: First 5 Minutes

1. [ ] Acknowledge the alert -- stop the pager
2. [ ] Open an incident channel (e.g., #inc-YYYYMMDD-brief-description)
3. [ ] Declare severity level (SEV1-4)
4. [ ] Assign Incident Commander (IC) role
5. [ ] Post initial status: what is known, what is impacted, who is investigating
6. [ ] If SEV1/SEV2: page additional responders per escalation policy
7. [ ] Start a shared incident document for real-time notes
```

### 4.2 Severity Levels

```markdown
## Severity Definitions

| Level | Impact | Examples | Response Time | Comms Cadence |
|---|---|---|---|---|
| SEV1 | Full service outage, data loss risk, revenue impact | Payments down, DB corruption | Immediate (< 5 min) | Every 15 min |
| SEV2 | Significant degradation, subset of users impacted | High latency, partial outage | < 15 min | Every 30 min |
| SEV3 | Minor degradation, workaround exists | Slow dashboard, non-critical feature down | < 1 hour | Every 2 hours |
| SEV4 | Cosmetic issue, no user impact | Internal tool glitch, log noise | Next business day | End of day |
```

### 4.3 Incident Commander Role

The IC owns the incident process, not the technical fix. Their job is coordination.

**IC Responsibilities:**

```markdown
## Incident Commander Duties

**DO:**
- Maintain the single source of truth (incident channel + doc)
- Assign clear roles: IC, Tech Lead, Communications Lead, Scribe
- Make decisions about escalation and severity changes
- Keep stakeholders updated at regular intervals
- Call for additional help early rather than late
- Declare incident resolved and schedule postmortem

**DON'T:**
- Debug the issue yourself (delegate to Tech Lead)
- Allow side conversations outside the incident channel
- Skip status updates because "we're still investigating"
- Change severity without documenting the reason
- Forget to hand off IC role if you need to step away
```

### 4.4 Communication Templates

**Internal Status Update:**

```markdown
## Incident Update: [INC-2026-0405] Payments API Elevated Error Rate

**Severity:** SEV2
**Status:** Investigating
**Duration:** 45 minutes
**Impact:** ~15% of payment requests returning 503 errors
**Current Actions:** 
- Identified unhealthy database replica; failover in progress
- Traffic rerouted away from affected availability zone
**Next Update:** 30 minutes or sooner if status changes
**IC:** @alice | **Tech Lead:** @bob
```

**External Status Page Update:**

```markdown
**[Investigating] Payment Processing Delays**

We are currently investigating elevated error rates affecting 
payment processing. Some transactions may fail or experience 
delays. Our engineering team is actively working on resolution.

We will provide an update within 30 minutes.
```

### Anti-Patterns

- **Freelancing:** Multiple engineers making production changes without IC coordination
- **Alert fatigue silence:** Ignoring alerts because "they always fire"
- **No severity levels:** Treating every issue as SEV1 burns out the team
- **Communication gaps:** Going dark for 2 hours during an active incident
- **Blame in the channel:** "Who deployed this?" during an active incident -- save it for the postmortem
- **No incident channel:** Debugging in DMs fragments context and blocks others from helping

---

## 5. Postmortems

### 5.1 Blameless Postmortem Template

```markdown
# Postmortem: [INC-2026-0405] Payments API Elevated Error Rate

## Incident Summary
- **Date:** 2026-04-05
- **Duration:** 2 hours 15 minutes (09:30 - 11:45 UTC)
- **Severity:** SEV2
- **Impact:** 15% of payment requests failed; estimated $45,000 revenue impact
- **Detection:** Automated SLO burn rate alert
- **IC:** Alice Chen

## Timeline (all times UTC)

| Time | Event |
|---|---|
| 09:28 | SLO burn rate alert fires for payments-api availability |
| 09:30 | On-call engineer Alice acknowledges alert, opens #inc-20260405-payments |
| 09:35 | Initial triage: elevated 503s from payments-api pods in us-east-1c |
| 09:42 | Identified: database replica db-replica-3 in us-east-1c is unresponsive |
| 09:50 | Attempted restart of db-replica-3 -- failed, disk I/O errors |
| 10:05 | Decision: failover traffic from us-east-1c to us-east-1a and 1b |
| 10:15 | Traffic rerouted; error rate dropping |
| 10:30 | Error rate back to baseline; monitoring for stability |
| 11:45 | Incident resolved; db-replica-3 replaced with new instance |

## Contributing Factors
1. **Disk degradation on db-replica-3** -- EBS volume showed increasing I/O latency over 48 hours before failure, but no alert was configured for gradual disk degradation
2. **No automatic failover for read replicas** -- Write primary has automatic failover, but read replicas require manual intervention
3. **Health checks too lenient** -- Replica was marked "healthy" despite serving errors because health check only verified TCP connectivity

## Lessons Learned

**What went well:**
- SLO burn rate alert detected the issue within 2 minutes
- IC process followed smoothly; clear roles assigned
- Traffic rerouting was executed in under 30 minutes

**What went poorly:**
- No alert on gradual disk degradation -- 48 hours of warning signals missed
- Manual replica failover added 25 minutes to resolution
- Runbook for database replica failure was outdated

## Action Items

| Action | Owner | Priority | Due Date | Ticket |
|---|---|---|---|---|
| Add EBS I/O latency alerting with 24h trend detection | Bob | High | 2026-04-12 | OPS-1234 |
| Implement automatic read replica failover | Carol | High | 2026-04-19 | OPS-1235 |
| Update health checks to verify query execution, not just TCP | Alice | Medium | 2026-04-15 | OPS-1236 |
| Update database failure runbook | Dave | Medium | 2026-04-10 | OPS-1237 |
| Add disk health to capacity dashboard | Bob | Low | 2026-04-30 | OPS-1238 |
```

### 5.2 Action Item Quality

Good action items are specific, assigned, time-bound, and tracked.

**Good Action Items:**
- "Add EBS I/O latency alerting with 24h trend detection" -- specific, measurable
- "Implement automatic read replica failover with < 30s switchover" -- has a success criterion
- "Update health checks to execute `SELECT 1` instead of TCP connect" -- prescriptive

**Bad Action Items:**
- "Improve monitoring" -- too vague
- "Be more careful with deployments" -- not actionable
- "Look into database reliability" -- no clear deliverable or owner

### 5.3 Contributing Factor Analysis

Use the "5 Whys" technique to dig beyond surface-level causes.

```markdown
## 5 Whys: Database Replica Failure

1. **Why did payments fail?**
   Because the database replica serving reads was returning errors.

2. **Why was the replica returning errors?**
   Because the underlying EBS volume had degraded I/O performance.

3. **Why wasn't the degradation detected earlier?**
   Because there was no alert on gradual I/O latency increases.

4. **Why was there no alert for gradual degradation?**
   Because our disk monitoring only checks for binary up/down status, 
   not performance trends.

5. **Why does our monitoring lack trend-based alerting?**
   Because we built monitoring reactively after incidents and never 
   did a systematic review of leading indicators for disk failure.

Root cause: Monitoring strategy lacks proactive, trend-based alerting 
for infrastructure degradation.
```

### 5.4 Review Meetings

```markdown
## Postmortem Review Meeting Guide

**Attendees:** Incident responders + service owner + SRE lead
**Duration:** 45-60 minutes
**Cadence:** Within 5 business days of incident resolution

**Agenda:**
1. (5 min)  Read the postmortem silently -- no pre-reading required
2. (10 min) Walk through the timeline -- fill gaps, correct details
3. (15 min) Discuss contributing factors -- add any missed factors
4. (10 min) Review action items -- confirm owners, priorities, due dates
5. (5 min)  Identify systemic patterns -- does this connect to other incidents?
6. (5 min)  Close -- publish final postmortem, create tickets

**Ground Rules:**
- No blame. Focus on systems and processes, not individuals.
- "How did the system allow this to happen?" not "Who caused this?"
- Everyone's recollection is valid -- memory is imperfect under stress
- Action items must be tracked in the team's issue tracker
```

### Anti-Patterns

- **Blame culture:** Naming individuals as root causes destroys trust and hides systemic issues
- **Action item graveyard:** Writing action items that never get prioritized or completed
- **Postmortem fatigue:** Requiring postmortems for every SEV4 -- reserve for SEV1/SEV2 and learning-rich SEV3s
- **Root cause singular:** Complex incidents have multiple contributing factors, not one root cause
- **Skipping the review meeting:** The postmortem document without discussion misses half the value

---

## 6. Monitoring and Alerting

### 6.1 USE and RED Methods

Two complementary frameworks for systematic monitoring coverage.

**USE Method (for resources -- CPU, memory, disk, network):**

| Signal | Definition | Example Metric |
|---|---|---|
| **U**tilization | Percentage of resource capacity used | `node_cpu_seconds_total` (rate) |
| **S**aturation | Degree of queued or deferred work | `node_cpu_load_average_5m` |
| **E**rrors | Count of error events | `node_disk_io_errors_total` |

**RED Method (for request-driven services):**

| Signal | Definition | Example Metric |
|---|---|---|
| **R**ate | Requests per second | `http_requests_total` (rate) |
| **E**rrors | Failed requests per second | `http_requests_total{code=~"5.."}` (rate) |
| **D**uration | Distribution of request latency | `http_request_duration_seconds` (histogram) |

**Example Prometheus recording rules implementing both methods:**

```yaml
groups:
  - name: red_method
    interval: 30s
    rules:
      # Rate
      - record: service:http_requests:rate5m
        expr: sum by (service)(rate(http_requests_total[5m]))
      
      # Errors
      - record: service:http_errors:rate5m
        expr: sum by (service)(rate(http_requests_total{code=~"5.."}[5m]))
      
      # Error ratio
      - record: service:http_error_ratio:rate5m
        expr: service:http_errors:rate5m / service:http_requests:rate5m
      
      # Duration (p50, p90, p99)
      - record: service:http_duration:p99_5m
        expr: |
          histogram_quantile(0.99,
            sum by (service, le)(rate(http_request_duration_seconds_bucket[5m]))
          )

  - name: use_method
    interval: 30s
    rules:
      # CPU Utilization
      - record: instance:cpu_utilization:ratio
        expr: 1 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[5m]))
      
      # Memory Utilization
      - record: instance:memory_utilization:ratio
        expr: |
          1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
      
      # Disk Utilization
      - record: instance:disk_utilization:ratio
        expr: |
          1 - (node_filesystem_avail_bytes{mountpoint="/"} 
               / node_filesystem_size_bytes{mountpoint="/"})
```

### 6.2 Alert Design

Every alert must be actionable, and every page must be urgent.

**Alert Quality Checklist:**

```markdown
For each alert, answer:
1. [ ] Does this alert indicate a real user impact or imminent risk?
2. [ ] Is there a clear, documented action to take when it fires?
3. [ ] Can the on-call engineer resolve this without escalation?
4. [ ] Does this alert fire infrequently enough to be meaningful?
5. [ ] Is the threshold based on data, not guesswork?
6. [ ] Does it include enough context to begin investigation?
```

**Well-Designed Alert:**

```yaml
- alert: PaymentsAPIHighErrorRate
  expr: service:http_error_ratio:rate5m{service="payments"} > 0.01
  for: 5m
  labels:
    severity: critical
    team: payments
    playbook: "https://runbooks.internal/payments-high-error-rate"
  annotations:
    summary: "Payments API error rate {{ $value | humanizePercentage }} exceeds 1% for 5m"
    impact: "Users experiencing payment failures"
    investigation: |
      1. Check deployment history: did a recent rollout cause this?
      2. Check downstream dependencies: kubectl get pods -l app=payments-db
      3. Check for resource saturation: Grafana dashboard link
    dashboard: "https://grafana.internal/d/payments-overview"
```

### 6.3 Paging Policies

```yaml
paging_policy:
  critical:
    route_to: on-call-primary
    escalate_after: 5m
    escalate_to: on-call-secondary
    final_escalation: 15m
    final_escalate_to: engineering-manager
    channels: [pagerduty, sms, phone]
    
  warning:
    route_to: on-call-primary
    escalate_after: 30m
    escalate_to: on-call-secondary
    channels: [slack, email]
    suppress_outside_hours: true
    
  info:
    route_to: team-slack-channel
    channels: [slack]
    no_paging: true
    
  dedup_rules:
    - group_by: [alertname, service]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
```

### 6.4 Prometheus and Grafana Patterns

**Grafana Dashboard Structure:**

```markdown
## Dashboard Layout: Service Overview

Row 1: SLO Summary
  - SLO compliance gauge (current error budget %)
  - Error budget burn rate (time series)
  - SLO status (traffic light: green/amber/red)

Row 2: RED Signals
  - Request rate (time series)
  - Error rate (time series + threshold line)
  - Latency distribution (heatmap)
  - p50/p90/p99 latency (time series)

Row 3: Dependencies
  - Downstream service latency (per dependency)
  - Database query duration (p99)
  - Cache hit ratio

Row 4: Infrastructure
  - Pod CPU/memory utilization
  - Pod count (desired vs available)
  - Network I/O
  - Disk utilization (if stateful)
```

### Anti-Patterns

- **Alert on everything:** If the on-call gets 50 pages per shift, they learn to ignore them
- **No runbook link:** Alert fires at 3 AM and the engineer has no idea where to start
- **Threshold without data:** Setting CPU alert at 80% "because it seems right" -- use historical baselines
- **Dashboard sprawl:** 40 dashboards with no clear starting point -- create a service index dashboard
- **Monitoring the monitor:** Spending more time maintaining Prometheus than the services it monitors
- **Missing SLO context:** Alerting on raw error counts instead of SLO burn rates

---

## 7. Reliability Patterns

### 7.1 Circuit Breakers

Prevent cascading failures by stopping requests to a failing dependency.

**Circuit Breaker States:**

```
    [CLOSED] --failures exceed threshold--> [OPEN]
       ^                                       |
       |                                  timeout expires
       |                                       |
       +---probe succeeds--- [HALF-OPEN] <----+
                              |
                         probe fails --> [OPEN]
```

**Implementation Example:**

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30, 
                 half_open_max_calls=3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_calls = 0
        self.lock = Lock()

    def call(self, func, *args, **kwargs):
        with self.lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                    self.half_open_calls = 0
                else:
                    raise CircuitOpenError(
                        f"Circuit open, retry after {self.recovery_timeout}s"
                    )
            
            if self.state == CircuitState.HALF_OPEN:
                if self.half_open_calls >= self.half_open_max_calls:
                    raise CircuitOpenError("Half-open call limit reached")
                self.half_open_calls += 1

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        with self.lock:
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED
            self.failure_count = 0

    def _on_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
```

### 7.2 Retries with Exponential Backoff

Retry transient failures without overwhelming the failing service.

```python
import random
import time

def retry_with_backoff(func, max_retries=3, base_delay=1.0, 
                       max_delay=60.0, jitter=True):
    """
    Retry a function with exponential backoff and optional jitter.
    
    Delay formula: min(base_delay * 2^attempt + jitter, max_delay)
    """
    for attempt in range(max_retries + 1):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_retries:
                raise MaxRetriesExceeded(f"Failed after {max_retries} retries") from e
            
            delay = min(base_delay * (2 ** attempt), max_delay)
            
            if jitter:
                # Full jitter: random value between 0 and calculated delay
                delay = random.uniform(0, delay)
            
            print(f"Attempt {attempt + 1} failed: {e}. "
                  f"Retrying in {delay:.2f}s...")
            time.sleep(delay)
```

**Retry Budget Pattern:**

```python
class RetryBudget:
    """Limit total retries across all clients to prevent retry storms."""
    
    def __init__(self, max_retry_ratio=0.10, window_seconds=60):
        self.max_retry_ratio = max_retry_ratio
        self.window_seconds = window_seconds
        self.requests = []
        self.retries = []
    
    def can_retry(self):
        now = time.time()
        cutoff = now - self.window_seconds
        
        # Clean old entries
        self.requests = [t for t in self.requests if t > cutoff]
        self.retries = [t for t in self.retries if t > cutoff]
        
        if len(self.requests) == 0:
            return True
        
        retry_ratio = len(self.retries) / len(self.requests)
        return retry_ratio < self.max_retry_ratio
    
    def record_request(self):
        self.requests.append(time.time())
    
    def record_retry(self):
        self.retries.append(time.time())
```

### 7.3 Load Shedding

When overloaded, intentionally reject some requests to protect the system and serve the rest well.

```python
class LoadShedder:
    """
    Shed load based on current server utilization.
    Priority levels: 0 (critical) to 3 (best-effort).
    """
    
    PRIORITY_THRESHOLDS = {
        0: 0.95,   # Critical: shed only above 95% utilization
        1: 0.85,   # High: shed above 85%
        2: 0.70,   # Medium: shed above 70%
        3: 0.50,   # Low/best-effort: shed above 50%
    }
    
    def should_serve(self, request_priority: int, 
                     current_utilization: float) -> bool:
        threshold = self.PRIORITY_THRESHOLDS.get(request_priority, 0.50)
        
        if current_utilization >= threshold:
            # Probabilistic shedding -- gradually increase drop rate
            overage = current_utilization - threshold
            max_overage = 1.0 - threshold
            drop_probability = overage / max_overage if max_overage > 0 else 1.0
            return random.random() > drop_probability
        
        return True
```

**Kubernetes-Level Load Shedding:**

```yaml
# Pod Disruption Budget -- protect minimum availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments-api-pdb
spec:
  minAvailable: "60%"
  selector:
    matchLabels:
      app: payments-api

---
# Resource limits prevent one pod from starving others
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-api-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

### 7.4 Graceful Degradation

When a dependency fails, serve a reduced experience rather than failing completely.

```python
class PaymentService:
    """Example of graceful degradation with fallback behavior."""
    
    def get_exchange_rate(self, currency_pair: str) -> float:
        """
        Primary: real-time rate from exchange API
        Fallback 1: cached rate (< 5 minutes old)
        Fallback 2: last known rate from database
        Fallback 3: reject transaction (safe failure)
        """
        # Primary source
        try:
            rate = self.exchange_api.get_rate(currency_pair)
            self.cache.set(f"rate:{currency_pair}", rate, ttl=300)
            return rate
        except ExchangeAPIError:
            pass
        
        # Fallback 1: recent cache
        cached = self.cache.get(f"rate:{currency_pair}")
        if cached and cached.age_seconds < 300:
            self.metrics.increment("exchange_rate.fallback.cache")
            return cached.value
        
        # Fallback 2: database (may be stale)
        db_rate = self.db.get_latest_rate(currency_pair)
        if db_rate and db_rate.age_seconds < 3600:
            self.metrics.increment("exchange_rate.fallback.database")
            return db_rate.value
        
        # Fallback 3: safe failure -- reject rather than use stale data
        self.metrics.increment("exchange_rate.fallback.reject")
        raise StaleDataError(
            f"No reliable exchange rate for {currency_pair}. "
            f"Last known rate is {db_rate.age_seconds / 3600:.1f}h old."
        )
```

### Anti-Patterns

- **Retrying non-idempotent operations:** Retrying a payment charge without idempotency keys causes double charges
- **Unbounded retries:** Retrying forever creates a retry storm that prevents recovery
- **Circuit breaker too sensitive:** Opening the circuit on a single failure causes unnecessary outages
- **No fallback for degradation:** Circuit opens, but the service returns a 500 instead of a degraded response
- **Load shedding without priority:** Randomly dropping critical requests while serving best-effort traffic
- **Missing jitter:** All clients retry at the same time, creating thundering herd spikes

---

## 8. On-Call

### 8.1 Rotation Design

```yaml
on_call_rotation:
  name: "Payments Platform"
  timezone: "UTC"
  
  primary:
    schedule: weekly
    rotation_day: monday
    handoff_time: "10:00"
    team_size: 6    # each engineer on-call ~every 6 weeks
    
  secondary:
    schedule: weekly
    offset: 1       # previous week's primary becomes secondary
    
  overrides:
    - Allow shift swaps with 24h notice
    - Holidays require opt-in volunteers or skip rotation
    - New team members shadow for 2 rotations before going primary
    
  compensation:
    weekday_on_call: "per-day stipend or comp time"
    weekend_on_call: "2x weekday rate"
    incident_response: "additional per-incident compensation for after-hours pages"
```

### 8.2 Runbook Structure

Every paging alert must link to a runbook. Every runbook follows the same structure.

```markdown
# Runbook: Payments API High Error Rate

## Alert
- **Name:** PaymentsAPIHighErrorRate
- **Severity:** Critical
- **Condition:** Error rate > 1% for 5 minutes

## Impact
- Users cannot complete payment transactions
- Revenue impact: ~$500/minute during peak hours

## Quick Diagnosis (< 5 minutes)

1. Check if a recent deployment caused this:
   ```bash
   kubectl rollout history deployment/payments-api -n payments
   ```

2. Check pod health:
   ```bash
   kubectl get pods -l app=payments-api -n payments
   kubectl top pods -l app=payments-api -n payments
   ```

3. Check downstream dependencies:
   ```bash
   # Database connectivity
   kubectl exec -it deploy/payments-api -n payments -- \
     pg_isready -h payments-db.internal -p 5432
   
   # Payment processor API
   curl -s https://api.stripe.com/healthcheck
   ```

4. Check recent error logs:
   ```bash
   kubectl logs -l app=payments-api -n payments --tail=100 \
     | grep -i error | tail -20
   ```

## Mitigation Steps

### If caused by bad deployment:
```bash
kubectl rollout undo deployment/payments-api -n payments
# Verify rollback
kubectl rollout status deployment/payments-api -n payments
```

### If caused by database issues:
```bash
# Check replica lag
kubectl exec -it payments-db-0 -n payments -- \
  psql -c "SELECT client_addr, state, sent_lsn, write_lsn, 
           replay_lsn FROM pg_stat_replication;"

# Failover to standby if primary is unhealthy
# (Follow database failover runbook: link)
```

### If caused by resource exhaustion:
```bash
# Scale up immediately
kubectl scale deployment/payments-api -n payments --replicas=12

# Check HPA status
kubectl get hpa payments-api -n payments
```

## Escalation
- **L1 (on-call):** Follow steps above
- **L2 (15 min no resolution):** Page payments team lead
- **L3 (30 min no resolution):** Page engineering manager + database team

## Resolution Verification
```bash
# Confirm error rate has returned to baseline
curl -s 'http://prometheus:9090/api/v1/query?query=service:http_error_ratio:rate5m{service="payments"}' \
  | jq '.data.result[0].value[1]'
# Expected: < 0.001 (0.1%)
```

## Post-Incident
- [ ] Update incident channel with resolution details
- [ ] Schedule postmortem if SEV1 or SEV2
- [ ] Update this runbook if steps were missing or wrong
```

### 8.3 Escalation Policies

```yaml
escalation_policy:
  name: "Payments Platform Escalation"
  
  levels:
    - level: 1
      description: "On-call primary"
      targets: [on-call-primary]
      timeout: 5m
      
    - level: 2
      description: "On-call secondary"
      targets: [on-call-secondary]
      timeout: 10m
      
    - level: 3
      description: "Team lead + secondary team"
      targets: [team-lead, platform-on-call]
      timeout: 15m
      
    - level: 4
      description: "Engineering manager"
      targets: [engineering-manager]
      timeout: 30m
      note: "For SEV1 incidents, skip directly to level 3"

  sev1_override:
    description: "SEV1 pages all levels simultaneously"
    targets: [on-call-primary, on-call-secondary, team-lead]
    additional_after_15m: [engineering-manager, vp-engineering]
```

### 8.4 Reducing Pager Fatigue

Pager fatigue is the number one reason SREs burn out and leave.

**Pager Health Metrics:**

```markdown
## On-Call Health Report: Q1 2026

| Metric | Target | Actual | Status |
|---|---|---|---|
| Pages per shift | < 2 | 1.3 | Healthy |
| Pages outside business hours | < 1/week | 3.2/week | UNHEALTHY |
| False positive rate | < 10% | 22% | UNHEALTHY |
| Mean time to acknowledge | < 5 min | 3.2 min | Healthy |
| Mean time to resolve | < 30 min | 45 min | At risk |
| Unique alerts (not deduped) | trending down | flat | Needs attention |

## Action Plan:
1. Audit all alerts that paged outside business hours -- suppress non-urgent
2. Review false positives: tune thresholds on disk utilization and GC alerts
3. Consolidate 4 overlapping latency alerts into 1 SLO burn rate alert
```

**Fatigue Reduction Strategies:**

```markdown
## Pager Fatigue Reduction Checklist

**Alert Hygiene:**
- [ ] Every alert has fired in the last 90 days (delete stale alerts)
- [ ] Every alert that paged resulted in a human action (not just "acknowledge and ignore")
- [ ] No two alerts fire for the same underlying issue (deduplicate)
- [ ] Warning-level alerts route to Slack, not pager
- [ ] Info-level alerts are dashboards, not notifications

**Structural Improvements:**
- [ ] Follow-the-sun rotation for globally distributed teams
- [ ] Maximum on-call shift length: 7 days (prefer shorter)
- [ ] Mandatory decompression time after heavy on-call weeks
- [ ] New alerts require runbook and peer review before activation
- [ ] Quarterly alert review: delete, tune, or document every alert
```

### 8.5 Handoff Procedures

```markdown
## On-Call Handoff Template

**Outgoing:** [Name] | **Incoming:** [Name]
**Rotation:** Payments Platform | **Week of:** 2026-04-06

### Active Issues
- [ ] INC-2026-0402: Intermittent latency spikes from payments-db during 
      peak hours. Monitoring; no action needed unless p99 > 800ms.
      Dashboard: [link]

### Resolved This Week
- INC-2026-0401: Certificate expiry on api.payments.internal -- renewed, 
  automated renewal ticket OPS-1240 in progress

### Upcoming Risks
- **April 8:** Marketing campaign launch, expected 2x traffic spike. 
  Pre-scaled to 12 pods (normal: 8). Auto-scaling tested.
- **April 10:** Database maintenance window 02:00-04:00 UTC. 
  Failover tested. Runbook: [link]

### Alert Changes
- NEW: Added `PaymentsDBReplicaLag` alert (> 30s lag triggers warning)
- TUNED: `PaymentsAPILatencyHigh` threshold raised from p99 > 300ms 
  to p99 > 500ms after false positive analysis
- SILENCED: `DiskUsageHigh` on metrics-server until storage migration 
  completes (April 12). Ticket: OPS-1239

### Notes for Incoming
- The new k8s node pool in us-west-2 is still warming up. 
  If you see scheduling delays, check node readiness first.
- Stripe webhook endpoint was flaky on Tuesday. It self-resolved 
  but worth watching.

**Handoff confirmed:** [ ] Outgoing | [ ] Incoming
```

### Anti-Patterns

- **No handoff notes:** The incoming on-call starts blind with no context on active issues
- **Permanent on-call:** One person always on-call because "they know the system best"
- **No shadowing:** New engineers go on-call primary with zero preparation
- **Runbook-less alerts:** Alert pages at 3 AM with no documentation on what to do
- **Ignoring pager health:** Accepting 10+ pages per shift as normal degrades the team
- **No compensation:** Expecting engineers to carry pager burden without recognition or comp time
- **Skipping handoff for "quiet" weeks:** Even quiet weeks have context worth passing along

---

## Output Format

When applying this skill, structure deliverables as follows:

```markdown
## SRE Assessment: [Service Name]

### Current State
- SLOs defined: [yes/no, with details]
- Error budget policy: [exists/missing]
- Monitoring coverage: [USE/RED/gaps]
- On-call health: [pages/week, false positive rate]
- Toil ratio: [estimated percentage]
- Incident process: [maturity level 1-5]
- Postmortem culture: [maturity level 1-5]

### Recommendations (prioritized)
1. [Highest impact action]
2. [Second priority]
3. ...

### Implementation Plan
- **Week 1-2:** [Quick wins]
- **Month 1:** [Foundation work]
- **Quarter 1:** [Full implementation]
```
