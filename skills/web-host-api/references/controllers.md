# WebHost Controllers Reference

## Controller Partitioning

| Section | Location | Route Prefix | Swagger Group | Auth Scheme |
| ------- | -------- | ------------ | ------------- | ----------- |
| **User API** | `Controllers/User` | `api/v1/[controller]` | `ControllerSections.USER_API` | JWT Bearer (`UserAuthOptions`) |
| **Admin API** | `Controllers/Admin` | `api/v1/admin/[controller]` | `ControllerSections.ADMIN_API` | Keycloak role (`AuthenticationRoles.Admin`) |

Apply these prefixes to new versioned APIs. Existing published routes are preserved until an explicitly approved compatibility/deprecation migration (`api-evolution.md`).

## Standard Controller Structure
```csharp
/// <summary>
/// Контроллер для работы с заказами.
/// </summary>
[ApiController]
[KebabCaseNaming]
[Route("api/v1/[controller]")]
[ApiExplorerSettings(GroupName = ControllerSections.USER_API)]
public class OrderController(IStoryExecutor storyExecutor) : ControllerBase
{
    /// <summary>
    /// Создание заказа.
    /// </summary>
    [HttpPost]
    public async Task<IActionResult> CreateOrder(
        [FromHeader(Name = "Idempotency-Key")] string idempotencyKey,
        [FromBody] CreateOrderRequestDto request,
        CancellationToken ct)
    {
        var result = await storyExecutor.Execute(
            new CreateOrderStoryContext { IdempotencyKey = idempotencyKey, Data = request, StoreId = User.GetStoreId() },
            ct);
        return Created($"/api/v1/order/{result}", result);
    }
}
```

The dispatch line depends on the solution's Stories mechanism and topology (`cqrs-stories`, `solution-architecture`):
- MediatR: `await mediator.Send(new CreateOrderCommand { ... }, ct)` with `IMediator mediator` injected.
- Executor Stories: as above, with `IStoryExecutor storyExecutor` injected.
- Gateway topology: the controller calls the generated gRPC client (`OrderService.OrderServiceClient`) and maps the response with `Common/Mappers` extension methods; it still contains no business logic.

## Controller Rules
- Decorate with `[ApiController]` and `[KebabCaseNaming]`.
- Explicit versioned route attributes (`Route("api/v1/...")` or `Route("api/v1/admin/...")`) for new APIs; `Location` links use the same public route/version.
- All actions take `CancellationToken ct` as their last parameter.
- Long-running operations (report generation, imports) are asynchronous: the action dispatches a Story that records the request and returns `202 Accepted` with the operation id and a `Location` header pointing to the status resource; the client polls it or receives a realtime push (`pdf-reports`, `realtime-signalr`). Never hold the HTTP request open for minutes.
- Action parameters explicitly declare their sources (`[FromBody]`, `[FromRoute]`, `[FromQuery]`, `[FromHeader]`).
- Do not perform validation in controllers; controllers only assemble the command / story context / gRPC request and dispatch it.
- Ownership identifiers (store, customer, tenant) are read from the authenticated user's claims and put into the command; never accept them from an end-user header, query or body (see `security`).
- Anonymous endpoints are marked `[AllowAnonymous]` with a comment explaining why; everything else is covered by the fallback authorization policy.
