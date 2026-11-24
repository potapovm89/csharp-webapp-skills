# Zero-Downtime Migrations (PostgreSQL)

## Principle: Expand → Migrate → Contract
During a rolling deployment the old and the new version of the service run **at the same time** against the same schema. Therefore every release is split into backward-compatible steps:

1. **Expand** (migration of release N): add new columns/tables/indexes. The old code keeps working, because nothing it uses changed.
2. **Migrate** (code of release N, plus background backfill): the new code writes both old and new structures (or only new ones, if nothing still reads the old) and reads the new ones; existing rows are backfilled in batches.
3. **Contract** (migration of release N+1 or later): once no running version uses the old structure, drop it.

A single migration that renames or drops something still used by the previous release is **prohibited**.

## Operation Rules

| Change | Safe way |
| ------ | -------- |
| Add nullable column | Safe in one step. |
| Add column with default | PostgreSQL 11+ can store a non-volatile default as metadata without rewriting existing rows. `now()` is STABLE (transaction timestamp), not VOLATILE. Volatile defaults such as `clock_timestamp()` and `gen_random_uuid()` require per-row evaluation: add nullable, backfill, then set the default. Check the exact expression and server version. |
| Make column NOT NULL | 1) add `CHECK (col IS NOT NULL) NOT VALID`; 2) backfill; 3) `VALIDATE CONSTRAINT`; 4) `SET NOT NULL` (uses the validated check, no full scan); 5) drop the check. |
| Rename column / table | Never in place. Add the new column → write both → backfill → switch reads → stop writing the old → drop the old in a later release. |
| Change column type | Same as rename: new column of the new type, dual write, backfill, switch, drop. |
| Drop column | Release N: code stops using it (for EF also remove it from the model/snapshot via a migration that does **not** drop it, or keep the property ignored). Release N+1: migration drops it. |
| Drop table | Only after no released version references it; keep a backup or rename to `<name>_deprecated` first if rollback may be needed. |
| Create index | `CREATE INDEX CONCURRENTLY` outside a transaction (see below). Plain `CREATE INDEX` blocks writes for the whole build. |
| Unique constraint | `CREATE UNIQUE INDEX CONCURRENTLY`, then `ALTER TABLE ... ADD CONSTRAINT ... UNIQUE USING INDEX ...`. |
| Foreign key | `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` in a separate statement/migration. |
| New enum value (stored as int) | Safe; old code must tolerate unknown values (map to a default/unknown branch, never crash). |
| Large data backfill | Not in the migration. A batched job/script (e.g. 1 000 rows per transaction, `FOR UPDATE SKIP LOCKED` or keyset by id) that can be stopped and resumed. |

Sources for defaults: [PostgreSQL ALTER TABLE](https://www.postgresql.org/docs/18/ddl-alter.html) and [function volatility](https://www.postgresql.org/docs/18/xfunc-volatility.html).

## Locks and Timeouts
Every migration that touches an existing table starts with timeouts, so a blocked `ALTER` fails fast instead of queueing all traffic behind it:

```sql
SET lock_timeout = '5s';
SET statement_timeout = '15min';
```

Retry the deployment after a lock timeout; never raise `lock_timeout` to "make it pass".

## Non-Transactional Statements
`CREATE INDEX CONCURRENTLY`, `DROP INDEX CONCURRENTLY` and `VACUUM` cannot run inside a transaction.

EF Core:
```csharp
migrationBuilder.Sql(
    "CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_payments_created ON payments (create_date);",
    suppressTransaction: true);
```

FluentMigrator:
```csharp
[Migration(202611131600, TransactionBehavior.None, "TASK-123")]
public class Migration_202611131600 : Migration
{
    public override void Up() =>
        Execute.Sql("CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_store_orders_store_id ON store_orders (store_id);");

    public override void Down() =>
        Execute.Sql("DROP INDEX CONCURRENTLY IF EXISTS ix_store_orders_store_id;");
}
```

- Keep a non-transactional statement alone in its migration: if it fails halfway, a concurrent index stays `INVALID` and must be dropped before retrying (`IF NOT EXISTS` would otherwise skip the broken index — check `pg_index.indisvalid`).
- EF Core generates a plain `CreateIndex`; for existing large tables replace it with the `Sql(..., suppressTransaction: true)` form by hand and keep the model snapshot unchanged.

## Running Migrations in Deployed Environments
- Migrations run **before** the new application version starts: a Kubernetes Job / pre-deploy step running the migration project once — not on startup of every replica.
- One runner at a time. EF Core 9+ takes a migration lock on supported providers; FluentMigrator does not — the pipeline must guarantee a single runner.
- The migration DB user has DDL rights; the application user does not (see `security`).
- The application must start against **both** the previous and the new schema version (expand first, deploy second).

## Rollback
- An expand migration's `Down` may drop structures that already contain new data. Revert the application first and normally leave the expanded schema in place. Run `Down` only after checking that no running version needs the structures, assessing data loss and obtaining the required approval; structural reversibility does not imply a lossless rollback.
- Contract migrations destroy data: their `Down` can recreate the structure but not the data. Run them only after the release that stopped using the structure is stable and recovery coverage has been verified against [data-recovery](../../data-recovery/SKILL.md). A recent backup job without a proven restore/reconciliation path is insufficient. Restoring data or replaying effects is never an automatic deploy-rollback step.
- Rolling back the application is preferred over rolling back a migration: the expanded schema is compatible with the previous version by design.

## MR Checklist for a Migration
- [ ] Which phase is it (expand / contract)? Contract only references structures unused by the current release.
- [ ] `lock_timeout` set for changes to existing tables.
- [ ] Indexes on existing tables created `CONCURRENTLY` in a separate non-transactional migration.
- [ ] No backfill of large tables inside the migration.
- [ ] NOT NULL / FK added through `NOT VALID` + `VALIDATE`.
- [ ] The previous application version works with the new schema.
