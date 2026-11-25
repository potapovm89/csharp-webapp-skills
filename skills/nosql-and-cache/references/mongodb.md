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
- Store timestamps as UTC instants. At ingestion, prefer `DateTimeOffset` for values with a supplied offset and use `value.UtcDateTime` for a BSON date. `12:00 +03:00` must store and round-trip as `09:00Z`. For wall-clock values with a known source zone, resolve through `TimeZoneInfo` using that zone and handle ambiguous or invalid DST times explicitly. Reject input with no reliable zone or offset unless the source contract explicitly says it is UTC. Use `DateTime.SpecifyKind(value, DateTimeKind.Utc)` only to label an already-UTC value whose `Kind` was lost.

## Repository Usage
All access must go through `IMongoDBRepository`:
- Query with typed predicates: `Expression<Func<T, bool>>`.
- Always pass `CancellationToken`.
- Document schema changes in `docs/database/mongodb/<dbname>/`.
