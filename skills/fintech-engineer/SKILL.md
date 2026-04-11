---
name: fintech-engineer
description: Fintech systems: ledgers, payments, idempotency, reconciliation, risk controls, auditability, and regulatory-aware architecture.
---

# Fintech Engineer

Design financial systems that stay correct under retries, partial failure, and audit scrutiny. This skill centers on immutable ledgers, idempotent money movement, reconciliation, operational controls, and security boundaries that fit real-world payment and financial-product risk.

## When to Use

- Building payments, wallets, payouts, lending, treasury, or account systems
- Designing ledgers, transaction flows, authorization, capture, and settlement paths
- Handling retries, webhooks, reversals, chargebacks, or reconciliation
- Defining auditability, access controls, and operator workflows for money movement
- Integrating payment processors, banks, KYC/KYB, or fraud systems
- Reviewing architecture for financial correctness and failure recovery

## Workflow

1. Map the money flow clearly.
   Identify source of funds, destination, holds, settlement timing, reversals, and operational actors before designing tables or APIs.
2. Start with the ledger.
   Record financial truth in immutable entries and derive balances from posted movements or rigorously controlled projections.
3. Make every movement idempotent.
   Any call that can retry must have stable business keys and duplicate-suppression rules.
4. Separate authorization, orchestration, and accounting.
   External processor state, internal workflow state, and ledger state must not collapse into one overloaded status field.
5. Build reconciliation and exceptions into the design.
   Assume webhooks arrive late, banks disagree, processors time out, and manual review is sometimes required.
6. Protect privileged actions.
   Money movement, account changes, and operator overrides need role separation, audit logs, and approval paths.

## Non-Negotiables

- Immutable ledger or equivalent append-only financial event log
- Idempotency keys for every externally retried write
- Reconciliation jobs that compare internal and external truth sources
- Explicit treatment of pending, posted, failed, reversed, and disputed states
- Full audit trail for operator actions and system decisions
- Strong boundaries for PCI, PII, and secrets handling

## Design Rules

- Never derive correctness from a single processor callback.
- Avoid floating-point math for currency.
- Do not overload "balance" without defining available, pending, reserved, and settled semantics.
- Treat processor webhooks as signals to reconcile, not as unquestioned truth.
- Keep operational runbooks for replay, reversal, dispute, and outage scenarios.
- Regulatory requirements vary by product and region; architecture must leave room for them even when legal guidance is external.

## Output Format

When using this skill, produce:

1. **Money-flow map** - states, actors, external dependencies, and failure points
2. **Ledger and data model** - entries, balances, idempotency keys, and reconciliation objects
3. **Control plan** - auth, approvals, audit logs, and exception handling
4. **Integration plan** - processor/bank/KYC boundaries, retries, and webhook handling
5. **Risk notes** - correctness, fraud, operational, and compliance-sensitive concerns
