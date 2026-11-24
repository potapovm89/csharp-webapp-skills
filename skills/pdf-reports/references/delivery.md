# Delivery (Storage, Statuses, Access, Retention)

## Report Entity

```dbml
Table generated_reports {
  id bigint [pk, increment]
  report_type int [not null]              // enum, XML-documented
  entity_id bigint [not null]             // aggregate the report is about
  status int [not null]                   // Pending=1, Generating=2, Ready=3, Failed=4
  storage_key varchar(500) [null]         // set together with Ready
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
- The row is the source of truth: "was this document ever issued, to whom, what state is it in". A `Ready` row without a stored object is a bug the drill/test must catch (`data-recovery`, consistency between stores).
- Schema changes go through migrations and are mirrored in `docs/database/` (`database-migrations`, `project-documentation`).

## Storage Keys and Write Order
- Deterministic key per the `s3-storage` rule: `reports/<report-type>/<entity-id>/<report-id>.pdf` — retrievable without a file index, and safe to re-derive in recovery.
- Order: render → upload to the key → one transaction that sets `Ready` + `storage_key`. S3 and PostgreSQL never share a transaction; uploading first guarantees that `Ready` implies "object exists". A crash after the upload leaves the row in `Generating`; the retry overwrites the same key.
- An object whose row is `Failed` or deleted is removed by the cleanup job (below) — never by the request path.

## Access
- The document is private. The only access path is `GetFileLink` (presigned, `LinkLiveTime`) issued by a Story that checks ownership of `entity_id` from claims (`security`) — possession of a link is not authorization, the link is short-lived.
- Proxying the object from S3 through a controller is prohibited unless the requirement explicitly demands ephemeral access without a shareable link — then the ownership check still happens in the Story and the decision is documented in the controller contract (`web-host-api`).

## Status Flow and Errors
- `Pending → Generating → Ready | Failed` (statuses as int per `postgresql-data-access`). A crash during generation leaves `Generating`; the scan job (WorkerHost, `worker-host-jobs`) dispatches a Story that moves `Generating` rows older than the threshold (from `TimeProvider`, never `now()` in SQL) back to `Pending` for a bounded number of attempts and then to `Failed`, so a user is never stuck on "generating forever".
- `error` carries a client-safe reason only; stack traces and payloads stay in logs (`observability`, no sensitive leaks).
- After the last attempt the report stays `Failed` and the user starts a new request; the runbook covers mass failures — usually fonts/ICU or S3 availability.

## Retention and Cleanup
- Every `report_type` declares its retention in `docs/` (152-FZ/GDPR — `security`): e.g. invoices follow accounting retention, ad-hoc user exports expire after N days. `expires_at_utc` is set when the report becomes `Ready`.
- A cleanup job dispatches a Story that deletes expired and failed objects from S3 first, then marks the rows deleted; a crash between the two leaves a row whose object is gone, which the next run finishes. The job is a documented, reviewed mechanism with the same authorization standing as any destructive operation, not an ad-hoc script (`postgresql-data-access` soft-delete rules).
- Storage growth is monitored with the S3 metrics of the platform; a report type whose volume dominates the bucket is a design signal (archive to a colder class or shorten retention) — `data-recovery` owns the restore perspective.
- If the report must reach a user by email, that flow belongs to the notifications scope — do not wire SMTP into the report pipeline here.
