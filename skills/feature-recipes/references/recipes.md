# Recipes

`<Area>` is the business area (e.g. `Payments`), `<Entity>` the aggregate, `<Project>` the solution prefix.

Names below are given for both Stories mechanisms: **MediatR** (`Command`/`Handler`) and **Executor** (`StoryContext`/`Story`). Use only the one recorded in `AGENTS.md` (`cqrs-stories`). In the gateway topology (`solution-architecture`) the host step calls RpcHost over gRPC, and the Story lives behind the gRPC method (recipe 6).

## 1. REST Endpoint: Write to the DB + Publish an Event
For a read-only endpoint, use the applicable contract, query, controller, test and documentation steps. For a write without an event, keep the required transaction and omit the outbox and broker steps.

1. **Contract** — request/response DTOs in `Contracts/<Area>/` with XML docs; check compatibility rules (`web-host-api`, api-evolution.md).
2. **Request object** — MediatR: `Stories/Commands/<Area>/<Action>Command` (`IRequest<TResult>`); Executor: `Stories/ExecutorStories/<Area>/<Action>/<Action>StoryContext` (`IResult<TResult>`). It carries the DTO and the owner id (filled from claims, not from the client).
3. **Validator** — `<Action>CommandValidator` / `<Action>Validator` : `AbstractValidator<...>`, own file; table-driven unit tests (`cqrs-stories`, `testing`).
4. **Mapping** — `To<Target>()` extension methods in `Common/Mappers/<Area>MappingExtensions.cs` + unit tests.
5. **Handler / Story** — validate → (optional friendly pre-check) → one transaction: domain service `Create/Update` + `IOutboxDomainService.Add(OutboxMessageFactory.Create(...))` → commit; map unique violation `23505` to a domain error; nothing is published directly (`message-brokers`, outbox.md; `postgresql-data-access`, concurrency.md).
6. **Message contract** — message type constant in kebab-case, payload versioned (`messageVersion`), documented in `docs/contracts/<broker>/`.
7. **Controller** — thin action in `WebHost/Controllers`: `[Authorize(Policy = ...)]`, owner id from `User` claims, `[ProducesResponseType]` for every status, `CancellationToken` passed through (`web-host-api`, `security`).
8. **Tests** — integration: 201/200 happy path, 400 validation, 401 without token, 403 wrong policy/claim, 404/409 cases; DB row and outbox row written in one transaction; duplicate request does not create a second row.
9. **Docs** — `docs/contracts/api/<controller>.md`, OpenAPI snapshot, broker contract.

## 2. Broker Consumer (Kafka / RabbitMQ)
1. **Message contract** — reuse the producer's documented contract; ignore unknown optional fields within a supported version. An unsupported `message-version` is a poison message sent to the DLQ with an alert; support for a newer version must be implemented and tested explicitly.
2. **Processor** — `IProcessor<TKey, TMessage>` implementation in the consumer host; it calls a Story (in-process) or RpcHost (gateway), no business logic in the host. It never swallows a failure: a malformed payload throws `PoisonMessageException`, anything else rethrows — logging and returning loses the message.
3. **Idempotency** — in the Story transaction: `IInboxDomainService.TryRegisterAsync(consumer, messageId, ct)`; `false` → already processed, return. Or rely on a natural unique key.
4. **Failure handling** — transient errors retried with bounded backoff; validation/poison errors throw `PoisonMessageException` → straight to DLQ. Kafka stores the offset only after processing or confirmed forwarding to retry/DLQ; RabbitMQ acknowledges after processing/confirmed forwarding, or nacks without requeue to a configured DLX (`message-brokers`, kafka.md / rabbitmq.md / handlers.md).
5. **Tracing & metrics** — extract trace context from headers (`MessageTracing`), consumer metrics recorded (`observability`).
6. **Configuration** — topic/queue, consumer group, DLQ names in options and Aspire resources (`aspire-environment`).
7. **Tests** — processed once on duplicate delivery; poison message lands in the DLQ; transient failure retried then succeeds.
8. **Docs** — consumer listed in the broker contract doc; DLQ replay runbook covers the new DLQ.

## 3. Scheduled Job on Several Replicas
1. **Job class** — Quartz `IJob` in `WorkerHost/Jobs`. Dispatch business work through the chosen Stories mechanism. The Story accesses domain services. The job may use `Common` infrastructure abstractions for coordination. Apply `[DisallowConcurrentExecution]` where the documented scheduling strategy needs it (`worker-host-jobs`).
2. **Cluster safety** — use this WorkerHost's documented choice: one replica, clustered ADO.NET store, `IDistributedJobLock.TryAcquireAsync(JobLockKeys.X, ct)` with a unique key (`null` → another replica runs it, exit), or row-level SKIP LOCKED distribution (clustering.md). A local `[DisallowConcurrentExecution]` alone does not coordinate replicas.
3. **Batching** — implement batching and transaction boundaries in Stories and DataAccess. For row-level distribution, the domain service claims rows with `FOR UPDATE SKIP LOCKED`. A concurrency conflict on one item skips it, not the whole run.
4. **Schedule** — 7-field cron in options; explicit time zone; misfire policy chosen.
5. **Shutdown** — check the cancellation token between items.
6. **Tests** — job processes eligible rows only; leader/lock approaches exclude a concurrent run, while SKIP LOCKED workers claim disjoint batches; concurrency conflicts are skipped and recovery/retries do not duplicate business effects.
7. **Observability** — log start/finish with counts; metric or alert if the job stops running.

## 4. New Entity + Migration
1. **Entity** — `Domain/Entities/<Entity>.cs` on the right base class/mixins (`domain-modeling`); `[Timestamp] uint Version` if it is updated after creation.
2. **Configuration** — `IEntityTypeConfiguration<<Entity>>`: table/column snake_case, max lengths, indexes (unique with `is_deleted = FALSE` filter), FKs.
3. **Domain service** — `I<Entity>DomainService` + implementation in DataAccess (EF or Dapper), registered in the area's DataAccess extension (`postgresql-data-access`).
4. **Migration** — generated, reviewed SQL (`dotnet ef migrations script`), zero-downtime rules: additive first, indexes `CONCURRENTLY`, backfill outside the migration, contract step in a later release (`database-migrations`).
5. **Tests** — integration test of the domain service against a Testcontainers database; migration applies on an empty DB in CI.
6. **Docs** — `docs/database/postgresql/<dbname>/schema.dbml`.

## 5. External REST API Integration
1. **Options** — `<Service>ClientOptions` (base URL, timeouts) validated on start; secrets from configuration only.
2. **Typed client** — interface in `Common/Services/<ServiceName>/` per `api-clients`, `AddHttpClient<I..., ...>()` with standard resilience (timeouts, retries only for idempotent calls, circuit breaker).
3. **Contracts** — external DTOs stay inside the client; map to internal DTOs at the boundary.
4. **Errors** — non-success responses mapped to domain exceptions; no external error bodies leak to our API.
5. **Security** — outgoing URL from configuration, never from user input (SSRF) (`security`, input-and-data.md).
6. **Tests** — unit tests with a fake `HttpMessageHandler`; integration tests with WireMock.Net (Apache-2.0) for timeouts, 4xx, 5xx.
7. **Docs** — `docs/contracts/api-clients/<service>.md`.

## 6. New gRPC Method
1. **Proto** — method + `<Method>Request/Response` in `Protos/Services/<name>_service.proto`; only additive changes in an existing package version (`grpc-services`, protobuf.md).
2. **Service** — RpcHost service method delegates to a Story through the solution's single mechanism (`IStoryExecutor` or MediatR `ISender` — see `cqrs-stories`); maps proto ↔ DTO with extension methods from `Common/Mappers`; null results handled explicitly. In the gateway topology the caller's identity (service principal) is checked by RpcHost authorization (`security`, Service-to-Service).
3. **Errors** — domain exceptions mapped to status codes by the interceptor; no stack traces in status details.
4. **Clients** — set deadlines; retries only for idempotent methods.
5. **Tests** — integration test through the generated client against the test host; contract compatibility check (`buf breaking` or descriptor snapshot).
6. **Docs** — `docs/contracts/gRPC/<service>.md`.

## 7. Release Pipeline and Artifact Promotion
1. **Contract** — inspect the actual CI provider, runner/credential boundaries, required gates and the project's release authorization (`delivery-and-supply-chain`).
2. **Inputs/gates** — pin the toolchain and reviewed dependency graph; wire tests, security/license policy and scanner failures into explicit blocking stage dependencies.
3. **Candidate** — build once per host, record digest/manifest, SBOM and required provenance/authenticity evidence; promotion verifies the same candidate instead of rebuilding.
4. **Acceptance** — test gate rejection with isolated fixtures; in an authorized test environment, verify migration-once, rollout/SLI observation, hold/rollback conditions (`runtime-deployment`, `reliability-and-performance`). Never use production to validate a pipeline edit.
5. **Docs** — `docs/delivery/pipeline.md`, applicable ADRs, rollback runbook and sanitized release-evidence links.

## 8. Persistent State and Recovery
1. **Scope** — identify authoritative stores, keys, external effects and ownership; reuse or obtain the applicable RPO/RTO and retention decisions (`data-recovery`).
2. **Implementation** — coordinate schema/storage changes with backup/PITR coverage, object versions and message/journal replay horizons; schema rollback and data restoration are different decisions.
3. **Verification** — an authorized isolated drill restores to a new target and tests integrity, business reconciliation and measured loss/time. Select fixture versus realistic-volume validation by risk; label what the evidence proves.
4. **Docs** — update `docs/recovery/plan.md`, restore/reconciliation runbook and drill evidence; preserve source data and prohibit production egress from restored workloads.

## 9. Critical Journey Reliability
1. **Definition** — owner, eligible/good/bad events, measurement boundary, target/window, load assumptions and failure behavior (`reliability-and-performance`).
2. **Instrumentation** — implement the counters/histograms and missing-data signals with `observability`; validate denominator/aggregation on controlled fixtures.
3. **Acceptance** — run relevant bounded load, overload and dependency-failure scenarios in the approved non-production environment; check correct completion, not just response speed.
4. **Operations** — connect burn-rate/operational alerts to runbooks and define rollout observation/hold criteria consumed by `delivery-and-supply-chain`.
5. **Docs** — `docs/reliability/<journey>.md`, test evidence and relevant runbooks; link data-loss/restore objectives to `data-recovery` where applicable.

## 10. Frontend Feature (React / Blazor)
1. **Contract** — request/response DTOs in `Contracts/<Area>/`, controller and policy decided (`web-host-api`, `security`); ownership identifiers stay server-side from claims. OpenAPI snapshot updated in the same MR (`web-host-api`, swagger.md).
2. **Client** — regenerate the typed client: `openapi-typescript`/`orval` for React; a domain client service backed by `IHttpRepository` for Blazor. Hand-written DTO types are prohibited (`react-frontend`, api-and-state.md; `blazor-frontend`, components.md).
3. **State** — TanStack Query (query-key factories, `signal`, mutations with Idempotency-Key) for React; client services + the five data states (loading/empty/success/validation/failure) for Blazor.
4. **Forms** — react-hook-form + zod with server ProblemDetails mapped to fields; Blazor: `EditForm` + `ValidationMessageStore` (`blazor-frontend`, components.md). Client validation is UX; business rules stay in Stories (`cqrs-stories`).
5. **i18n** — every visible string through i18next / IStringLocalizer; dates and numbers via Intl/culture; check the longest locale for truncation (`react-frontend`, `blazor-frontend`, localization.md).
6. **Realtime (if applicable)** — subscribe through the RealtimeHost hub, pushes invalidate or patch the query cache, re-subscribe on reconnect; a lost push must not break correctness (`realtime-signalr`, clients.md).
7. **Tests** — Vitest + Testing Library + MSW, Playwright for the critical path, axe smoke; bUnit for Blazor state transitions/forms (`react-frontend`/`blazor-frontend`, testing-and-quality.md).
8. **Docs** — `docs/contracts/api/<controller>.md` updated for consumed endpoints; the UI feature list in `docs/description.md` only if it changes the product picture.
