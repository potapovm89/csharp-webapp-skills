# Isolated Recovery Drills

## Preconditions
- Use a named non-production target and approved source dataset; resolve endpoints and account/cluster/bucket identifiers before execution. Network isolation and disabled dispatch are established **before** starting any restored host, worker or consumer.
- Agree scope, operator, time/storage bounds, abort conditions and cleanup owner. Use synthetic data by default; sensitive copies require explicit data-access permission and masking. Never fetch production backups under this skill's authority.
- Record the source application/schema/config version, backup/restore tool versions, chosen recovery point, archive coverage and hashes/ids of backup inputs. Preserve originals for diagnosis and avoid mutating the only copy.

## Drill Sequence
1. Create known business markers/transactions in the test source with durable sequence ids and recorded acknowledgment times. Include data with DB/S3 relationships and a message/external-effect case where those exist. Use a test sink for every external effect.
2. Take the planned backup/archive and induce the chosen failure in the disposable source only. Record the agreed reference time and recovery start. The scenario should expose a backup gap, corrupt write or lost instance rather than merely copying a healthy empty database.
3. Restore into the isolated target with compatible tools; keep writers, schedulers, consumers and outbox relay stopped. Inspect restore/archive errors and stop if the target point is unreachable. A partial restore is not a success to work around by skipping checks.
4. Validate restored schema/migration history, extensions, permissions, indexes, constraints and representative data. Check key relationships, row/document/object checksums or counts where meaningful, and business invariants such as order totals/balances and tenant isolation.
5. Reconcile all state boundaries using [backup-policy.md](./backup-policy.md): DB references to S3 versions, producer/consumer progress, pending/published outbox records, inbox and idempotency keys. Compare with the durable external test ledger. A previously emitted payment or email is not “un-sent” by rewinding a DB.
6. Establish the replay/cutover plan: explicit offsets/start points, duplicate handling, ordering and throttling. Re-enable one isolated consumer/relay path at a time, verify retries do not repeat external effects, then run the critical business smoke checks. Never blindly reset consumer offsets or replay every pending outbox row after a DB rewind.
7. Record when the agreed service acceptance criteria are met and measure recovery time/loss. A gap or duplicate outside the agreed objective fails the drill even if the database process is healthy.
8. Stop the drill at its limit or on wrong target, production egress, missing required key/archive, integrity failure or uncontrolled external effects. Preserve evidence, isolate the target and escalate. Cleanup of disposable targets follows its agreed scope and authorization; source data is never cleanup material.

## Measuring Outcomes
- **Measured RPO/loss:** compare the acknowledged business markers expected at the chosen reference point with those actually recovered/reconciled. Identify the earliest missing acknowledgment, latest contiguous recovered progress and any holes; do not infer loss solely from the newest row timestamp. Record missing ids/counts and the loss time span.
- **Measured RTO:** elapsed wall-clock time from the agreed incident/recovery trigger through restoration, reconciliation and verified service acceptance. If detection/decision time is excluded by the contract, report it separately; do not hide it by starting the clock after the slow work.
- Compare measured results to the approved objectives from `docs/recovery/plan.md`. Record actual data volume, environment capacity, archive distance and parallelism. A small fixture proves procedure behavior, not production-scale RTO.

## Drill Report (`docs/recovery/drills/<date>-<scenario>.md`)
Include scenario, authorized scope/operator, objective values and timing definitions, backup ids/coverage, source/target versions, expected/recovered marker range and gaps, measured loss/RTO, integrity/reconciliation checks, unexpected side effects, abort/cleanup outcome and remediation owners/dates. Store sensitive logs outside git and link access-controlled evidence. State clearly whether the exercise was a tabletop, a small fixture or a realistic-volume restore.

## Minimum Failure Cases
Exercise a missing archive segment/unreachable recovery point; unavailable decryption key; a restore preceding an already delivered external operation; missing S3 version; and a safe abort on an unintended target. Apply only the cases relevant to the system; make unsupported scenarios explicit. Repeat drills on the agreed schedule and after material changes to backup tooling, storage/schema, encryption or recovery topology.

The production runbook is a reviewed operator artifact. No instruction here lifts `AGENTS.md`'s prohibition on production connections, tests or restore/cutover actions by this agent.
