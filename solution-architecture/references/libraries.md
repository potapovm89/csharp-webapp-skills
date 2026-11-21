# Solution Libraries Reference

## Mandatory Libraries in Solution

- **Itl.<Project>.Common**: Infrastructure code with zero business logic. Contains API attributes, filters, middlewares, exception handlers, authentication schemes/roles, broker extensions, common exceptions (`NotFoundException`), configuration extensions (`AddEnvironmentConfigurations`, `AddServiceDefaults`, `AddOptionsWithValidation`), manual mapping extension methods (`Mappers/`, no AutoMapper), and utility tools.
- **Itl.<Project>.Contracts**: Inter-service contracts and public API DTOs. Root contains common DTOs (`HttpErrorDto`, `IdDto`), while feature-specific DTOs reside in domain folders (`Account/`, `Orders/`, etc.). All classes have the `Dto` suffix. Does not depend on any internal projects.
- **Itl.<Project>.DataAccess**: Persistence access (PostgreSQL via EF / Dapper, MongoDB, Redis, S3-compatible storage). Contains base domain services, concrete `I<Entity>DomainService` implementations, technology options, and DI registration extensions.
- **Itl.<Project>.Domain**: Domain model with zero dependencies on other projects. Contains entities (`Entities/`), EF Core configurations (`Configurations/`), enums (`Enumerations/`), internal DTOs (`Dtos/`), and the EF `DbContext` + context factory.
- **Itl.<Project>.Stories**: Business logic using CQRS. Contains FluentValidation validators and the units of the solution's single Stories mechanism: MediatR commands/queries/handlers **or** Executor Stories (`cqrs-stories`). Marker interface `IStoryLibrary` is used for DI assembly scanning.
- **Itl.<Project>.Migrations**: Standalone runnable migration console projects (EF Migrations and/or FluentMigrator): run by Aspire before the hosts locally and once as a pre-deploy job in deployed environments (see `database-migrations`).
- **Itl.<Project>.Tests**: Shared base classes for tests (`BaseTestCasesClass`, `BaseInvalidRequest`).
- **Itl.<Project>.UnitTests**: Unit tests mirroring host project structures.
- **Itl.<Project>.IntegrationTests**: Integration tests with Testcontainers.

## Conditional / Optional Libraries

- **Itl.<Project>.Executor**: Generic story execution engine (`IStory<TContext, TResult>`, `IStoryExecutor`, `StoryExecutor`, `AddExecutors`).
- **Itl.<Project>.Protos**: Protobuf definitions and generated gRPC contracts.
- **Itl.<Project>.AspireHost**: Orchestration harness for local development and integration tests.
- **Itl.<Project>.<Type>Host**: Runnable entrypoint applications (`WebHost`, `RpcHost`, `RealtimeHost`, `KafkaHost`, `RabbitHost`, `WorkerHost`, `BlazorHost`).
- **Itl.<Project>.Documents** (optional): Razor class library with document templates rendered to PDF; references only Contracts (`pdf-reports`).
