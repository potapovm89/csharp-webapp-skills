---
name: postgresql-data-access
description: >-
  Use this skill when designing PostgreSQL schemas, writing database queries, configuring database connections, implementing domain services using Entity Framework Core or Dapper, or managing transactions with ITransactionScope.
  Triggers (RU): PostgreSQL, запрос к БД, доменный сервис, EF Core, Dapper, транзакция, блокировки, конкурентность, производительность запросов, пагинация.
---

# PostgreSQL Data Access

This skill defines database modeling rules, naming standards, and data access implementations for PostgreSQL in `Itl.<Project>.DataAccess`.

## Schema Design Rules

- **Table Names**: `snake_case`, plural (`customers`, `store_orders`, `order_statuses`).
- **Column Names**: `snake_case` (`customer_id`, `created_date`, `is_deleted`).
- **Primary Key**: Column `id`, type `bigint` identity (`NumericKeyEntity`) or `uuid` (`GuidKeyEntity`).
- **Foreign Keys**: `<singular_table>_id` (e.g. `customer_id`, `store_order_id`).
- **Standard Columns**:
  - Timestamps (UTC), depending on the entity base (see `domain-modeling`):
    - `create_date` / `update_date` (nullable) for entities derived from `BaseEntity`;
    - `created_date` / `updated_date` + `created_by` / `updated_by` for audited entities (`IKnowCreator` / `IKnowUpdater`).
  - `is_deleted` boolean for soft deletion.
- **Strict Guidelines**:
  - Soft deletion (`is_deleted`) is the default for normally deletable business entities. Approved erasure/anonymization, retention expiry and immutable-record lifecycles are documented exceptions (see `domain-modeling` and `security`). Physical deletion still requires authorization; a soft-delete flag is not erasure.
  - Unique indexes on soft-deleted tables must include `.HasFilter("is_deleted = FALSE")`.
  - All timestamps stored in UTC. Monies stored in `decimal`. Statuses/enums stored as `int`.
  - Max length constraints are mandatory for all string columns.
  - Modifying the database manually is strictly forbidden; all schema evolution must go through migrations.

## Choosing EF Core vs. Dapper

Each database has one primary access mechanism:
- **Entity Framework Core**: For main domain databases with complex models, relationships, and state mutations. See [ef-core.md](./references/ef-core.md).
- **Dapper**: For high-load, read-heavy workloads, simple tables, and broker consumers. See [dapper.md](./references/dapper.md).

Both mechanisms expose identical external ergonomics to Stories:
1. Access via `I<Entity>DomainService`.
2. Multi-entity atomic operations via `ITransactionScope`.

## Concurrency & Performance
- Entities updated after creation carry a concurrency token (`xmin`); conflicts become 409 / skip, never a silent overwrite. Every entity that has an Update story/handler gets it:
  ```csharp
  /// <summary>
  /// Версия строки (системная колонка PostgreSQL xmin).
  /// </summary>
  [Timestamp]
  public uint Version { get; set; }
  ```
  The client sends the version it read (DTO field `Version`); `DbUpdateConcurrencyException` is mapped to `ConflictException` → 409. Details: [concurrency.md](./references/concurrency.md).
- Uniqueness is enforced by unique indexes, not by "check then insert".
- Reads are projections with `AsNoTracking()`; no N+1; keyset pagination for large tables; plans checked with `EXPLAIN (ANALYZE, BUFFERS)`.

## References
- [ef-core.md](./references/ef-core.md) — EF Core domain services and transaction scope.
- [dapper.md](./references/dapper.md) — Dapper domain services.
- [concurrency.md](./references/concurrency.md) — optimistic (`xmin`) and pessimistic locking, unique constraints, advisory locks, Idempotency-Key table.
- [performance.md](./references/performance.md) — projections, N+1, pagination, bulk writes, indexes, connection pool, load testing (k6 / JMeter).
