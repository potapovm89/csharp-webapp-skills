# Caching Strategy Reference

## When to Cache
- Read-heavy data that changes rarely and tolerates bounded staleness: reference data, configuration, aggregated reports.
- Never cache data that must be strictly consistent (balances, a payment status shown right after a change) and never cache personal data without a short TTL.

## Pattern: Cache-Aside via HybridCache
Use `Microsoft.Extensions.Caching.Hybrid` (`HybridCache`, MIT): in-memory L1 + Redis L2 and tag-based invalidation. Stampede protection coordinates one factory call per key **within one HybridCache instance**; it is not a distributed lock across replicas. Account for simultaneous misses from several replicas when sizing the database.

```csharp
var product = await hybridCache.GetOrCreateAsync(
    CacheKeys.Product(productId),
    async ct => await productDomainService.GetDtoAsync(productId, ct),
    new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),
        LocalCacheExpiration = TimeSpan.FromMinutes(1)
    },
    tags: [CacheTags.PRODUCTS],
    cancellationToken: ct);
```

- Cache DTOs, not EF entities.
- Keys come from one static helper class (`CacheKeys`) and include a version prefix (`v1:product:{id}`), so a DTO shape change never deserializes old entries.
- Every entry has an expiration; add jitter for large key sets to avoid synchronized expiry.

## Invalidation
- After a successful commit, remove or refresh the affected keys (`RemoveAsync(key)` / `RemoveByTagAsync(tag)`).
- Key/tag invalidation affects the current server and L2; do not assume it immediately clears the L1 of every other replica. Choose and document the tolerated staleness and bound `LocalCacheExpiration` accordingly. If every replica needs an invalidation event, use explicit fan-out; a competing-consumer group delivers to one consumer, not every cache instance.
- Several services caching the same data: publish a change event through the outbox and invalidate in each relevant service. This does not by itself guarantee synchronous invalidation of every process's L1.
- Never invalidate before the commit: a concurrent read would cache the old value again.

## Failure Behavior
- The cache is an optimization: its failure should not fail an otherwise serviceable read. Verify the resolved HybridCache/Redis versions for read, write, invalidation and timeout failures; cache-aside alone does not guarantee fail-open for every exception. Where needed, handle cache-specific failures at the cache boundary and use a bounded database fallback. Do not swallow source/database errors or caller cancellation, and do not execute the source factory twice when it has already completed.
- Bound cache timeouts and fallback concurrency so a Redis outage does not overwhelm the database. Record degraded-cache metrics and rate-limited warnings without keys containing personal data.
- Do not use the cache as a message bus or as the only copy of data.

## Verification
- With two app instances, populate both L1 caches, update the source and invalidate on one instance; the other returns fresh data within the documented staleness bound.
- Stop Redis or inject read/write/invalidation timeouts and verify the intended fallback, cancellation and database load limits for the resolved package versions.
- Concurrent misses invoke the factory once per key per instance; do not assert a single invocation across all replicas.

Source: [Microsoft HybridCache documentation](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid?view=aspnetcore-10.0).
