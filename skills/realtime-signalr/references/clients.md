# Clients (React and Blazor)

## Common Rules
- The hub URL comes from runtime configuration (`/config.json` for React — `VITE_*` values are public, `react-frontend`; options for Blazor); never constructed from user input (`security`, SSRF).
- Transport: WebSockets with skipped negotiation (`scale-out-and-resilience.md`, option A) unless the project chose sticky sessions.
- One connection per app, created by a provider/service, started after login and stopped on logout; component code never opens connections.
- Every (re)connection re-subscribes explicitly: group memberships die with the old connection. The client keeps the set of followed ids and replays `SubscribeTo…` after reconnect — each call goes through the hub's authorization Story again (`hubs-and-contracts.md`).
- Retry policy: exponential backoff with jitter, bounded; when it gives up (`onclose` / `Closed`), the UI shows the realtime failure state and data keeps coming from refetch.
- Handlers are idempotent and cheap. Use cache invalidation or a dirty flag and refetch through the normal query layer. If a hint triggers a visible action, such as a toast, deduplicate it using the stable event identity across broker retries.

## Authentication Profiles
| Frontend profile | How the hub is reached |
| ---------------- | ---------------------- |
| React / Blazor with direct OIDC PKCE | the client passes the in-memory access token through `accessTokenFactory` / `AccessTokenProvider`; it lands in the `access_token` query parameter of the WebSocket URL (SKILL.md) |
| Blazor (or React) behind a **BFF** | the browser has no token. It connects to `/realtime/...` on the **BFF origin** with its session cookie; the BFF forwards the WebSocket to RealtimeHost through a reverse proxy (YARP, WebSockets enabled) and attaches the user's access token from its server-side token store as the `Authorization` header |

BFF rules (`blazor-frontend`, security.md):
- The proxy route covers only the hub paths, requires the authenticated cookie session, and the BFF keeps tokens server-side (refresh before forwarding). Proxying is infrastructure — no business logic and no hub in the BFF.
- A cookie-authenticated WebSocket is exposed to cross-site WebSocket hijacking: the BFF validates the `Origin` header on the hub path against the app origin and rejects others; antiforgery tokens do not apply to WebSockets.
- Because the token is attached server-side, `CloseOnAuthenticationExpiration` on RealtimeHost closes the connection when the forwarded token expires; the client reconnects and the BFF forwards a refreshed one.

## React (TypeScript)
Package `@microsoft/signalr` (MIT) — added when the app consumes realtime pushes (`react-frontend`, new app checklist):

```ts
// src/realtime/connection.ts
import {
  HubConnectionBuilder, HubConnectionState, HttpTransportType, LogLevel, type HubConnection, type IRetryPolicy,
} from "@microsoft/signalr";

const retryPolicy: IRetryPolicy = {
  nextRetryDelayInMilliseconds: ({ previousRetryCount }) => {
    if (previousRetryCount >= 6) return null;                  // give up → onclose → failure state
    const base = Math.min(30_000, 1_000 * 2 ** previousRetryCount);
    return base / 2 + Math.random() * (base / 2);              // exponential backoff with jitter
  },
};

export function createRealtimeConnection(baseUrl: string, getToken: () => Promise<string>): HubConnection {
  return new HubConnectionBuilder()
    .withUrl(`${baseUrl}/realtime/orders`, {
      accessTokenFactory: getToken,                            // from the oidc-client-ts user (direct OIDC profile)
      transport: HttpTransportType.WebSockets,
      skipNegotiation: true,
    })
    .withAutomaticReconnect(retryPolicy)
    .configureLogging(import.meta.env.PROD ? LogLevel.Error : LogLevel.Warning)
    .build();
}
```

Pushes are wired into **TanStack Query**, not into a parallel store:

```ts
// src/realtime/orders.ts — registered once, in the provider that owns the connection
type OrderStatusChangedDto = components["schemas"]["OrderStatusChangedDto"];   // generated, see hubs-and-contracts.md

const followed = new Set<number>();

export function wireOrders(connection: HubConnection, queryClient: QueryClient, setStatus: (s: RealtimeStatus) => void) {
  connection.on("OrderStatusChanged", (dto: OrderStatusChangedDto) => {
    queryClient.setQueryData(orderKeys.detail(dto.orderId), (old?: OrderDto) => (old ? { ...old, status: dto.status } : old));
    void queryClient.invalidateQueries({ queryKey: orderKeys.detail(dto.orderId) });
  });
  connection.onreconnecting(() => setStatus("reconnecting"));
  connection.onreconnected(async () => {
    await Promise.all([...followed].map((id) => connection.invoke("SubscribeToOrder", id)));
    void queryClient.invalidateQueries({ queryKey: orderKeys.all });   // pushes during the gap are lost
    setStatus("connected");
  });
  connection.onclose(() => setStatus("failed"));                       // retries exhausted or stopped
}

export async function followOrder(connection: HubConnection, orderId: number) {
  followed.add(orderId);
  if (connection.state === HubConnectionState.Connected) await connection.invoke("SubscribeToOrder", orderId);
}
```

- Payload types come from the generated schema; a hand-written payload type is a violation (`react-frontend`, api-and-state.md).
- Strict TS: no `any` handlers; payload narrowing before use.
- BFF profile: drop `accessTokenFactory` and use a same-origin relative URL.

## Blazor WebAssembly
Package `Microsoft.AspNetCore.SignalR.Client` (MIT) — works in WASM over the browser WebSocket. The client service lives under `Services/` (never in a component — `blazor-frontend`, components.md) and registers handlers exactly once:

```csharp
public sealed class OrdersRealtimeClient(IOptions<RealtimeOptions> options, IAccessTokenProvider tokens) : IAsyncDisposable
{
    private readonly HashSet<long> _followed = [];
    private readonly SemaphoreSlim _startGate = new(1, 1);
    private HubConnection? _connection;

    /// <summary>
    /// Изменение статуса заказа, на который подписано соединение.
    /// </summary>
    public event Action<OrderStatusChangedDto>? StatusChanged;

    /// <summary>
    /// Изменение состояния соединения.
    /// </summary>
    public event Action<HubConnectionState>? StateChanged;

    public async Task StartAsync(CancellationToken ct)
    {
        await _startGate.WaitAsync(ct);
        try
        {
            var connection = _connection ??= CreateConnection();
            if (connection.State != HubConnectionState.Disconnected) return;

            // A failed or cancelled first connection remains eligible for explicit StartAsync.
            await connection.StartAsync(ct);
            await RestoreSubscriptionsAsync(connection, ct);
            StateChanged?.Invoke(HubConnectionState.Connected);
        }
        finally
        {
            _startGate.Release();
        }
    }

    private HubConnection CreateConnection()
    {
        var connection = new HubConnectionBuilder()
            .WithUrl($"{options.Value.BaseUrl}/realtime/orders", o =>
            {
                o.Transports = HttpTransportType.WebSockets;
                o.SkipNegotiation = true;
                o.AccessTokenProvider = GetTokenAsync;           // direct OIDC profile; omit behind a BFF
            })
            .WithAutomaticReconnect(new JitterRetryPolicy())
            .Build();

        connection.On<OrderStatusChangedDto>("OrderStatusChanged", dto => StatusChanged?.Invoke(dto));
        connection.Reconnecting += _ => { StateChanged?.Invoke(HubConnectionState.Reconnecting); return Task.CompletedTask; };
        connection.Reconnected += async _ =>
        {
            await RestoreSubscriptionsAsync(connection, CancellationToken.None);
            StateChanged?.Invoke(HubConnectionState.Connected);
        };
        connection.Closed += _ => { StateChanged?.Invoke(HubConnectionState.Disconnected); return Task.CompletedTask; };

        return connection;
    }

    private async Task RestoreSubscriptionsAsync(HubConnection connection, CancellationToken ct)
    {
        foreach (var orderId in _followed.ToArray())
        {
            await connection.InvokeAsync("SubscribeToOrder", orderId, ct);
        }
    }

    public async Task FollowOrderAsync(long orderId, CancellationToken ct)
    {
        _followed.Add(orderId);
        if (_connection?.State is HubConnectionState.Connected)
        {
            await _connection.InvokeAsync("SubscribeToOrder", orderId, ct);
        }
    }

    private async Task<string?> GetTokenAsync()
    {
        var result = await tokens.RequestAccessToken();
        return result.TryGetToken(out var token) ? token.Value : null;
    }

    public async ValueTask DisposeAsync()
    {
        if (_connection is not null) await _connection.DisposeAsync();
    }
}
```

- Components subscribe to the service events in `OnInitialized`, update state through `await InvokeAsync(StateHasChanged)`, and unsubscribe in `Dispose` — a handler left attached after the component is gone is a leak.
- Behind a BFF: no `AccessTokenProvider`, `BaseUrl` is the host origin (`NavigationManager.BaseUri`); the browser sends the session cookie with the WebSocket handshake.
- Data screens keep the five data states of `blazor-frontend` (loading, empty, success, validation, failure); the connection state is a separate indicator ("live" / "reconnecting" / "offline"), and an offline connection never blocks the data shown from the API.

## Testing
- Integration: start RealtimeHost via `WebApplicationFactory` + the applicable Testcontainers dependencies (`testing`). With a real `HubConnection`, verify unauthorized connect fails, unauthorized subscribe → `HubException`, owned subscribe + event → payload received, and forced reconnect requires re-subscription. Verify a durable inbox record suppresses ordinary broker duplicates.
- Replay after a failure between send and durable completion. Deliver both hints to the client and assert no duplicate visible action and correct state after refetch.
- For the Blazor client, fail or cancel the initial connection, restore the network and call `StartAsync` again. Verify subscription succeeds and one subsequent push invokes the handler once. Automatic reconnect starts only after a successful initial connection.
- Unit: client handlers are tested with a fake connection (an interface over `.On` registrations), never a live socket.
- E2E: run the critical journey against the real hub in the test environment. Assert an update caused by a verified hub event with the fallback held off for that assertion. Separately drop the hint or connection and assert that refetch restores the authoritative state.
