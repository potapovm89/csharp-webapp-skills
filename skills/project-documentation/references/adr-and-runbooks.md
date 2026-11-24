# ADR & Runbooks Reference

## Architecture Decision Records (`docs/adr/`)
Write an ADR when a decision is hard to reverse or will be questioned later: a new library or service, a storage choice, a delivery guarantee, a clustering approach, an exception to a skill rule.

File name: `NNNN-<kebab-case-title>.md` (sequential number, never reused). Template:

```markdown
# NNNN. <Decision title>

- Status: proposed | accepted | superseded by NNNN | deprecated
- Date: YYYY-MM-DD
- Task: <task id>

## Context
What problem we solve, constraints (load, licenses, team, deadlines).

## Options
1. <Option> — pros / cons.
2. <Option> — pros / cons.

## Decision
What we chose and why.

## Consequences
What becomes easier, what becomes harder, what we must monitor, what follow-up work is needed.
```

- ADRs are immutable after acceptance; a changed decision gets a new ADR that supersedes the old one.
- Link the ADR from the code or doc where the decision is visible (e.g. the WorkerHost clustering choice).

## Runbooks (`docs/runbooks/`)
A runbook is written for the on-call engineer at 3 a.m.: symptoms → checks → actions, with copy-paste commands and no theory.

Required runbooks for a service:
- Every alert defined in `observability` (one runbook per alert, linked from the alert annotation).
- DLQ replay (below).
- Outbox backlog growth.
- Failed or stuck database migration.
- Rotation of secrets/certificates used by the service.
- Release verification/hold/rollback where CI/CD is configured (`delivery-and-supply-chain`): verified digest/config, observation and stopping conditions.
- Restore/PITR and cross-store reconciliation for persistent state (`data-recovery`), with isolated drill/abort instructions and operator-only production steps clearly distinguished.
- Customer-impacting SLO burn or required operational recovery signals (`reliability-and-performance`), with dashboard, owner and safe mitigation.

Reference approved objectives from `docs/reliability/` and `docs/recovery/plan.md`; do not duplicate target values independently in each runbook. Store sensitive release/restore logs outside git and link authorized evidence.

Template:

```markdown
# <Alert or situation>

## Symptoms
Which alert fires / what users see.

## Impact
What is broken, what still works, data-loss risk.

## Diagnosis
1. Check <dashboard/metric/log query>.
2. ...

## Actions
1. <Command or step>, expected result.
2. Escalate to <team> if <condition>.

## After the incident
What to record, what to fix permanently.
```

## DLQ Replay Runbook (template)
1. **Find the cause first.** Read the DLQ messages' error headers and the consumer logs by trace id. Replaying before the cause is fixed only refills the DLQ.
2. Deploy the fix (or confirm the dependency is back).
3. Dry run: start the replay tool with `--dry-run` and the available filters (`--message-id`, `--max`) and review the list.
4. Replay in small batches; watch the consumer error rate and the DLQ depth metric.
5. Messages that must never be processed (invalid by business rules) are archived with a reason, not replayed.
6. Record the count of replayed and archived messages in the incident ticket.

Replay is safe only because consumers are idempotent (inbox / unique keys); never replay into a consumer that is not.
