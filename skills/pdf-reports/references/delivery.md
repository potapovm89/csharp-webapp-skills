# Delivery (Storage, Statuses, Access, Retention)

## Report Entity

```dbml
Table generated_reports {
  id bigint [pk, increment]
  report_type int [not null]              // enum, XML-documented
  entity_id bigint [not null]             // aggregate the report is about
  status int [not null]                   // Pending=1, Generating=2, Ready=3, Failed=4, CleanupPending=5
  storage_key varchar(500) [null]         // Ready object key or retained cleanup target
  error text [null]                       // client-safe failure reason
  expires_at_utc timestamptz [null]       // retention deadline per report type
  is_deleted boolean [not null, default: false]
  created_by varchar(100) [not null]      // IKnowCreator: requesting user id from claims
  created_date timestamptz [not null]
  updated_by varchar(100) [null]          // IKnowUpdater: user or the system job identity
  updated_date timestamptz [null]
}
```

- Timestamps come from one source per entity (`domain-modeling`, entity.md): the report has an author, so it uses the audit mixins `IKnowCreator`/`IKnowUpdater` and never the `BaseEntity` columns.
- Status transitions are checked in the domain service (an invalid transition throws). Optimistic concurrency uses the system column `xmin` mapped on the entity (`postgresql-data-access`, concurrency.md) — not a declared column, so it is absent from the DBML. The `Pending → Generating` update is the claim: a concurrency conflict means another worker took the report.
- The row records who requested the report and its current availability. `Ready` is committed only after upload. Cleanup must leave `Ready` before touching the object. A missing object while the row is `Ready` is an inconsistency to investigate (`data-recovery`, consistency between stores).
- Schema changes go through migrations and are mirrored in `docs/database/` (`database-migrations`, `project-documentation`).

## Storage Keys and Write Order
- Use a stable prefix from the report identity and an immutable key per upload, for example `reports/<report-type>/<entity-id>/<report-id>/<claim-id>/<upload-id>.pdf`. Use unique upload IDs or enforce create-only writes if an attempt can be dispatched twice. Never upload a new attempt to the key of a Ready report.
- Carry the claim ID from the claim transaction through rendering and upload. After upload, conditionally change the same `Generating` claim to `Ready` and persist its key in one transaction. Check claim ownership and the row concurrency token. A stale worker cannot make its object visible or overwrite the selected object's bytes.
- Track unselected keys separately from the Ready key. Clean them after their claim is retired and cannot publish the key. Keep an attempt ledger or use a bounded re-scan of the report prefix with a retention policy for late uploads. Before deleting, check that the exact key is not selected and that a publish with it can no longer succeed. If the DB commit result is unknown, verify the recorded winner before deleting anything.

## Access
- The document is private. Before issuing a presigned link, the Story checks ownership of `entity_id` from claims, `status == Ready`, `is_deleted == false` and `expires_at_utc > now` using `TimeProvider`. The signed lifetime must fit within both `LinkLiveTime` and the remaining retention window. If the signer only supports a fixed lifetime that exceeds this window, withhold the link until a shorter lifetime is supported. The DB state gates new issuance. Previously issued links are subject to storage authorization and their signed expiry. [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- Proxying the object from S3 through a controller is prohibited unless the requirement explicitly demands ephemeral access without a shareable link — then the ownership check still happens in the Story and the decision is documented in the controller contract (`web-host-api`).

## Status Flow and Errors
- `Pending → Generating → Ready | Failed` (statuses as int per `postgresql-data-access`). A crash during generation leaves `Generating`; the scan job (WorkerHost, `worker-host-jobs`) dispatches a Story that moves `Generating` rows older than the threshold (from `TimeProvider`, never `now()` in SQL) back to `Pending` for a bounded number of attempts and then to `Failed`, so a user is never stuck on "generating forever".
- `error` carries a client-safe reason only; stack traces and payloads stay in logs (`observability`, no sensitive leaks).
- `CleanupPending` is an internal state. Use the existing expired or unavailable response for the public API.
- After the last attempt the report stays `Failed` until it is eligible for cleanup. The user starts a new request. The runbook covers mass failures such as missing fonts/ICU or S3 availability.

## Retention and Cleanup
- Every `report_type` declares its retention in `docs/` (152-FZ/GDPR — `security`): e.g. invoices follow accounting retention, ad-hoc user exports expire after N days. `expires_at_utc` is set when the report becomes `Ready`.
- The cleanup job is a documented, authorized mechanism that dispatches a Story. The Story uses the following order:
  1. In a DB transaction, claim an expired `Ready` or eligible `Failed` row by transitioning it to `CleanupPending`. Retain the selected `storage_key` as the cleanup target. If no Ready key was committed, use the separate attempt cleanup path. A report identity identifies the prefix, not a winning object. Keep `is_deleted = false` so a later job can select the row. A row that has already reached `CleanupPending` resumes at the next step.
  2. After the transaction commits, delete the target from S3. An S3 timeout or access denial keeps the row pending. Successful deletion or confirmed absence satisfies this step. If versioning is enabled, use the policy for the report's object versions. In Amazon S3, a delete marker alone does not purge earlier versions. [Amazon S3 DeleteObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html)
  3. In a new DB transaction, finalize the same `CleanupPending` row with `is_deleted = true`. A crash before finalization leaves a discoverable row and the same cleanup target. On retry, complete the row after deletion or confirmed absence. Finalized rows are excluded from the scan.
- Storage growth is monitored with the S3 metrics of the platform; a report type whose volume dominates the bucket is a design signal (archive to a colder class or shorten retention) — `data-recovery` owns the restore perspective.
- If the report must reach a user by email, that flow belongs to the notifications scope — do not wire SMTP into the report pipeline here.
