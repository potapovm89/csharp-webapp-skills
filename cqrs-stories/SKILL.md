---
name: cqrs-stories
description: >-
  Use this skill when implementing or modifying business logic in the Stories layer: the solution uses exactly one mechanism — MediatR (Commands, Queries, Handlers) or Executor Stories (IStory / IStoryExecutor) — plus FluentValidation validators.
  Triggers (RU): бизнес-логика, команда, запрос, хендлер, MediatR, IStoryExecutor, стори, валидатор.
---

# CQRS in Stories

All business logic lives in `Itl.<Project>.Stories`. Hosts (Controllers, gRPC services, Consumers) never contain business rules — they strictly dispatch commands and queries to Stories.

## Execution Mechanism: exactly one per solution

A solution uses **either** MediatR **or** Executor Stories for all of its business logic — never both. The choice is made once, at project start, by the user (not by the agent) and recorded in `AGENTS.md` → «Информация о проекте» (`Механизм Stories: MediatR | IStoryExecutor`).

A template or reference repository may demonstrate both mechanisms in separate hosts (no host registers both). That exception is recorded in that repository's own ADR and architecture tests and never carries over to a solution created from it.

| | MediatR (≤ 12.5.0) | Executor Stories (`IStory` + `IStoryExecutor`) |
|---|---|---|
| Unit of logic | `Command`/`Query` + `Handler` + `Validator` | `Story` + `StoryContext` + `Validator` |
| Folder | `Stories/Commands`, `Stories/Queries` | `Stories/ExecutorStories` |
| Host dispatch | `IMediator.Send(...)` / `ISender` | `IStoryExecutor.Execute(context, ct)` |
| Input models | DTOs from `Contracts` | DTOs from `Contracts` or proto messages from `Protos` |

Rules:
- Every host (WebHost, RpcHost, RealtimeHost, WorkerHost, KafkaHost, RabbitHost, BlazorHost) dispatches through the chosen mechanism. Adding the other one "for this host only" is prohibited.
- If `AGENTS.md` does not state the mechanism, look at the code (`AddMediatR` vs `AddExecutors` in hosts). If the code has neither or both, **stop and ask the user** — do not pick one yourself.
- The template may ship with both registered. When the user chose one, remove the other mechanism's registrations, packages and sample code in the same change, and say so in the MR.
- Switching the mechanism of an existing solution is an architectural change: ADR in `docs/adr/` and explicit user approval.
- Whether hosts call Stories **in-process** (host references `Stories`) or **through RpcHost over gRPC** (host is a gateway) is a separate decision; see `solution-architecture` → «Host-to-Stories topology».

## Project Structure

Only the folders of the chosen mechanism exist in a solution.

```
Stories/
├── IStoryLibrary.cs        # Marker interface for DI scanning
├── Commands/               # MediatR only: modifies state
│   └── <Entity>/
│       ├── <Action><Entity>Command.cs
│       ├── <Action><Entity>CommandHandler.cs
│       └── <Action><Entity>CommandValidator.cs
├── Queries/                # MediatR only: reads state
│   └── <Entity>/
│       ├── <Action><Entity>Query.cs
│       └── <Action><Entity>QueryHandler.cs
└── ExecutorStories/        # Executor only
    └── <Domain>/<Action>/
        ├── <Action>Story.cs
        ├── <Action>StoryContext.cs
        └── <Action>Validator.cs
```

One type per file: a Story, its Context and its Validator are three files even when each is short. Shared helpers of an area go to `ExecutorStories/<Domain>/` (validators of nested DTOs) or `Stories/Common/` (cross-area); mapping methods do not live in Stories (see `csharp-code-style`, Object Mapping).

## Implementing a MediatR Command

```csharp
[DataContract]
public class CreatePaymentCommand : IRequest<string>
{
    /// <summary>
    /// Идентификатор магазина.
    /// </summary>
    [DataMember]
    public string? StoreId { get; set; }

    /// <summary>
    /// Запрос на проведение оплаты.
    /// </summary>
    [DataMember]
    public required PaymentRequestDto Data { get; set; }
}

public class CreatePaymentCommandHandler(
    IValidator<CreatePaymentCommand> validator,
    IPaymentDomainService paymentDomainService,
    ITransactionScope transactionScope) : IRequestHandler<CreatePaymentCommand, string>
{
    public async Task<string> Handle(CreatePaymentCommand request, CancellationToken ct)
    {
        // 1. Mandatory validation at handler entry
        await validator.ValidateAndThrowAsync(request, ct);

        // 2. Business logic & domain service operations in one transaction
        string paymentId;
        await using (await transactionScope.BeginAsync(ct))
        {
            try
            {
                paymentId = await paymentDomainService.CreateAsync(request.StoreId!, request.Data, ct);
                await transactionScope.CommitAsync(ct);
            }
            catch
            {
                await transactionScope.RollbackAsync(CancellationToken.None); // a cancelled ct must not skip the rollback
                throw;
            }
        }

        return paymentId;
    }
}
```

### Command Guidelines
- Commands are POCOs decorated with `[DataContract]` / `[DataMember]` and implement `IRequest<TResult>`.
- Every property must have XML documentation ending with a period.
- Handlers implement `IRequestHandler<TCommand, TResult>`.
- Handlers receive dependencies via primary constructor and MUST execute `await validator.ValidateAndThrowAsync(request, ct)` as their first action.
- Multi-entity changes go through `ITransactionScope` with explicit `CommitAsync` and `RollbackAsync` in `catch` (same pattern as in `postgresql-data-access`). `BeginAsync` returns `IAsyncDisposable`: always `await using (await transactionScope.BeginAsync(ct))`.
- A handler that changes data and must notify other services writes an outbox row in the same transaction; it never publishes to Kafka/RabbitMQ directly (see `message-brokers`, outbox.md).
- Validators inherit from `AbstractValidator<TCommand>` (FluentValidation) and reside next to the command. Nested DTOs are validated using `.SetValidator()`.

## Implementing an Executor Story

```csharp
// CreateAuditRecordStoryContext.cs
public class CreateAuditRecordStoryContext : IResult<Guid>
{
    /// <summary>
    /// Запрос на создание записи аудита.
    /// </summary>
    public required CreateRecordRequest Request { get; init; }
}

// CreateAuditRecordStory.cs
public class CreateAuditRecordStory(
    IValidator<CreateAuditRecordStoryContext> validator,
    IAuditDomainService auditDomainService,
    ITransactionScope transactionScope,
    TimeProvider timeProvider) : IStory<CreateAuditRecordStoryContext, Guid>
{
    /// <inheritdoc />
    public async Task<Guid> ExecuteAsync(CreateAuditRecordStoryContext context, CancellationToken ct)
    {
        await validator.ValidateAndThrowAsync(context, ct);

        var utcNow = timeProvider.GetUtcNow().UtcDateTime;
        // ... business logic in one transaction (ITransactionScope) ...
        return recordId;
    }
}

// Host side
var id = await storyExecutor.Execute(new CreateAuditRecordStoryContext { Request = request }, ct);
```

The `CancellationToken` is a parameter of `ExecuteAsync` / `IStoryExecutor.Execute`, not a property of the context. Older copies of the template keep it in the context (`ExecuteAsync(context)`); the solution's `Executor` project is the source of truth — read `IExecution.cs` before writing a story.

### Executor Story Guidelines
- The validator runs first; it validates the whole context (`IValidator<TContext>`), nested proto/DTO validators are attached with `.SetValidator()`.
- Multi-entity changes go through `ITransactionScope`, same rules as for MediatR handlers above; notifications to other services go through the outbox.
- Current time comes from the injected `TimeProvider`, never `DateTime.UtcNow` (see `csharp-code-style`).
- Ownership identifiers in the context (tenant, store, customer) are filled by the host from claims (or from the authenticated caller in RpcHost — see `security`, Service-to-Service); the story filters every read and write by them.
- `CreateRecordRequest` above is a protobuf message from `Itl.<Project>.Protos`. Executor Stories are allowed to reference `Protos` and accept proto messages in their context: like `Contracts`, `Protos` depends on nothing, and duplicating every gRPC request as a DTO adds no value.
- MediatR Commands/Queries use DTOs from `Contracts`, never proto messages.
- Stories must not reference any `Hosts` project or `Grpc.AspNetCore` server types (`ServerCallContext`); pass only the message and `CancellationToken`.

## Dependency Injection Setup

Register validators plus **one** of the two blocks below, in every host that runs Stories in-process.

```csharp
builder.Services.AddValidatorsFromAssemblyContaining<IStoryLibrary>();

// Variant A — MediatR
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<IStoryLibrary>());

// Variant B — Executor Stories
builder.Services.AddExecutors(options =>
{
    options.ExecutionTypes = [typeof(IStory<>), typeof(IStory<,>)];
    options.ExecutorInterfaceType = typeof(IStoryExecutor);
    options.ExecutorImplementationType = typeof(StoryExecutor);
}, typeof(IStoryLibrary));
```
