# Database Documentation (DBML) Reference

## Locations
- PostgreSQL: `docs/database/postgresql/<dbname>/schema.dbml` (dbdiagram.io syntax).
- MongoDB: `docs/database/mongodb/<dbname>/` (collections, fields, indexes).
- Redis: `docs/database/redis/<purpose>/` (key patterns, types, TTLs, writers/readers).

## DBML Schema Conventions
- Table and column names strictly match database `snake_case`.
- Relationships documented via `Ref: child_table.parent_id > parent_table.id`.
- Filtered unique indexes and notes: `(email, is_deleted) [unique, note: 'is_deleted = FALSE']`.
- Must be updated in the exact same MR as the migration.
