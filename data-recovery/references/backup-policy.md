# Backup Policy & Recovery Scope

## Recovery Contract (`docs/recovery/plan.md`)
For each failure scenario (accidental deletion, corrupt update, lost cluster, stolen credentials), record:

| Field | Required decision |
|---|---|
| Business scope | Critical operations, authoritative state and acknowledged-data boundary |
| Objectives | Maximum RPO/RTO; incident/reference timestamp and start/end timing rules |
| Method | Backup/PITR/replica/reconstruction approach and version-compatible tools |
| Copies | Locations, separate failure/credential boundary, encryption and key recovery |
| Coverage | DBs, globals/roles, extensions, objects/versions, broker data, auth/configuration, keys |
| Retention | Required recovery window, archive coverage, expiry and deletion/legal-hold controls |
| Operations | Backup/restore owners, monitoring/alerts, drill cadence, escalation and costs |
| Evidence | Last successful restore, measured loss/time and limitations |

Objectives must cover the **whole business service**, not only time to start a database. For example, meeting a DB recovery time while spending hours reconciling payments can still miss service RTO. Numeric examples in other skills are not automatic recovery targets. Agree targets/cadence before a release that depends on new persistent state; missing target values are not permission to assume they passed.

## PostgreSQL
- Logical dumps suit selected database/object recovery and some migration cases. PITR requires a compatible physical base backup plus an uninterrupted WAL archive covering the chosen recovery point; `pg_dump` is not a PITR base backup. Physical recovery scope is the cluster, not an arbitrary single table.
- Record server major version, extensions, tablespaces, roles/privileges and required configuration. Plan the exact compatible restore tooling; an upgrade or extension change may need a different recovery path and a fresh drill.
- Monitor usable recovery coverage and WAL archive continuity/lag, backup completion/age, capacity and key availability. “Last backup job succeeded” alone cannot establish RPO. Check a backup manifest/checksums where supported and perform restore drills; checksums alone do not establish business correctness.
- Validate that the requested PITR target is reachable and choose a target before the corrupt operation. Use the platform's supported restoration procedure and preserve the original source. Do not publish generic `rm -rf`/drop/overwrite commands as a runbook without verified target checks and authorization.
- Retention must retain the base backup and every needed archive segment; deleting an older base or key can invalidate an apparently recent recovery window. A replica can copy corruption/deletion and is not a separately protected backup.

Source: [PostgreSQL continuous archiving and PITR](https://www.postgresql.org/docs/18/continuous-archiving.html), reviewed 2026-11-14. Check the documentation for the deployed major version; use existing managed-service/backup tooling instead of inventing a universal restore command.

## Other Persistent State
- **S3:** record object keys/version ids, metadata, lifecycle/versioning, encryption keys and separate backup/replication retention. Check delete markers and whether referenced versions remain restorable. Versioning in the same compromised account is not an independent recovery copy. Reconcile DB object references against restored versions/checksums; test missing/orphaned objects.
- **MongoDB:** choose a supported consistent backup/point-in-time mechanism for the actual topology. Independent collection exports may not represent a consistent cross-collection state. Validate indexes, schema rules, roles and app behavior as well as document counts.
- **Kafka/RabbitMQ:** topology/definitions export is not backup of message contents. Record topic/queue retention, DLQ contents, consumer groups/offsets and replay sources. If a required history can expire before recovery completes, extend durable capture or change the recovery design.
- **Outbox/inbox/idempotency:** retain journals at least through the permitted replay/recovery horizon or use a durable business-key guarantee. Restoring DB journals to an earlier time can forget an already executed external effect; reconcile before replaying it. Do not promise exactly-once effects after restore from broker producer idempotence alone.
- **Redis/HybridCache:** normally reconstruct from authoritative stores. Clear/version isolated caches after restore and control warming/fallback load. If a feature relies on Redis as its only durable truth, report the conflict with `nosql-and-cache` and resolve the design before declaring recoverability.
- **Identity/configuration:** recover Keycloak realm/client configuration and needed keys using protected mechanisms. Configuration/IaC in git is useful but not a backup of live state. Restore/rotate credentials deliberately and prevent restored identities from accessing production during drills.

## Protection and Monitoring
Apply the original data's classification, retention and approved storage location to backups and drill copies. Separate backup administration from application credentials; the app's compromised account must not be able to destroy every recovery copy. Where required by the threat model, use offline/immutable copies with documented retention controls. Test access to decryption keys through an independent recovery path; encryption without recoverable keys is data loss.

Alert on lost recovery coverage, overdue backup/restore drills, archive failures, inaccessible keys and exhausted storage. Each alert has an owner and runbook. Operational dashboards should expose sanitized status/timestamps, never connection strings or recovered payloads.
