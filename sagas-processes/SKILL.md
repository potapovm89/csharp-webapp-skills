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

## Non-Negotiable Invariants (both patterns)

1. **Every step is idempotent** — it may run again after a crash; deduplicate by `(process_id, step)` in the inbox or a unique key (`message-brokers`). An external call inside a step carries a provider idempotency key derived from the same `(process_id, step)`: local dedup cannot prevent a second charge after a crash between the call and the commit (`api-clients`).
2. **State transition + business effect + outbox row = one transaction.** A step that changed data but lost its "next step" message breaks the process; the reverse loses data. Never publish a "step done" event outside the transaction.
3. **Compensations are explicit and ordered.** List per step what undoes it; only completed steps get compensated; a compensation runs through the same outbox/inbox discipline and is itself idempotent.
4. **Non-compensable side effects go last** (or get a human-decision state): money sent to a provider, a message to an external system — "DB rollback does not undo a payment already sent" (`data-recovery`). The process must surface these as explicit states, not silently undo them.
5. **Stuck processes are visible**: every non-terminal process has `next_wake_at_utc` (NULL once terminal), the runner claims due ones with a lease, and the gauge alerts on overdue ones; a Failed state requires a runbook, not a log line. "Now" comes from `TimeProvider`, never from `now()` in SQL.
6. **State machine is explicit**: an enum of states and allowed transitions in one place; no ad-hoc booleans describing process progress.

## Where Things Live
- Process state and steps are business logic: step Stories and the claim Story in `Stories` (`cqrs-stories`), persistence through domain services in `DataAccess` (`postgresql-data-access`), side effects to other services through the outbox. The WorkerHost runner job and broker processors only dispatch Stories (`worker-host-jobs`, `message-brokers`); they never call external systems or domain services themselves.
- Process definitions (states, steps, compensation per step) are documented in `docs/adr/` (one ADR per non-trivial process) and diagrammed in `docs/arch/` (`project-documentation`).
- Tables (`process_instances`, `process_events`) are created by migrations with the standard rules (`database-migrations`).

## Subtopic References
- [process-manager.md](./references/process-manager.md) — the default: schema, runner, step handlers, timeouts, compensation, metrics. **Read it before implementing a process.**
- [choreography.md](./references/choreography.md) — event-driven chains without central state, and when to promote them to a process manager.
