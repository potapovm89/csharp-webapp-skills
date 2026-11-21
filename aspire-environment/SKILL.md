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
   - Register migration runners with infrastructure dependencies.
4. **Host Projects**:
   - Add projects with references: `.AddInfrastructureReference(...)` and `.WaitForCompletion(migrations)`.

## Parameters and Secrets in Resources
- Pass `ParameterResource`s and `ReferenceExpression`s to `WithEnvironment(...)` / `WithArgs(context => ...)`; Aspire resolves them at start: `context.Args.Add(ReferenceExpression.Create($"--db-password={postgres.Resource.PasswordParameter}"));`.
- `ParameterResource.Value` is obsolete; when code really needs the value (e.g. a client in a `ResourceReadyEvent` handler), use `await parameter.GetValueAsync(ct)`.
- Postgres user name: `PostgresServerResource.UserNameReference` (works with and without a custom user parameter).

## Golden Invariant
Resource names in AspireHost must strictly match connection string keys in `shared/configs/connectionStrings.{env}.json` and `builder.Configuration.GetConnectionString("<Name>")`.

## Subtopic References
- [external-resources.md](./references/external-resources.md) — Prebuilt and custom Docker resource builders.
- [settings.md](./references/settings.md) — Configuration alignment and kebab-case validators.
