# Concurrency, Locking & Idempotency Reference

## Optimistic Concurrency (default for updates)
Lost updates happen when two requests read the same row and both write. Every entity that is updated after creation has a concurrency token.

EF Core + PostgreSQL — map the system column `xmin`, no schema change needed:

```csharp
/// <summary>
/// Версия строки (системная колонка PostgreSQL xmin).
/// </summary>
[Timestamp]
public uint Version { get; set; }
```

- Npgsql maps `[Timestamp] uint` to `xmin`. The generated migration lists the column, but the generated SQL does not create it — keep it in the model snapshot. Verify with `dotnet ef migrations script`.
- On conflict EF throws `DbUpdateConcurrencyException`. Handle it explicitly: API → 409 Conflict with ProblemDetails (the client reloads and retries); background job → skip the row and process it on the next run.
- Do not "fix" conflicts with `ChangeTracker.Clear()` and a blind retry of the write. Retry the **whole** use-case (read → decide → write) or report the conflict.
- Dapper: `UPDATE ... SET ..., version = version + 1 WHERE id = @Id AND version = @Version`; 0 rows affected means a conflict.

## Pessimistic Locking (only when needed)
- `SELECT ... FOR UPDATE` inside a transaction when a decision depends on the current value and conflicts are frequent (balances, counters).
- `FOR UPDATE SKIP LOCKED` to distribute rows between workers (outbox relay, job batches).
- Keep locking transactions short; never call external services while holding a row lock.

## Uniqueness Instead of "Check Then Insert"
- "Does it exist? → insert" races under concurrency. Enforce uniqueness with a unique index. If translating a known constraint violation into a domain error (400/409), check both `SqlState == "23505"` and the expected `ConstraintName`. Roll back the failed transaction and propagate unexpected violations.
- Keep the pre-check only for a friendly error in the common case; the unique index is the guarantee.

## Distributed Locks
- PostgreSQL advisory locks (`pg_try_advisory_lock` on a dedicated connection, or `pg_try_advisory_xact_lock` inside a transaction) for "run once across replicas" (see `worker-host-jobs`, clustering.md).
- Redis locks only with a TTL and a fencing check; prefer the database you already write to.

## Idempotency-Key for REST
For `POST` endpoints without a natural business key, scope the key to the authenticated owner and bind it to the complete operation using a canonical request fingerprint:

```sql
CREATE TABLE idempotency_keys (
    owner_id       varchar(100) NOT NULL,   -- store / user from claims
    key            varchar(100) NOT NULL,
    request_hash   char(64)     NOT NULL,   -- SHA-256 of the canonical operation, resource and inputs
    status_code    int          NULL,
    response_body  jsonb        NULL,
    created_date   timestamptz  NOT NULL DEFAULT now(),
    PRIMARY KEY (owner_id, key)
);
```

1. Authenticate and authorize the caller before looking up or replaying a cached result. Hash a structured canonical tuple of HTTP method, stable operation identifier (include API version when semantics differ), validated target resource type and identifier, applicable effect-changing route/query inputs, and canonical request body. An empty body still has an operation and may have a target resource.
2. In the use-case transaction, reserve the owner's key using `INSERT ... ON CONFLICT (owner_id, key) DO NOTHING`. If inserted, execute the use case and store the status and response in the same transaction.
3. If the key exists, check the entire fingerprint before using its result. Matching completed request → return the stored status and body. Matching request with a visible in-progress record → return 409. Different fingerprint → return 422 without replaying the response or executing the mutation. A concurrent reservation must have one winner.
4. Expire keys per the documented retry window. When migrating existing body-only hashes, verify the original operation and target before replaying any unexpired record. If the context cannot be recovered, use the documented ambiguous-key response and withhold both replay and re-execution. Preserve the guarantee during rolling deployment.

**Regression test:** for one owner, send `POST /orders/A/cancel` with key `K` and an empty body. Repeat it and assert the original response and one cancellation of A. Send `POST /orders/B/cancel` with the same key and empty body. Assert 422, no replay of A's response, and B remains active. With a fresh key, B can be cancelled.
