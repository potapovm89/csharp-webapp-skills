# MongoDB Data Access Reference

## Connection & DI
```csharp
builder.AddMongoDBClient("mongoDbForRpc");
builder.Services.AddOptionsWithValidation<MongoDbOptions>(builder.Configuration.GetRequiredSection(nameof(MongoDbOptions)).Bind);
builder.Services.AddScoped<IMongoDBRepository, MongoDBRepository>();
```

## Entity Guidelines
- Document entities inherit from `GuidKeyEntity`.
- Decorate the primary key with `[BsonId]`:
  ```csharp
  public class AuditRecord : GuidKeyEntity
  {
      [BsonId]
      public override Guid Id { get; set; }
      public required string Action { get; set; }

      /// <summary>
      /// Дата создания записи (UTC).
      /// </summary>
      public DateTime CreatedDate { get; set; }   // Kind = Utc only; the BSON date type is a UTC instant
  }
  ```
- Timestamps are UTC only (`csharp-code-style`, time-money-culture.md): the driver serializes BSON dates as UTC instants — a local time silently shifts on read. Apply `DateTime.SpecifyKind(..., DateTimeKind.Utc)` or use `DateTimeOffset`.

## Repository Usage
All access must go through `IMongoDBRepository`:
- Query with typed predicates: `Expression<Func<T, bool>>`.
- Always pass `CancellationToken`.
- Document schema changes in `docs/database/mongodb/<dbname>/`.
