---
name: data-recovery
description: >-
  Use this skill when defining backup/restore policies, RPO/RTO, PostgreSQL PITR, coordinated database/object-store/message recovery, recovery runbooks or isolated restore drills. Triggers (RU): резервное копирование, восстановление данных, backup, restore, PITR, RPO, RTO, аварийное восстановление, проверка бэкапа.
---

# Data Recovery

Specify and prove recoverability of persistent business state. Replication, soft delete, a successful backup job and a rollback image are not evidence of a recoverable service. Read [backup-policy.md](./references/backup-policy.md) before selecting a strategy and [recovery-drills.md](./references/recovery-drills.md) before planning or running a drill.

## When to Use (and When Not)
- **Use** when adding a new persistent store, changing how business data is stored or deleted, preparing a release that depends on new persistent state, or writing a recovery runbook/drill.
- **Skip** for features that add tables/fields to an already covered database — the existing plan applies; update it only if retention or criticality changes.

## Quick Checklist
1. List the stores and external effects the change adds (DB, S3, broker history, outbox/inbox, keys).
2. Record RPO/RTO and the owner in `docs/recovery/plan.md`; unknown values stay open decisions.
3. Describe backup coverage and the ordered restore + reconciliation steps.
4. Plan an isolated drill (synthetic data, external effects disabled) and its pass criteria.
5. Never touch production: prepare the runbook for the authorized operator.

## Workflow
1. Inventory authoritative stores and external effects: PostgreSQL, MongoDB, S3, broker retention/offsets, outbox/inbox/idempotency journals, Keycloak/configuration and required keys. Mark reconstructible caches separately. Identify the recovery owner and dependencies; do not add a database/backup product merely because it is listed here.
2. Record business-approved loss/time limits in `docs/recovery/plan.md`: **RPO** is the tolerated loss of acknowledged business data; **RTO** is the tolerated elapsed time to validated service restoration. Define the incident scenario, timing boundaries, recovery scope, retention, drill frequency and owners. Unknown targets remain explicit open decisions; do not invent zero-loss claims.
3. Choose backup, archive and isolation controls that can meet those targets at realistic volume. Include encryption keys, access, copies outside the source's failure/credential boundary, archive-gap detection and expiry rules. Respect `security` data classification/location and `configuration-secrets`; no backup contents or secrets in git.
4. Write an ordered recovery/reconciliation runbook. Restore to a new isolated target, verify integrity and measure recovered business progress before any traffic cutover. Coordinate rollback/replay with `message-brokers` and `database-migrations`; DB rollback does not undo a payment already sent to an external provider.
5. In an authorized non-production drill, measure actual loss/time, validate application behavior and produce a sanitized report under `docs/recovery/drills/` or an access-controlled evidence link. A missed objective creates remediation and must not be reported as a passed recovery check.

## Execution Boundaries
- `AGENTS.md` prohibits this agent from connecting to production. These instructions authorize no production backup read, restore, deletion, replay or cutover; prepare the concrete runbook for the authorized operator.
- Before a non-production drill, resolve and verify the target host/cluster/bucket/database and its isolation. If a target, data permission or cleanup scope is ambiguous, stop dependent execution and clarify; do not guess a connection string.
- Use synthetic data or approved masked copies. Disable production egress, jobs, consumers, webhooks, email and payment dispatch before restored services start. Restoring credentials from a backup must not grant access to original services.
- Set time/storage limits, abort criteria and a cleanup owner. Preserve the source and recovery evidence; never “test restore” by overwriting the original dataset.

## Completion Evidence
- [ ] All required stores, keys and external effects have a recovery/reconciliation path and responsible owner.
- [ ] RPO/RTO, monitoring and drill cadence are defined by criticality, not by generic defaults.
- [ ] A drill verifies schema/records and business invariants at realistic size, including lost/duplicate messages and side effects where relevant.
- [ ] Measured results, gaps and a safe cutover/abort procedure are documented; no production operation is implied by preparing them.

Technical reference, reviewed 2026-11-14: [PostgreSQL backup and restore](https://www.postgresql.org/docs/18/backup.html). Use the actual server/storage versions and existing platform tooling when implementing the plan.
