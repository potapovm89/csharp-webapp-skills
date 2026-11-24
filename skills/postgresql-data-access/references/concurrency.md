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
- "Does it exist? → insert" races under concurrency. Enforce uniqueness with a (filtered) unique index and translate `PostgresException { SqlState: "23505" }` into a domain error (400/409).
- Keep the pre-check only for a friendly error in the common case; the unique index is the guarantee.

## Distributed Locks
- PostgreSQL advisory locks (`pg_try_advisory_lock` on a dedicated connection, or `pg_try_advisory_xact_lock` inside a transaction) for "run once across replicas" (see `worker-host-jobs`, clustering.md).
- Redis locks only with a TTL and a fencing check; prefer the database you already write to.

## Idempotency-Key for REST
For `POST` endpoints without a natural business key:

```sql
CREATE TABLE idempotency_keys (
    owner_id       varchar(100) NOT NULL,   -- store / user from claims
    key            varchar(100) NOT NULL,
    request_hash   char(64)     NOT NULL,   -- SHA-256 of the canonical request body
    status_code    int          NULL,
    response_body  jsonb        NULL,
    created_date   timestamptz  NOT NULL DEFAULT now(),
    PRIMARY KEY (owner_id, key)
);
```

1. In the use-case transaction: `INSERT ... ON CONFLICT DO NOTHING`.
2. Inserted → execute the use-case and store status and response in the same transaction.
3. Not inserted → same hash and stored response: return it; same hash, no response yet: 409 (in progress); different hash: 422.
4. Keys expire (cleanup job, e.g. after 24 h). The key is scoped by the owner from claims, never global.
