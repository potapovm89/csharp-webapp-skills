# Redis (Valkey) Reference

## Connection & DI
```csharp
builder.Services.AddRedis(builder.Configuration.GetRequiredSection(nameof(RedisOptions)).Bind);
builder.Services.AddScoped<IRedisRepository, RedisRepository>();
```

## Repository API
- `SaveAsync<T>(key, value, expiry)`: Saves JSON value with mandatory TTL.
- `GetAsync<T>(key, expiry?)`: Reads value, optionally sliding expiry.
- `DeleteAsync(key)`: Removes key.
- `ContainsKey(key)`: Checks existence.
- `GetKeys(pattern, pageSize, pageOffset)`: iterates `SCAN` internally (never the blocking `KEYS` command); paging helpers are for diagnostics and cleanup, not for exact pagination.

## SCAN Semantics
- `SCAN` returns an opaque cursor: "page N" is not a stable snapshot — keys can appear, disappear or repeat between iterations. The helper pages by continuing the cursor; results are **eventually consistent**.
- Never base authorization or correctness decisions on SCAN results: it serves diagnostics, cleanup and key inventory (`docs/database/redis/`).
- On Redis Cluster, scanning runs per node and multi-key operations need hash tags (`{purpose}` in the key) — confirm the deployment topology before writing key helpers.

## Key Naming & Rules
- Mandatory TTL (`TimeSpan expiry`) on every save.
- Keys must be generated via dedicated helper classes (e.g. `PaymentKeyHelper.GetKey(...)`).
- Format: `<purpose>_<identifier1>_<identifier2>` (e.g. `to_update_status_<storeId>_<orderId>`).
- Document all keys in `docs/database/redis/<purpose>/`.
