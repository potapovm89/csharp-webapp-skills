---
name: aspire-environment
description: >-
  Use this skill when configuring .NET Aspire orchestration in AspireHost, wiring infrastructure containers (Postgres, Kafka, RabbitMQ, Mongo, Redis, RustFS (S3), Keycloak), or managing local environment dependencies.
  Triggers (RU): Aspire, локальный полигон, AppHost, контейнеры для разработки, ресурсы, docker-compose.
---

# Aspire Local Orchestration

This skill covers orchestrating local development environments using Aspire in `Itl.<Project>.AspireHost`.

## Overview
- **Version line**: Aspire 13.x — the release line shipped together with .NET 10 (the product was renamed from ".NET Aspire" to "Aspire" in November 2025). The Aspire SDK / AppHost package versions are pinned in `AspireHost`; an Aspire upgrade is a separate task, not a side effect of a feature.
- `AspireHost` launches the entire infrastructure stack in Docker containers with a single command.
- Provides a centralized dashboard with OpenTelemetry traces, metrics, logs, and health statuses.
- Runs only on local developer machines; never deployed to production.

## Composition Sequence in `Program.cs`
1. **Infrastructure Resources**:
   - `builder.UseKafka(...)`, `builder.UseRabbitMq(...)`, `builder.UsePostgreSQL(...)`, `builder.UseRedis(...)`, `builder.UseMongoDb(...)`, `builder.UseRustFs(...)`, `builder.UseKeycloak(...)`.
2. **Databases**:
   - Create named databases matching connection string keys: `postgres.CreateDatabase("postgresdbForAPI", out var apiDb)`.
3. **Migration Projects**:
   - Register each migration runner with its actual database reference and a readiness dependency. Use `.WaitFor(database)` with a configured health check when the runner needs a usable database. Without a health check, `WaitFor` only waits for the resource to be running. [Aspire WaitFor](https://aspire.dev/hub/glossary/waitfor/)
4. **Host Projects**:
   - Add the required connection references with `.AddInfrastructureReference(...)`. A dependent host uses `.WaitForCompletion(migrations)` to wait for the migration runner's successful exit. For other infrastructure resources, use a readiness wait when the host needs the resource available at startup. [Aspire WaitForCompletion](https://aspire.dev/hub/glossary/waitforcompletion/)

**Example** — A service depends on a PostgreSQL resource and a migration runner. Use `WaitFor` on the database with a readiness health check and `WaitForCompletion` on the runner. Verify that the service starts only after the database is ready and the runner has succeeded.

## Verify Local Startup

1. Read the AppHost project and its `launchSettings.json`. Select the configured local launch profile and verify that its effective connection references lead to local test resources.
2. If the graph needs containers, check the configured runtime is reachable, for example with `docker info`. If it is unavailable, report the blocked check and validate the wiring from code and configuration.
3. Run `dotnet run --project <AspireHost.csproj> --launch-profile <local-profile>` with the resolved project and profile. Use the dashboard or logs to observe prerequisite health, migration exit status and dependent host startup. A failed migrator blocks dependent startup, so inspect its error and configuration first.

## Parameters and Secrets in Resources
- Pass `ParameterResource`s and `ReferenceExpression`s to `WithEnvironment(...)` / `WithArgs(context => ...)`; Aspire resolves them at start: `context.Args.Add(ReferenceExpression.Create($"--db-password={postgres.Resource.PasswordParameter}"));`.
- `ParameterResource.Value` is obsolete; when code really needs the value (e.g. a client in a `ResourceReadyEvent` handler), use `await parameter.GetValueAsync(ct)`.
- Postgres user name: `PostgresServerResource.UserNameReference` (works with and without a custom user parameter).

## Golden Invariant
Resource names in AspireHost must strictly match connection string keys in `shared/configs/connectionStrings.{env}.json` and `builder.Configuration.GetConnectionString("<Name>")`.

## Subtopic References
- [external-resources.md](./references/external-resources.md) — Prebuilt and custom Docker resource builders.
- [settings.md](./references/settings.md) — Configuration alignment and kebab-case validators.

## Completion Evidence

- Record the actual launch command, the changed dependency order and the observed resource states. For a DB-backed host, verify database readiness, successful migration completion and then the host's readiness endpoint.
- When changing startup dependencies, use a local fixture to delay required resource readiness or fail the migrator. Verify that the dependent host stays blocked until the required condition is satisfied. Record blocked or unavailable runtime checks explicitly.
