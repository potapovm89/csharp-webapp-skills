---
name: domain-modeling
description: >-
  Use this skill when defining or modifying Domain entities, DbContext, EF Core entity configurations, enums, or DTOs (both in Contracts and Domain).
  Triggers (RU): сущность, entity, DTO, доменная модель, конфигурация EF, мягкое удаление.
---

# Domain Modeling & Contracts

This skill covers the design and organization of domain models, persistence entities, EF configurations, and data transfer objects.

## Entity Implementation Guidelines

All entities live in `Itl.<Project>.Domain/Entities`:
- Inherit from a base class according to primary key: `NumericKeyEntity` (long, default), `GuidKeyEntity` (Guid), `StringKeyEntity` (string), or `BaseEntity` (long + `CreatedAtUtc`/`UpdatedAtUtc` timestamps).
- Implement applicable mixins: `ICanBeSoftDeleted` for normally deletable business entities, `IKnowCreator`/`IKnowUpdater` for author auditing and `ISynchronizable` when cross-service identity is needed. Approved retention/erasure flows and immutable records follow an explicit lifecycle policy (see `security`, input-and-data.md).
- Use the base classes and mixins that exist in `Domain/Entities/Base`; do not invent parallel ones (e.g. an `IAuditable` that duplicates `IKnowCreator` + `IKnowUpdater` with other column names). Timestamp source is either `BaseEntity` or the audit mixins, never both; author-audited entities use the appropriate key base plus `IKnowCreator`/`IKnowUpdater` with columns `created_date`, `created_by`, `updated_date`, `updated_by`. A new base type needs the user's approval and an update of this skill.
- Table and column names must be `snake_case` using attributes `[Table("...")]` and `[Column("...")]`.
- Navigation properties are declared without `[Column]`.
- All properties have XML docs ending with a period.

```csharp
/// <summary>
/// Таблица с информацией о покупателях.
/// </summary>
[Table("customers")]
public class Customer : BaseEntity, ICanBeSoftDeleted, ISynchronizable
{
    /// <summary>
    /// Идентификатор для межсервисной синхронизации сущностей.
    /// </summary>
    [Column("service_uuid")]
    public Guid ServiceUuid { get; set; }

    /// <summary>
    /// Имя покупателя.
    /// </summary>
    [Column("name")]
    public required string Name { get; set; }

    /// <summary>
    /// Связанные заказы.
    /// </summary>
    public ICollection<Order> Orders { get; set; } = [];

    /// <inheritdoc />
    [Column("is_deleted")]
    public bool IsDeleted { get; set; }
}
```

## Entity Configurations

Create a configuration class in `Domain/Configurations` deriving from the base matching the key type (`BaseNumericKeyConfiguration<T>` for `NumericKeyEntity` and `BaseEntity`, `BaseGuidConfiguration<T>`, `BaseStringConfiguration<T>`):
- Define `HasMaxLength` for every string column.
- Index `is_deleted` filters on unique indexes: `.HasFilter("is_deleted = FALSE")`.
- Configure foreign keys and cascade rules.

For more details on entities and DTOs:
- [entity.md](./references/entity.md)
- [dto.md](./references/dto.md)
