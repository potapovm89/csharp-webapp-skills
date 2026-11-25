---
name: sagas-processes
description: >-
  Use this skill when a business process spans multiple transactions, services or external calls and needs compensation: choosing between a local transaction, an orchestrated process (in-house process manager on PostgreSQL + outbox + WorkerHost timeouts) and choreography on broker events; implementing idempotent steps, compensation handlers, stuck-process detection and process tests. Triggers (RU): сага, процесс, компенсация, многошаговый, оркестрация, хореография, process manager.
---

# Sagas and Long-Running Processes

A saga exists because some steps cannot share one database transaction: an external payment, a broker side effect, a step that happens hours later. A saga does **not** make the process atomic — it makes the outcome *eventually consistent* with explicit compensations. If every step of the process can run in one `ITransactionScope` against one database, this skill does not apply: write one Story, not a process (`cqrs-stories`).

## Choosing the Pattern

| Situation | Pattern |
| --------- | ------- |
| All steps in one DB, one host, no external effects | One Story + `ITransactionScope` — no saga |
| Steps across time/external systems, need visibility, retries and compensations | **Process manager (orchestration)** — default in this solution |
| 2–3 participants, each owning its reaction, no central visibility needed | **Choreography** — events via outbox, each consumer compensates itself |

The default is the **in-house process manager** built on primitives this solution already runs: PostgreSQL state, `ITransactionScope`, the transactional outbox (`message-brokers`), and WorkerHost timeouts (`worker-host-jobs`). Introducing an external workflow engine (Temporal, Camunda, MassTransit sagas beyond `message-brokers` caps) is a licensing/infrastructure decision requiring an ADR and explicit user approval (`csharp-code-style`, libraries.md) — do not add one to escape modeling work.

**Scope example** — A delivery workflow calls a provider, records the outcome and then posts a webhook. Exercise a crash after the provider accepts the send and before the outcome commits. Select a saga when that process needs explicit compensation or coordination across steps.

## Model the Transitions

Read the reference for the selected pattern before implementing. For each step and compensation, fill in this record in the process ADR:

| Starting state and trigger | Owner and guard | Atomic local writes and outbox event | External effect and idempotency key | Deadline and retry limit | Next state on success, failure or unknown outcome | Compensation or manual action |
|---|---|---|---|---|---|---|
| One row per accepted transition | Name the owning Story or participant | Include the state change and deduplication record where applicable | State how a prior effect is recognized | Use the project time source | Name all outcomes explicitly | State the reverse order and when to stop |

### External Effect With an Unknown Outcome

For a provider without deduplication, persist an exclusive dispatch reservation before sending. Include the process and step identity and any provider correlation key. If the worker crashes before recording the result, the reservation sends recovery to reconciliation. A timeout, expired lease or exhausted retry budget alone leaves the outcome unknown.

| Reconciliation result | Transition |
|---|---|
| Effect confirmed | Record success and advance once, without another send. |
| Effect confirmed absent and the prior attempt can no longer complete | Retry if the policy and budget allow. Otherwise handle the confirmed failure and compensate completed steps if required. |
| Effect still unknown or impossible to query | Remain in reconciliation or escalate to the documented manual state. Hold forward dispatch, compensation and dependent terminal events until resolution. |

Within the retry budget, a provider with confirmed deduplication may be retried using the original key and payload. If it remains unresolved at exhaustion, take the reconciliation/manual path.

## Non-Negotiable Invariants (both patterns)

1. **Every step is idempotent.** Deduplicate by `(process_id, step)` in the inbox or with a unique key (`message-brokers`). When the provider supports idempotency, derive a stable provider key from the same step identity and reuse it on every attempt. If the outcome is unknown and the provider cannot deduplicate, reconcile the effect or enter the documented manual state before any replay (`api-clients`).
2. **State transition + local business writes + outbox row = one transaction.** Record the outbox event in that transaction and publish it through the relay. External effects have their own failure boundary and must use the idempotency or reconciliation path above.
3. **Compensations are explicit and ordered.** List per step what undoes it; only completed steps get compensated; a compensation runs through the same outbox/inbox discipline and is itself idempotent.
4. **Non-compensable side effects go last** (or get a human-decision state): money sent to a provider, a message to an external system — "DB rollback does not undo a payment already sent" (`data-recovery`). The process must surface these as explicit states, not silently undo them.
5. **Stuck processes are visible.** Automatic states schedule `next_wake_at_utc` and the runner claims due work with a lease. Terminal and operator-only states clear the wake time. Monitor overdue automatic work and unresolved manual states, and document the runbook and owner. Read the current time from `TimeProvider`.
6. **State machine is explicit**: an enum of states and allowed transitions in one place; no ad-hoc booleans describing process progress.

## Where Things Live
- Process state and steps are business logic: step Stories and the claim Story in `Stories` (`cqrs-stories`), persistence through domain services in `DataAccess` (`postgresql-data-access`), side effects to other services through the outbox. The WorkerHost runner job and broker processors only dispatch Stories (`worker-host-jobs`, `message-brokers`); they never call external systems or domain services themselves.
- Process definitions (states, steps, compensation per step) are documented in `docs/adr/` (one ADR per non-trivial process) and diagrammed in `docs/arch/` (`project-documentation`).
- Tables (`process_instances`, `process_events`) are created by migrations with the standard rules (`database-migrations`).

## Subtopic References
- [process-manager.md](./references/process-manager.md) — read for orchestration: schema, runner, timeouts, compensation and metrics.
- [choreography.md](./references/choreography.md) — read for event-driven chains: local ownership, compensation events and when to promote the chain to a process manager.

## Completion Evidence

- In an isolated process test, deliver the same forward trigger twice and assert a single local effect, state transition and outbox record. Include a stale or invalid trigger and assert that it leaves the state and effects unchanged.
- Advance the test clock and exhaust retries after a confirmed failure. With completed steps A and B and definitively failed step C, expect compensation B then A and none for C. Replaying compensation must produce no additional effect.
- With a provider fixture that has no deduplication, persist the reservation, record one external effect, then crash before the result commits. Expire the lease and reach the retry limit. Assert one provider send, zero compensations and no terminal event while the outcome is unknown. Then make reconciliation confirm the effect and assert one forward transition without a second send.
- For a provider that supports deduplication, assert that a permitted replay uses the original key and payload and produces one external effect. Record test commands and observed transitions and effects.
