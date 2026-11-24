# Integration Testing Reference

## Stack & Infrastructure
- **Framework**: xUnit + `Microsoft.AspNetCore.Mvc.Testing` (`WebApplicationFactory<Program>`).
- **Real Infrastructure**: Testcontainers (`PostgreSql`, `RabbitMq`, `Redis`, RustFS (generic `ContainerBuilder` with `rustfs/rustfs:<tag>`, wait for `/health`), `Kafka`, `MongoDb`, Keycloak).
- Requires Docker running locally or in CI.
- Pass a pinned image to the builder constructor: `new PostgreSqlBuilder("postgres:17.4")`. The parameterless constructors are obsolete in Testcontainers 4.x and depend on whatever default the package ships; never use `latest`.

## Container Fixtures & Collections
Share containers across test suites via xUnit collection definitions:

```csharp
[CollectionDefinition(TestCollectionNames.RedisRabbitMqS3)]
public class RedisRabbitMqS3Collection :
    ICollectionFixture<RedisContainerFixture>,
    ICollectionFixture<RabbitMqContainerFixture>,
    ICollectionFixture<RustFsContainerFixture>;

[Collection(TestCollectionNames.RedisRabbitMqS3)]
public class PaymentControllerTests(ApplicationWebHostFactory factory) : IDisposable { ... }
```

## In-Memory Host Configuration
Override connection strings and mock external 3rd-party services:
```csharp
var client = factory.WithWebHostBuilder(builder =>
{
    builder.UseSetting("ConnectionStrings:redis", redisConnectionString);
    builder.UseSetting("ConnectionStrings:rabbitMessaging", rabbitMqConnectionString);
    builder.MockAuthentication(); // Bypasses Keycloak for standard API tests
    builder.ConfigureServices(sc =>
    {
        sc.AddSingleton<AuditService.AuditServiceClient>(_ => auditServiceMock.Object);
    });
}).CreateClient();
```

## Test Structure
- Arrange / Act / Assert structure.
- Naming matches unit test convention.
- Verify end-to-end effects: HTTP status code, response payload, database state, queue messages, Redis entries.
- Every filter, sort and paging parameter of a domain service or query endpoint has a test with data that the filter must exclude; a filter that is accepted but not applied returns "correct-looking" results otherwise.
