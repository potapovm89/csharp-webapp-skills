---
name: api-clients
description: >-
  Use this skill when integrating external HTTP/REST APIs or external gRPC services, configuring typed HttpClients, resilience policies (Polly), or gRPC client factories.
  Triggers (RU): внешний API, HttpClient, интеграция, вызов стороннего сервиса, gRPC-клиент, ретраи, таймауты.
---

# API Clients Integration (REST & gRPC)

This skill covers invoking external REST and gRPC services safely and idiomatically.

## REST API Clients

For a new or changed HTTP call, check its effective retry policy before editing:

1. Find the typed client registration and `AddServiceDefaults` / `ConfigureHttpClientDefaults`. Check for client-specific overrides and caller or job retries around the same operation.
2. If the operation changes state, keep automatic retries disabled unless the receiving system guarantees deduplication for that operation. If the result is unknown and deduplication is unavailable, reconcile the outcome before resending. For a supported retry, reuse the same key and payload across attempts and keep one retry owner within the caller's time budget.
3. Verify the policy through the client built by the real DI registration. If the shared default needs changing, check its impact on other clients and report any correction outside the task's scope.

**Example from `it-letter`** — `WebhookDispatcher` sends a POST through a typed client that inherits `AddServiceDefaults`. The presence of an HTTP client registration alone cannot show whether the POST is retried. Inspect the shared resilience handler and test that registered client with a fake webhook receiver.

- **Location**: `Itl.<Project>.Common/Services/<ServiceName>/`
- **Pattern**: Interface `I<Service>Client` + implementation `<Service>Client(HttpClient client)`.
- **Registration**:
  - Never instantiate `new HttpClient()`.
  - Base addresses come only from options; never call a URL built from user input (SSRF, see `security`).
  - Use `AddHttpClient<TInterface, TImplementation>` with typed options inheriting `BaseUrlOptions`.
  - `builder.AddServiceDefaults()` registers one resilience handler per client through shared defaults. Verify its configuration: the standard handler retries all HTTP methods by default. Shared defaults must disable retries for unsafe methods; enable retries for a write only when its end-to-end idempotency is documented and tested.

Configure the shared defaults **once inside `AddServiceDefaults`**, replacing its existing unconfigured handler registration rather than stacking a second handler:

```csharp
services.ConfigureHttpClientDefaults(http =>
    http.AddStandardResilienceHandler(options =>
        options.Retry.DisableForUnsafeHttpMethods()));
```

This conservative default disables retries for POST/PATCH/PUT/DELETE/CONNECT. A client-specific override replaces the inherited handler and keeps one retry layer; it must preserve the same idempotency guarantees. See [Microsoft HTTP resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience?tabs=package-reference).

```csharp
public static IServiceCollection AddExchangeRatesClient(
    this IServiceCollection services,
    Action<ExchangeRatesClientOptions> configureOptions)
{
    services.AddOptionsWithValidation(configureOptions);

    services
        .AddHttpClient<IExchangeRatesClient, ExchangeRatesClient>((sp, client) =>
        {
            var options = sp.GetRequiredService<IOptions<ExchangeRatesClientOptions>>().Value;
            client.BaseAddress = new Uri(options.Address);
        })
        .SetHandlerLifetime(TimeSpan.FromMinutes(5));

    return services;
}
```

- **Rules**:
  - Validate parameters before executing requests (`ArgumentException`).
  - Always pass `CancellationToken`.
  - Bound total call duration (including attempts and backoff) by the caller's remaining budget; honor `Retry-After` within that budget. Do not retry caller cancellation or permanent errors, and do not multiply retries across client, gateway and job layers.
  - A retried write reuses the same idempotency key and payload for every attempt; test a server commit followed by a lost response. Availability of a retry handler alone does not make an operation safe to repeat.
  - Do not catch and swallow HTTP exceptions locally: let `AppExceptionHandler` translate errors.
  - Never log sensitive payload bodies in error logs.
  - Document external APIs in `docs/contracts/api-clients/<client>.md`.

## gRPC Clients

- **Contracts**: Defined in `Itl.<Project>.Protos`.
- **Client Generation**: Generated via `Grpc.Tools` / `Grpc.Net.ClientFactory`.
- **Registration**: Use helper `AddGrpcService<TClient, TOptions>` from `Common`:
  ```csharp
  builder.Services.AddGrpcService<AuditService.AuditServiceClient, AuditRpcOptions>(
      builder.Configuration.GetRequiredSection(nameof(AuditRpcOptions)).Bind);
  ```
- **Rules**:
  - Never instantiate raw `GrpcChannel.ForAddress(...)`.
  - Propagate `CancellationToken` on every gRPC call.
  - Set a deadline per call or through the existing client factory; retries are bounded by that deadline and enabled only for idempotent methods (see `grpc-services`, protobuf.md).
  - Do not swallow `RpcException`; let global exception handlers map status codes (`NotFound` -> 404, `InvalidArgument` -> 400).
  - Map models via manual extension methods in `Common/Mappers` or partial proto model conversion operators. AutoMapper is prohibited (see `csharp-code-style`).

## Completion Evidence

- For a changed HTTP write, use a local test receiver that records the effect and then drops the first response. With write retries disabled, assert one request and one effect. When a retry is explicitly supported, make the next attempt succeed and assert that both requests carry the same idempotency key and payload and produce one effect.
- When the change enables read retries, fail the first read with a transient error and assert that the next attempt succeeds within the configured budget. When changing gRPC retries, check the effective method policy and verify that a transient failure cannot retry a non-idempotent method or outlive its deadline.
- Record the test command, observed attempt and effect counts for the applicable cases, and any check that could not run.
