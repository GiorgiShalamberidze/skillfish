---
name: database-administrator
description: Database administration: backups, replication, query tuning, maintenance windows, capacity planning, access control, and incident response.
---

# Database Administrator

Operate production databases with safety before cleverness. This skill emphasizes restore confidence, controlled change management, performance diagnosis from evidence, clear access boundaries, and capacity planning that prevents emergencies instead of reacting to them.

## When to Use

- Running or reviewing production database operations
- Planning backup, restore, replication, and maintenance strategies
- Diagnosing slow queries, lock contention, replication lag, or storage pressure
- Managing user access, rotation, auditing, and privileged workflows
- Preparing schema or infrastructure changes for critical databases
- Writing database runbooks and incident response plans

## Workflow

1. Classify the workload and business expectations.
   Identify OLTP vs analytics behavior, recovery objectives, peak periods, and data retention constraints before tuning anything.
2. Verify recovery before optimization.
   A backup policy is incomplete until restore drills prove RTO and RPO expectations.
3. Triage with evidence.
   Use query plans, wait events, lock graphs, disk usage, and connection metrics instead of changing indexes or parameters blindly.
4. Separate safe changes from risky changes.
   Tune queries, indexes, vacuum/analyze windows, or connection limits before considering topology or engine-level changes.
5. Control access and blast radius.
   Use least privilege, break-glass procedures, audited admin access, and explicit credential rotation ownership.
6. Document the operational contract.
   Capture runbooks for failover, restore, schema changes, vacuum/reindex, and on-call escalation.

## Triage Ladder

| Symptom | Check First | Common Root Cause |
|---|---|---|
| Slow endpoint | Query plan and wait events | Missing index, bad join order, row explosion |
| Rising CPU | Top queries and connection count | Inefficient query mix or connection storm |
| Replication lag | WAL/binlog volume and slow replica queries | Write spike, network saturation, replica contention |
| Disk pressure | Retention, temp files, table/index growth | Runaway logs, bloat, forgotten archives |
| Lock contention | Blocking query chain | Long transaction, hot-row contention, migration lock |

## Operational Rules

- Prefer additive schema changes and staged rollouts for live systems.
- Treat backups, replicas, and snapshots as different tools; one does not replace the others.
- Keep connection pools bounded. "More connections" often makes a busy database slower.
- Tune the highest-cost queries before global server settings.
- Review long-running transactions and idle-in-transaction sessions aggressively.
- Rehearse failover and restore while things are calm, not during the incident.

## Change Management Checklist

- Clear maintenance window or rollout plan
- Rollback and restore path
- Capacity impact estimate
- Replica and backup status confirmed
- Alert thresholds adjusted for known temporary noise
- Stakeholder communication for risky operations

## Output Format

When using this skill, produce:

1. **Workload assessment** - traffic shape, recovery targets, and operational constraints
2. **Risk summary** - backup, access, replication, performance, and capacity issues
3. **Action plan** - immediate fixes, scheduled maintenance, and structural improvements
4. **Verification steps** - metrics, restore checks, query evidence, and rollback validation
5. **Runbook updates** - exact procedures that should be documented or revised
