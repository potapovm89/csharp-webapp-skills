# Hubs, Groups and the Push Pipeline

## Hub Structure
Hubs are thin adapters, exactly like controllers and gRPC services (`cqrs-stories`). The example uses Executor Stories; with MediatR the hub injects `ISender` and sends a query — one mechanism per solution.

```csharp
[Authorize(Policy = "OrdersRealtime")]
public sealed class OrdersHub(IStoryExecutor storyExecutor) : Hub
{
    /// <summary>
    /// Подписка текущего соединения на события заказа.
    /// </summary>
    public async Task SubscribeToOrder(long orderId)
    {
        // The Story decides; the client only nominates an id. Throws ForbiddenException / NotFoundException.
        await storyExecutor.Execute(new EnsureCanFollowOrderStoryContext
        {
            RequesterId = Context.UserIdentifier!,       // from claims, never from the client payload
            OrderId = orderId
        }, Context.ConnectionAborted);

        await Groups.AddToGroupAsync(Context.ConnectionId, GroupNames.Order(orderId), Context.ConnectionAborted);
    }

    /// <summary>
    /// Отписка текущего соединения от событий заказа.
    /// </summary>
    public Task UnsubscribeFromOrder(long orderId) =>
        Groups.RemoveFromGroupAsync(Context.ConnectionId, GroupNames.Order(orderId), Context.ConnectionAborted);
}
```

- The Story is a read-only authorization query: it loads the entity, checks ownership/tenant from `RequesterId` and throws on denial. Stories never touch SignalR types (`IHubContext`, `IGroupManager`) — the hub adds the connection to the group after the Story returns. This keeps the dependency direction of `solution-architecture`.
- Group membership is ephemeral connection state: SignalR removes the connection from all groups when it disconnects, and the backplane tracks membership per server. There is no subscription table and no cleanup job; after a reconnect the client subscribes again (`clients.md`).
- Use `Context.ConnectionAborted` for cancellation inside hub methods.
- Hub methods validate input (validators, `cqrs-stories`), return nothing except ack/`HubException`, and are covered by the same exception mapping as other hosts: an unexpected exception reaches the client as a generic error (`EnableDetailedErrors = false`); `HubException` messages are client-safe — no internals, no stack traces (`grpc-services`, interceptors.md — same rule).
- In the gateway topology the hub calls RpcHost for the authorization check (`api-clients`) instead of a local Story.
- Streaming (`IAsyncEnumerable` / `ChannelReader`) is allowed only for bounded server-side streams (e.g. progress of a report job, `pdf-reports`); unbounded streams to every client are prohibited.

## Addressing: Users and Groups
| Target | How | Added when |
| ------ | --- | ---------- |
| The user's own connections | `Clients.User(userId)` | automatically — no group needed |
| Entity followers | group `order_<orderId>` | after the authorization Story succeeds (`SubscribeToOrder`) |
| Tenant-wide | group `tenant_<tenantId>` | on subscribe, after the Story verifies membership; tenant id from claims (`solution-architecture`, multitenancy.md) |

- `Clients.User` resolves `Context.UserIdentifier` through `IUserIdProvider` — by default the `ClaimTypes.NameIdentifier` claim, which JwtBearer maps from `sub` while `MapInboundClaims` is on. If inbound mapping is disabled, register an `IUserIdProvider` that reads `sub`. Either way it works across replicas through the backplane.
- Group names come from one static helper class (`GroupNames`), snake_case prefixes plus validated ids only; never free user input. Client-nominated group names are never trusted — the hub builds the name from the id after the check.
- Membership is checked once, at subscribe time. When access can be revoked while connected (user removed from a tenant), the revoking Story publishes an event and the RealtimeHost removes that user's connections from the group, or the policy is documented as "until reconnect" with the token lifetime as the bound.

## Contracts and Client Generation
- Payload DTOs live in `Itl.<Project>.Contracts/Realtime/` with XML docs, like REST DTOs (`domain-modeling`, DTO rules).
- Frontends generate types from the OpenAPI document (`react-frontend`, api-and-state.md). Hub payloads are not reachable from any endpoint, so they would be missing from the document. Register them explicitly: a document transformer in the host that publishes the frontend's document adds each realtime DTO to `components.schemas` (`var schema = await context.GetOrCreateSchemaAsync(typeof(OrderStatusChangedDto), null, ct); document.AddComponent(nameof(OrderStatusChangedDto), schema);` — built-in `Microsoft.AspNetCore.OpenApi` 10). The contract snapshot test then covers them too (`web-host-api`, swagger.md).
- Blazor references the Contracts project directly — no generation needed.
- `docs/contracts/realtime/<hub>.md` lists per hub: path, policy, client→server methods, server→client message names with payload DTO and version, groups and the event that produces each message (`project-documentation`).

## Push Pipeline (outbox → broker → RealtimeHost)

```
Story: business change + outbox row (one transaction)
  → relay publishes "order-status-changed" (message-brokers)
    → RealtimeHost IProcessor consumes
      → dispatches a push Story/handler: dedup + mapping to the Contracts DTO (Common/Mappers)
        → IHubContext<OrdersHub>.Clients.Group(GroupNames.Order(id))
              .SendAsync("OrderStatusChanged", dto, ct);
```

- The send through `IHubContext` is transport, so it lives in the RealtimeHost behind a small `IRealtimePublisher` abstraction defined in `Common`; the push handler calls the abstraction, never `IHubContext` directly from Stories (`solution-architecture`).
- The consumer is idempotent (inbox, `message-brokers`): a duplicate event must not push twice — clients get at-most-once anyway, but double pushes cause UI flicker and duplicate toasts.
- The processor **throws on failure** like any other processor; the event goes to retry/DLQ. A failed push does not lose business state — the client refetches.
- Payload versioning follows `message-brokers`: additive changes keep `message-version`; unknown fields are ignored; unsupported versions go to the DLQ, not to clients.

## Limits and Backpressure
- Inbound: `MaximumReceiveMessageSize` (default 32 KB, lowered in the host minimum) caps client payloads; file uploads go through REST + S3 (`s3-storage`), never through the hub.
- `MaximumParallelInvocationsPerClient` defaults to 1 — keep it; raising it allows one client to occupy several server threads.
- Connection cap: Kestrel `Limits.MaxConcurrentUpgradedConnections` per instance (WebSocket connections are upgraded connections), sized by the soak test; SignalR itself has no connection limit.
- Outbound: DTOs stay small (ids, status, short text). A payload that can exceed ~16 KiB is a design smell — send the id and let the client refetch.
- Per-user send rate is capped in the push handler (e.g. a counter in Redis with TTL, `nosql-and-cache`); bursts collapse into "dirty flag" pushes instead of one message per change.

## Observability
ASP.NET Core already emits SignalR connection metrics; add only the push-specific ones:

| Instrument | Source | Tags |
| ---------- | ------ | ---- |
| `signalr.server.active_connections` | built-in (`Microsoft.AspNetCore.Http.Connections` meter) | transport, status |
| `signalr.server.connection.duration` | built-in | transport, status |
| `realtime.messages.pushed` | custom counter | hub, message_type |
| `realtime.push.duration` | custom histogram (s) | hub, message_type |

- Active connections close to `MaxConcurrentUpgradedConnections` is an alert; push-duration p99 growth usually means fan-out amplification (see scale-out-and-resilience.md).
- The consuming processor continues the producer's trace from `traceparent` (`message-brokers`); custom instruments use the host's meter (`observability`, metrics.md).
