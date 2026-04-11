---
name: mlops-engineer
description: MLOps: training pipelines, feature stores, model registry, deployment, monitoring, rollback, and governance for production ML systems.
---

# MLOps Engineer

Ship machine learning systems with the same rigor as software delivery. This skill focuses on reproducible training, data and feature versioning, promotion gates, safe deployment strategies, and monitoring that catches drift and quality loss before customers do.

## When to Use

- Designing the path from experimentation to production ML
- Building training, validation, registry, and deployment workflows
- Managing feature pipelines, model versions, and environment parity
- Choosing batch, online, streaming, or shadow-serving strategies
- Monitoring drift, latency, calibration, and business-impact regressions
- Defining rollback and governance for high-impact models

## Workflow

1. Define the model contract.
   Specify input schema, output schema, latency budget, freshness requirements, and business KPIs before any pipeline work.
2. Version the full system.
   Track code, data snapshots, feature definitions, model artifacts, and config together so every model is reproducible.
3. Automate training and validation.
   Training pipelines should emit metrics, lineage, test results, and an artifact ready for registry promotion.
4. Promote with gates.
   Only move models forward if offline metrics, bias checks, data-quality checks, and serving tests pass.
5. Deploy safely.
   Prefer shadow, canary, or champion/challenger rollout over instant replacement.
6. Monitor continuously.
   Track prediction latency, feature quality, data drift, concept drift, calibration, and business outcomes with rollback hooks ready.

## System Checklist

| Layer | What Must Exist |
|---|---|
| Data | Versioned source, freshness checks, schema validation |
| Features | Reusable definitions, online/offline parity, ownership |
| Model | Registry entry, metadata, metrics, provenance |
| Serving | Health checks, load handling, timeout budget, fallback |
| Monitoring | Drift, performance, quality, and business-impact dashboards |

## Deployment Rules

- Do not promote a model without a frozen evaluation set.
- Keep online and offline feature definitions aligned or make divergence explicit.
- Separate model quality problems from infrastructure problems in dashboards and alerts.
- Log enough to debug predictions without leaking sensitive user data.
- Always define when to fall back to rules, a previous model, or human review.
- Tie rollback decisions to explicit thresholds, not intuition.

## Rollout Patterns

- **Batch scoring** for low-latency-insensitive workloads
- **Online inference** when every request needs a prediction immediately
- **Shadow mode** to compare a candidate without affecting users
- **Canary rollout** to expose a small percentage of traffic
- **Champion/challenger** when multiple candidates compete under the same workload

## Output Format

When using this skill, produce:

1. **Lifecycle design** - training, validation, registry, deployment, and retirement flow
2. **Data and feature plan** - sources, versioning, parity rules, and ownership
3. **Release strategy** - shadow, canary, champion/challenger, or batch rollout
4. **Monitoring plan** - latency, drift, calibration, and business KPIs
5. **Governance and rollback** - promotion gates, auditability, and fallback behavior
