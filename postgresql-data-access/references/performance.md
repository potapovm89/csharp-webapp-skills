# Query Performance Reference

## Reading Data
- Project to DTOs with `.Select(...)` in the domain service; never load entity graphs to return a list.
- `AsNoTracking()` for reads; tracked queries only when the entities are modified in the same unit of work.
- Avoid N+1: no queries inside loops over results. Load related data in one query (`Include`, projection, `WHERE id = ANY(@Ids)`).
- Several collection `Include`s in one query cause a cartesian explosion; use `AsSplitQuery()` or separate queries.
- `IQueryable` never leaves the domain service.

## Pagination
- Keyset (cursor) pagination for large or growing tables: `WHERE (created_date, id) < (@Date, @Id) ORDER BY created_date DESC, id DESC LIMIT @Limit` with a matching composite index.
- `OFFSET` gets linearly slower and skips/duplicates rows under concurrent inserts; use it only for small bounded lists.
- `COUNT(*)` on big tables is expensive; return totals only when the UI needs them.
- Every paged query has an explicit `ORDER BY` on a unique key (or ends with one); without it PostgreSQL may return rows in a different order per page.
- Filters are passed as `Expression<Func<TEntity, bool>>` and applied with `.Where(filter)` before projection and paging; cover each filter with an integration test (see `testing`).

## Writing Data
- Bulk changes with `ExecuteUpdateAsync` / `ExecuteDeleteAsync` (one statement) instead of loading and saving entities one by one.
- Large inserts: batched `AddRange` + one `SaveChanges`, or Npgsql binary `COPY` for imports.
- Long backfills run as resumable batches outside migrations (see `database-migrations`, zero-downtime.md).

## Indexes
- Every foreign key column and every column used in frequent `WHERE` / `ORDER BY` has an index; composite index column order follows the query (equality columns first, then range/sort).
- Partial indexes for hot subsets (`WHERE processed_date IS NULL`, `WHERE is_deleted = FALSE`).
- Check plans with `EXPLAIN (ANALYZE, BUFFERS)` on a realistic data volume before and after a change. Evaluate latency, rows read, selectivity and buffers against the scenario's target; a `Seq Scan` can be optimal when reading much of a table. Do not force an index solely to remove a sequential scan.
- Indexes on existing tables are created `CONCURRENTLY`.

## Connections
- Watch the Npgsql pool metrics (`observability`): pool exhaustion shows up as timeouts, not as SQL errors.
- Keep business transactions short; no HTTP/gRPC/broker calls inside them. The outbox relay may hold **outbox-only** locks during bounded broker publishing as explicitly described in `message-brokers`, outbox.md; it is not a pattern for business handlers.
- Set `Command Timeout` in the connection string; long reports go to a read replica if one exists.

## Load Testing
- Performance-sensitive endpoints get a load test scenario before release: target RPS, p95/p99 latency, error rate.
- Allowed tools: k6 (AGPL, used as a standalone tool — allowed), Apache JMeter (Apache-2.0). NBomber is commercial for organizations — prohibited.
