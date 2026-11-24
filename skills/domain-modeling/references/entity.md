# Entity Conventions Reference

## Base Entity Classes (`Domain/Entities/Base`)
- `NumericKeyEntity`: `long` (identity bigint). Default for relational PostgreSQL tables.
- `GuidKeyEntity`: `Guid`. Used for MongoDB documents or entities with global UUID keys.
- `StringKeyEntity`: `string`. Natural string keys.
- `BaseEntity`: `long` key + system timestamps `CreatedAtUtc` (column `create_date`) and `UpdatedAtUtc` (column `update_date`, nullable).

## Mixin Interfaces (`Domain/Entities/Base`)
- `IKnowCreator`: Adds `CreatedDate` (column `created_date`) and `CreatedBy` (`created_by`).
- `IKnowUpdater`: Adds `UpdatedDate` (column `updated_date`) and `UpdatedBy` (`updated_by`).

Timestamps come from **one** source per entity: either `BaseEntity` (`CreatedAtUtc`/`UpdatedAtUtc`) or the audit mixins `IKnowCreator`/`IKnowUpdater` (`CreatedDate`/`UpdatedDate` + author). Never combine both in one entity, and never invent other variants (`CreatedAt`, `CreationTime`). All timestamps are UTC.
- `ICanBeSoftDeleted`: Adds `bool IsDeleted`; use for normally deletable business entities. Soft delete does not satisfy erasure or retention expiry. An approved deletion/anonymization flow defines dependent data, retention and audit requirements; immutable records follow their own lifecycle. Do not physically delete business data without the required authorization.
- `ISynchronizable`: Adds `Guid ServiceUuid` for cross-service identity.

## EF Core Configuration (`Domain/Configurations`)
- Choose the configuration base class by key type:
  - `NumericKeyEntity` and `BaseEntity` (both `long`) -> `BaseNumericKeyConfiguration<T>`.
  - `GuidKeyEntity` -> `BaseGuidConfiguration<T>`.
  - `StringKeyEntity` -> `BaseStringConfiguration<T>`.
- Mandatory string length caps: `builder.Property(x => x.Name).HasMaxLength(100);`.
- For `ICanBeSoftDeleted`, filter unique indexes:
  ```csharp
  builder.HasIndex(u => new { u.Email, u.IsDeleted })
      .HasFilter("is_deleted = FALSE")
      .IsUnique();
  ```
- All configurations are automatically loaded via `ApplyConfigurationsFromAssembly` in DbContext.
