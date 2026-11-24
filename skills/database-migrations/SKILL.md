---
name: database-migrations
description: >-
  Use this skill when creating, executing, rolling back, or verifying database migrations for PostgreSQL using EF Core Migrations or FluentMigrator, planning zero-downtime schema changes (expand/contract, concurrent indexes, NOT NULL and FK without long locks, backfills), and when updating DBML schema documentation.
  Triggers (RU): миграция, схема БД, EF миграция, FluentMigrator, миграция без простоя, индекс CONCURRENTLY, откат.
---

# Database Migrations

All database schema evolutions in the solution must be executed strictly via migrations. Manual modifications to database schemas are prohibited.

## Core Rules

1. **Standalone Migration Projects**:
   - Migrations live in dedicated runnable console projects located in the `Migrations/` solution folder:
     - `Itl.<Project>.EntityFramework.Migrations` (for EF Core databases).
     - `Itl.<Project>.FluentMigrator.Migrations` (for Dapper databases).
   - In AspireHost, migration projects run before host applications: hosts depend on `.WaitForCompletion(<migrationsProject>)`.
2. **Never Edit Merged Migrations**:
   - Once a migration is committed to the main branch, editing it is strictly forbidden. Create a new subsequent migration.
3. **Reversible**:
   - Every migration has a reviewed rollback strategy and a working structural `Down` where applicable. A `Down` that drops added data is destructive, not a lossless undo. Contract migrations run only after the previous release is stable and the recovery plan is verified; backup/restore acceptance follows [data-recovery](../data-recovery/SKILL.md).
4. **Zero Downtime**:
   - The previous application version keeps running during a rolling deployment, so every migration is backward compatible: expand → migrate code and backfill → contract in a later release. Never rename or drop in place what the previous release still uses.
   - Changes to existing tables set `lock_timeout`; indexes on existing tables are created `CONCURRENTLY` in a separate non-transactional migration; NOT NULL and foreign keys go through `NOT VALID` + `VALIDATE`; large backfills run as batched jobs, not inside migrations.
   - In deployed environments migrations run once, as a pre-deploy job, before the new version starts — not on startup of every replica.
   - [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md) binds this Job to the release manifest and blocks rollout on failure. Material schema/storage/extension changes also review recovery compatibility and applicable restore-drill evidence; migration tests alone do not establish RPO/RTO.
   - See [zero-downtime.md](./references/zero-downtime.md). **Read it before writing any migration that touches an existing table.**
   - **`dotnet ef migrations add` output is never zero-downtime as generated.** For every operation on an existing table, hand-edit the migration per the table in zero-downtime.md and keep the model snapshot unchanged. Review the generated SQL (`dotnet ef migrations script`) before committing.
   - Typical generated anti-patterns to rewrite:
     - `AddColumn(..., nullable: false, defaultValue: Guid.Empty / 0)` for a new FK column + `AddForeignKey` → the FK fails on existing rows or points to a non-existent parent. Rewrite: add the column nullable → backfill real values (job or explicit `UPDATE` for small tables) → `NOT VALID` FK → `VALIDATE` → NOT NULL via validated `CHECK`, contract step in a later release.
     - `CreateIndex` / `DropIndex` on an existing table → `Sql("CREATE/DROP INDEX CONCURRENTLY ...", suppressTransaction: true)` in a separate migration.
     - Several existing tables changed in one migration without `SET lock_timeout`.
   - "The table is empty in my environment" is not a reason to skip these rules: the migration must be safe for any environment that already has data.
5. **Synchronized Documentation**:
   - Whenever a migration modifies the database schema, update `docs/database/postgresql/<dbname>/schema.dbml` in the exact same commit or PR.

## EF Core Migrations

- Add migration:
  ```bash
  dotnet ef migrations add <MigrationName> --project src/Migrations/Itl.<Project>.EntityFramework.Migrations --startup-project src/Migrations/Itl.<Project>.EntityFramework.Migrations
  ```
- Rollback:
  - Run the migration project with `--d <TargetMigrationName>` (or `--d 0` to roll back everything).

## FluentMigrator

- Structure: Group migrations by year (`2024/`, `2025/`).
- Naming: Class named `Migration_yyyyMMddHHmm`.
- Migration attribute specifies timestamp version and task number:
  ```csharp
  [Migration(202611052200, "TASK-123")]
  public class Migration_202611052200 : Migration
  {
      public override void Up()
      {
          Create.Table("product_categories")
              .WithColumn("id").AsInt64().NotNullable().PrimaryKey().Identity()
              .WithColumn("name").AsString(100).NotNullable()
              .WithColumn("is_deleted").AsBoolean().NotNullable().WithDefaultValue(false)
              .WithColumn("created_date").AsDateTimeOffset().NotNullable().WithDefault(SystemMethods.CurrentUTCDateTime);
      }

      public override void Down()
      {
          Delete.Table("product_categories");
      }
  }
  ```

## DBML Schema Documentation

Update `docs/database/postgresql/<dbname>/schema.dbml`:
```dbml
Table product_categories {
  id bigint [pk, increment]
  name varchar(100) [not null]
  is_deleted boolean [not null, default: false]
  created_date timestamptz [not null]
}
```
