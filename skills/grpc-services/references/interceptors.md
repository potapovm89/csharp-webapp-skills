# gRPC Interceptors Reference

## Registration & Order
Interceptors are registered globally in `AddGrpc` (first registered = outermost, it wraps everything registered after it) or per service via `.Intercept(...)`. Order: metrics, then logging, then `AppExceptionInterceptor` **last** (innermost). The exception interceptor maps every exception to an `RpcException` before the observing interceptors see it, so logging and metric counters record the mapped status code. With `AppExceptionInterceptor` registered first they would see the raw exception instead.

If an approved profile uses a caller-auth interceptor that emits a gRPC denial status, register it outside `AppExceptionInterceptor`. Verify the denial reaches the client and the service handler is skipped. The authorization branch follows [the host profile](../SKILL.md#authentication-and-programcs-pipeline).

```csharp
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<MetricRpcInterceptor>();     // outermost
    options.Interceptors.Add<LoggingRpcInterceptor>();
    options.Interceptors.Add<AppExceptionInterceptor>();  // innermost: maps exceptions first
});
```

## AppExceptionInterceptor
The only place where exception→status translation lives. The gateway topology maps gRPC status back to HTTP via `AppExceptionHandler` (`solution-architecture`), so both sides keep their meaning across the hop.

```csharp
public class AppExceptionInterceptor(ILogger<AppExceptionInterceptor> logger) : Interceptor
{
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        try
        {
            return await continuation(request, context);
        }
        catch (ValidationException ex)      { throw new RpcException(new Status(StatusCode.InvalidArgument, ex.Message)); }
        catch (ArgumentException ex)        { throw new RpcException(new Status(StatusCode.InvalidArgument, ex.Message)); }
        catch (NotFoundException ex)        { throw new RpcException(new Status(StatusCode.NotFound, ex.Message)); }
        catch (UnauthorizedAccessException) { throw new RpcException(new Status(StatusCode.Unauthenticated, "Authentication required.")); }
        catch (ForbiddenException)         { throw new RpcException(new Status(StatusCode.PermissionDenied, "Access denied.")); }
        catch (OperationCanceledException) { throw new RpcException(new Status(StatusCode.Cancelled, string.Empty)); }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unhandled exception in {Method}", context.Method);
            // No exception details, no stack traces in the status: they can leak internals.
            throw new RpcException(new Status(StatusCode.Internal, "Internal error."));
        }
    }
}
```

- Unary only is shown. Streaming methods do not pass through `UnaryServerHandler`: override `ServerStreamingServerHandler`, `ClientStreamingServerHandler` and `DuplexStreamingServerHandler` with the same mapping (one private `Map(Exception)` method), or the service has no streaming methods.
- `ValidationException` is checked before `ArgumentException` only for readability: they are unrelated types. Keep the list in sync with `AppExceptionHandler` (HTTP).
- "Exists but belongs to someone else" stays `NotFound` when existence itself is sensitive (`security`).
- Extend the mapping only together with the HTTP mapping in `AppExceptionHandler` — one exception taxonomy, two transports.
- Client-side callers translate statuses back to domain errors (`api-clients`); a raw `RpcException` must not escape a client call site.

## Logging & Metrics Interceptors
- Logging and metrics interceptors share one base that wraps all four call kinds (unary, client/server/duplex streaming) and reads the status from the `RpcException` produced by the inner `AppExceptionInterceptor`; an order test pins the registration (mapped `NotFound`/`Internal`, not `Unknown`).
- Logging: one structured line per call with method, duration and resulting status code; the exception itself is logged once, by `AppExceptionInterceptor`. Never log request/response payloads (personal data); metadata keys only, values only for non-sensitive keys.
- Timing with `Stopwatch`/`ValueStopwatch` — never `DateTime.Now`.
- Metrics via `IMeterFactory`: a duration histogram and an outcome counter, low-cardinality tags only (constant method name, status code):

| Instrument | Type | Tags | Alert |
| ---------- | ---- | ---- | ----- |
| `rpc.server.requests` | counter | `rpc.method`, `rpc.grpc_status_code` | spike of `INTERNAL` / `UNAVAILABLE` |
| `rpc.server.duration` | histogram (s) | `rpc.method` | p99 growth |

- Host-level gRPC spans and metrics come from ASP.NET Core instrumentation inside `AddServiceDefaults()` (`observability`, metrics.md). The interceptor adds domain counters; it does not duplicate framework telemetry.

## gRPC Health Checks
- Package `Grpc.AspNetCore.HealthChecks`: `builder.Services.AddGrpcHealthChecks()` in `RpcHost` and `host.MapGrpcHealthChecksService()`.
- Register the same readiness checks that `/health` uses (database, broker) so the gRPC health report and the HTTP probes on the management port do not diverge.
- The health service follows the exposure rules of `MapDefaultEndpoints`: management plane only, not reachable through the ingress in production.

## Documentation
Each gRPC service must be documented in `docs/contracts/gRPC/<service>.md`.
