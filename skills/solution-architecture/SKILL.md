---
name: solution-architecture
description: >-
  Use this skill when designing or restructuring solution projects, adding new libraries or host applications, configuring project dependencies, or implementing bootstrap code in Program.cs across hosts.
  Triggers (RU): архитектура, слои, новый проект в солюшене, зависимости проектов, мультитенантность.
---

# Solution Architecture

This skill defines the modular monorepo architecture for the project. Exact signatures of project helpers (`AddServiceDefaults`, `AddOptionsWithValidation`, base domain services, etc.) are defined in the current solution's `Common`, `DataAccess` and `Domain` projects: read them there instead of guessing.

This project's Domain includes EF Core configurations/DbContext and is therefore persistence-aware; zero dependencies on **solution projects** does not mean zero infrastructure-package dependencies. This is a deliberate application architecture, not a universal persistence-independent domain. Keep business invariants in Stories under the chosen convention. A different domain/module boundary requires an explicit ADR; do not restructure an existing solution as part of a routine feature.

## Architectural Layers & Dependencies

Dependencies flow strictly **downward**:

```
Hosts (WebHost, RpcHost, RealtimeHost, KafkaHost, RabbitHost, WorkerHost, BlazorHost, AspireHost)
   │
   ▼
Stories (Business logic: Commands, Queries, Handlers, Story Executors)
   │
   ▼
DataAccess (Data access: Domain services for Postgres, Mongo, Redis, S3)
   │
   ▼
Domain (Entities, DbContext, EF Configurations, Enums, Internal DTOs)

Contracts  ─ Public API / inter-service DTOs (depends on nothing)
Protos     ─ gRPC protobuf definitions (depends on nothing)
Common     ─ Cross-cutting infrastructure, helpers, extensions (used by layers above Domain)
Executor   ─ Generic story execution mechanism (used by Stories and RpcHost)
```

### Strict Dependency Rules
- **Domain** and **Contracts** must NEVER reference any other project in the solution.
- **Hosts** are thin entry points (controllers, gRPC endpoints, message loops, scheduled jobs, SignalR hubs) and must never contain business logic.
- **Hosts** must NEVER access `DataAccess` directly, bypassing `Stories` (the only exception is registering services in DI). This includes broker processors (`IProcessor`) and Quartz jobs, not only controllers: they build a command and send it to a Story. Infrastructure without business rules (outbox relay/cleanup, distributed job locks) lives in `Common` behind its own abstraction.
- **Stories** may reference `Contracts`, `Protos` (Executor Stories only), `Executor`, `Common`, `DataAccess` and `Domain`.
- **Common** may reference `Contracts`, `Protos`, `Domain`, `DataAccess` and `Documents` (DI extensions, mappers, broker infrastructure, the document renderer); it must never reference `Stories` or any host.
- **Documents** (optional Razor class library with PDF/document templates, `pdf-reports`) may reference only `Contracts`.
- **DataAccess** may reference `Domain` and `Contracts`: read methods may project straight into Contracts DTOs (`Select(x => new XxxDto { ... })`) to avoid loading entities.
- Reusable infrastructure code with zero business logic belongs in `Common`.
- These rules are enforced by `tests/Itl.<Project>.ArchitectureTests` (see `testing`); a new project reference that breaks them fails the build.

## Host-to-Stories Topology

Hosts reach Stories in one of two ways. The choice is the user's, made at project start and recorded in `AGENTS.md` («Информация о проекте») and an ADR; do not switch silently.

Before adding or changing a host, check the project decision, `.csproj` references and actual dispatch wiring. Follow the recorded topology for that host. If the decision is missing or conflicts with the wiring and the change depends on it, report the discrepancy and resolve it before changing the dispatch path.

1. **In-process (default)**: each host references `Stories`, registers the Stories mechanism (`cqrs-stories`) and `DataAccess`, and dispatches directly.
2. **Gateway**: WebHost, KafkaHost, RabbitHost and RealtimeHost dispatch through RpcHost instead of referencing `Stories`/`DataAccess` for business operations. WorkerHost can use the gateway or retain in-process Story dispatch as recorded for that host. Mandatory for gateway calls:
   - RpcHost authenticates each caller and enforces method permissions using the approved service-to-service profile in [security](../security/SKILL.md#changing-service-to-service-authorization). Use JWT or platform mTLS for their respective profiles. Preserve a project-specific exception only within the policy and ADR scope, with equivalent caller, method and delegation checks.
   - Ownership identifiers (tenant, store) sent in a gRPC request are trusted only from an authenticated service principal; RpcHost never accepts them from end-user–facing clients.
   - Errors keep their meaning across the hop: `AppExceptionInterceptor` → gRPC status → `AppExceptionHandler` → HTTP status.
   - Contract changes are made in `.proto` first, with the compatibility rules of `grpc-services`.

**Example** — A web gateway dispatches through gRPC and has no Stories or DataAccess project references. A worker dispatches through `IStoryExecutor` in-process. Verify both paths and the gateway policy recorded for service-token calls.

## Solution and Project File Formats

- **Solution: one `.slnx` file** (XML solution format) in the repository root. The legacy `.sln` format is not used: convert an existing one with `dotnet sln <name>.sln migrate`, check that `dotnet sln <name>.slnx list` shows the same projects and folders, then delete the `.sln` in the same MR. Requires .NET SDK 9.0.200+ and Visual Studio 17.13+ / Rider 2024.3+.
- **Projects: SDK-style `.csproj`** (`<Project Sdk="Microsoft.NET.Sdk...">`) with `PackageReference`. Legacy non-SDK projects and `packages.config` are not allowed. There is no `.csprojx` format: the modern project format is the SDK-style `.csproj` itself.
- New projects are added with `dotnet sln <name>.slnx add <path> --solution-folder <Folder>` so they land in the right solution folder.
- Guarded by architecture tests (`testing`): exactly one `.slnx`, no `.sln`, no `packages.config`, every `.csproj` is SDK-style and included in the solution.

## Solution Folders Structure

The structure below is the set of **solution folders inside the `.slnx`** (virtual). On disk, projects live in `src/` and `tests/`; do not move them into physical `Hosts/`, `Libs/` directories. Every project, including the frontend app, is placed into its solution folder.

```
project_root/
├── Front/        # UI applications (Blazor WebAssembly / React)
├── Hosts/        # Runnable host applications
├── Libs/         # Common, Contracts, DataAccess, Domain, Stories, Protos, Executor, Documents (optional)
├── Migrations/   # Standalone DB migration runners
├── Shared/       # Directory.Build.props, Directory.Packages.props, .editorconfig, configs
└── Tests/        # Tests, UnitTests, IntegrationTests
```

## Mandatory Elements for Every Host Application

Each runnable host (`Program.cs`) must:
1. Call `builder.AddEnvironmentConfigurations()` — loads shared configuration files from `shared/configs/*.{env}.json` and environment variables.
2. Call `builder.AddServiceDefaults()` — sets up OpenTelemetry (logs, metrics, traces), health checks, service discovery, and HttpClient resilience.
3. Call `host.MapDefaultEndpoints(port)` — exposes `/health` and `/alive` on the dedicated service port (`ServicePurpose:Port`).
4. Register typed options via `AddOptionsWithValidation<TOptions>()` (DataAnnotations validation with `ValidateOnStart`).
5. Split `Program.cs` into two clear methods: `CreateWebAppBuilder` (service registration) and `ConfigureApp` (middleware pipeline).

For library composition details, see [libraries.md](./references/libraries.md).

For several isolated clients in one deployment (tenant resolution from claims, `tenant_id` + query filters, isolation models), see [multitenancy.md](./references/multitenancy.md).

Runtime behavior of hosts in containers (probes, graceful shutdown, migrations before deploy) is covered by the `runtime-deployment` skill.

## Completion Evidence

- Record the decision, changed host and actual dispatch path. Run the project's architecture checks. Verify a gateway host has no direct business dependency on Stories/DataAccess, and an in-process host reaches DataAccess through Stories. Verify the expected path with an integration test for a changed entry point.
- For a new or changed gateway edge, verify through the configured boundary that an allowed caller reaches the method, missing identity is rejected and a valid caller without method permission is denied. Include the delegation check when the edge forwards an owner or tenant identifier.
- Record the test commands, results and any checks that could not run.
