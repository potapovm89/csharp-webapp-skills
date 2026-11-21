# Project Helpers (signatures)

Reference signatures of template helpers. Namespaces follow `Itl.<Project>.<Layer>`. Confirm them against the current solution; if a helper is missing or differs, use its real API and implement/report missing required behavior rather than inventing a call. Examples using `IOutboxDomainService` below consistently use `Add` plus the surrounding transaction commit.

## Configuration (`Common/Extensions`)
```csharp
// Binds, validates DataAnnotations, fails on start if invalid.
public static OptionsBuilder<T> AddOptionsWithValidation<T>(
    this IServiceCollection services, Action<T> configureOptions) where T : class;

// Usage
builder.Services.AddOptionsWithValidation<MyOptions>(
    builder.Configuration.GetRequiredSection(nameof(MyOptions)).Bind);
```

Host bootstrap (every host): `builder.AddEnvironmentConfigurations();`, `builder.AddServiceDefaults();`, `app.MapDefaultEndpoints(port);`.

## Transactions (`DataAccess/EF|Dapper/TransactionScopes`)
```csharp
public interface ITransactionScope
{
    Task<IAsyncDisposable> BeginAsync(CancellationToken ct);
    Task CommitAsync(CancellationToken ct);
    Task RollbackAsync(CancellationToken ct);
}

// Usage
await using (await transactionScope.BeginAsync(ct))
{
    try
    {
        // domain service calls
        await transactionScope.CommitAsync(ct);
    }
    catch
    {
        await transactionScope.RollbackAsync(CancellationToken.None);
        throw;
    }
}
```

## Outbox (`Common/Brokers/Outbox`, `DataAccess/EF/Services/Outbox`)
```csharp
public static class OutboxMessageFactory
{
    // Captures the current trace context and correlation id into headers.
    public static OutboxMessage Create<TPayload>(
        OutboxBroker broker,          // Kafka | RabbitMq
        string destination,           // topic or exchange
        string? messageKey,           // aggregate id / routing key
        string messageType,           // kebab-case constant
        TPayload payload,
        int messageVersion = 1,
        string? correlationId = null);
}

public interface IOutboxDomainService
{
    void Add(OutboxMessage message);  // saved by the surrounding transaction
    Task<IReadOnlyList<OutboxMessage>> LockPendingAsync(int batchSize, CancellationToken ct);
    Task<int> DeleteProcessedAsync(DateTime processedBefore, int batchSize, CancellationToken ct);
    Task<(long Pending, DateTime? OldestCreatedDate)> GetBacklogAsync(CancellationToken ct);
}

// Relay registration (host that publishes)
services.AddOutboxRelay(configuration.GetRequiredSection(nameof(OutboxOptions)).Bind);
```

## Consumers (`Common/Services/ConsumerProcessor`, `DataAccess/Dapper/Services/Inbox`)
```csharp
public interface IProcessor<TKey, TMessage>
{
    Task ExecuteAsync(ConsumedMessageDto<TKey, TMessage> message, CancellationToken ct = default);
}

public interface IInboxDomainService
{
    // true — first delivery, process it; false — duplicate, skip.
    Task<bool> TryRegisterAsync(string consumer, Guid messageId, CancellationToken ct);
}

// Throw to send a message straight to the DLQ without retries.
public class PoisonMessageException : Exception;
```

## Locks (`Common/Services/Locks`, implementation over `DataAccess/Locks`)
```csharp
public interface IDistributedJobLock
{
    // null — the lock is held by another replica. Dispose releases it.
    Task<IAsyncDisposable?> TryAcquireAsync(long key, CancellationToken ct);
}

services.AddDistributedJobLock();   // PostgreSQL advisory lock on a dedicated connection
```

Jobs use `IDistributedJobLock`; the low-level `PostgresAdvisoryLock.TryAcquireAsync(DbConnection, key, ct)` stays inside the implementation.

Lock keys are constants in one class per host (`JobLockKeys`), never reused.

## Cache (`DataAccess/Redis`)
```csharp
public interface IRedisRepository
{
    Task SaveAsync<T>(string key, T value, TimeSpan expiry);   // expiry is mandatory
    Task<T?> GetAsync<T>(string key, TimeSpan? expiry = null);
    Task<bool> DeleteAsync(string key);
    Task<bool> ContainsKey(string key);
    Task SetExpiry(string key, TimeSpan expiry);
    IAsyncEnumerable<RedisKey> GetKeys(string? keyPattern = null, int pageSize = 250, int pageOffset = 0); // SCAN
}
```

## Security (`Common/Authentication`, `Common/Api/Middlewares`)
```csharp
public static string GetStoreId(this ClaimsPrincipal principal);   // throws ForbiddenException if the claim is missing
public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder app);
// Policy names: AuthorizationPolicies.STORE_CLIENT, AuthorizationPolicies.ADMIN; claim names: AppClaims.STORE_ID.
```
