---
name: nosql-and-cache
description: >-
  Use this skill when integrating or working with NoSQL document storage (MongoDB) or caching and fast key-value storage (Redis / Valkey), including repositories, entity models, key conventions, and TTL configurations.
  Triggers (RU): Redis, Valkey, KeyDB, кэш, MongoDB, TTL, HybridCache, инвалидация кэша.
---

# NoSQL and Caching (MongoDB & Redis)

This skill covers working with non-relational document data (MongoDB) and caching / distributed state (Redis / Valkey).

## MongoDB (Document Storage)
- **Use case**: Weakly structured or audit data (e.g. audit logs in RpcHost).
- **Client**: `Aspire.MongoDB.Driver` via `builder.AddMongoDBClient("...")`.
- **Model**: Inherits from `GuidKeyEntity`, ID decorated with `[BsonId]`.
- **Repository**: All access is through `IMongoDBRepository` using typed lambda expressions (`Expression<Func<T, bool>>`). Direct usage of `IMongoClient` in business logic is prohibited.
- See [mongodb.md](./references/mongodb.md).

## Redis / Valkey (Cache & Ephemeral State)
- **Use case**: Distributed caching, short-lived processing locks, and operation statuses. Never use as the primary persistent data store.
- **Client**: `Aspire.StackExchange.Redis` / `DataAccess/Redis`.
- **Repository**: All access is through `IRedisRepository`.
- **Mandatory TTL**: Every saved item MUST have an explicit `expiry` parameter.
- **Key naming**: Generate keys via static helper classes using pattern `<purpose>_<id1>_<id2>`.
- **Keys retrieval**: Always use cursor-based SCAN via `GetKeys(pattern, ...)`, never the blocking `KEYS` command.
- See [redis.md](./references/redis.md).

## Caching Strategy
- Cache-aside through `HybridCache` (L1 memory + L2 Redis, stampede protection); cache DTOs with versioned keys and an expiration.
- Invalidate only after the commit; if Redis is down, fall back to the database instead of failing the request.
- See [caching-strategy.md](./references/caching-strategy.md).
