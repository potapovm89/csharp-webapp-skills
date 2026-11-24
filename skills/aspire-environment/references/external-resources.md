# Aspire External Resources Reference

## Standard Resource Builders in `AspireHost`
- `UsePostgreSQL("PostgreSqlOptions", out var postgres)`: Spawns PostgreSQL container with persistent volume (`WithDataVolume`). Databases created via `postgres.CreateDatabase("dbname", out var resource)`.
- `UseKafka("KafkaOptions", out var kafkaMessaging)`: Spawns Kafka container; topics auto-created based on `KafkaOptions.Topics`.
- `UseRabbitMq("RabbitMqOptions", out var rabbitMessaging)`: Spawns RabbitMQ with exchanges/queues defined in `RabbitMqOptions.Exchanges`.
- `UseRedis("RedisOptions", out var redis)`: Spawns Redis / Valkey (image per `RedisOptions`; Valkey is the default recommendation for new setups — `csharp-code-style`, libraries.md).
- `UseMongoDb("MongoDbOptions", out var mongo)`: Spawns MongoDB.
- `UseRustFs("RustFsOptions", out var s3Storage)`: Spawns RustFS (S3-compatible, Apache-2.0, `rustfs/rustfs` with a pinned tag) and initializes buckets from `RustFsOptions.Buckets`. MinIO is not used: its images are no longer published openly (Docker Hub repository removed, other registries require authentication).
- `UseKeycloak("KeycloakOptions", postgres, out var keycloak)`: Spawns Keycloak backed by Postgres, mounting realm JSON from `Configs/`.

## Resource Referencing Rules
- Use `AddInfrastructureReference(resource)` on projects that need the resource's connection information. Inspect the local helper's implementation to see whether it also calls `WaitFor`. If it only adds the reference, add a readiness wait when startup depends on the resource being usable. A readiness wait requires a health check on the resource to go beyond the running state.
- Use `WaitForCompletion(migrationsProject)` for a host that needs the schema created by that runner.
