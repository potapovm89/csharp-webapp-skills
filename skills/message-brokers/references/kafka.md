# Kafka Integration Reference

## Configuration
Options class: `KafkaOptions`. Producer/consumer specific configs are nested; properties of the nested sections are bound into the Confluent client config.
Connection string: `kafkaMessaging`.

```json
"KafkaOptions": {
  "AllowAutoCreateTopics": false,
  "KafkaProducer": {
    "Topic": "order-events",
    "Acks": "All",
    "EnableIdempotence": true,
    "LingerMs": 5,
    "CompressionType": "Lz4"
  },
  "KafkaConsumer": {
    "Topic": "order-events",
    "GroupId": "billing-service-order-events",
    "EnableAutoCommit": true,
    "EnableAutoOffsetStore": false,
    "AutoOffsetReset": "Earliest"
  }
}
```

## Topics
- Topics are created by infrastructure, never by clients (`AllowAutoCreateTopics: false`). Aspire creates them locally from `KafkaOptions.Topics`.
- Every topic is documented in `docs/contracts/kafka/<topic>.md`: partitions, replication factor, retention, key semantics, message types and versions, DLQ topic, producers and consumers.
- Defaults for deployed environments: replication factor 3, `min.insync.replicas` 2.
- Retry and dead-letter topics: `<topic>-retry`, `<topic>-dlq` (kebab-case, validated by `[KebabCaseCollectionItemsValidation]`).

## Producer
```csharp
builder.AddKafkaProducer<string, string, KafkaOptions>("KafkaProducer");
// Injects IProducer<string, string>
```

Rules:
- `Acks = All` + `EnableIdempotence = true` — mandatory; without them a broker failover can lose or duplicate messages.
- **Key** = identifier of the aggregate the message is about (order id, customer id). Kafka guarantees order only within a partition, i.e. per key. Never use a random key or `null` when consumers rely on order.
- If the same use-case writes to the database, publish through the **outbox** (see `outbox.md`), never directly.
- Always `await ProduceAsync(...)` and let a `ProduceException` propagate; never fire-and-forget `Produce` without a delivery handler.
- Standard headers (UTF-8 values):

| Header | Value |
| ------ | ----- |
| `message-id` | Unique id (`Guid.CreateVersion7()`); equals the outbox row id |
| `message-type` | kebab-case type, e.g. `order-created` |
| `message-version` | Contract version, integer |
| `traceparent` | W3C trace context (see "Tracing") |
| `correlation-id` | Business correlation id, when available |

## Consumer (KafkaHost)
```csharp
builder.AddKafkaConsumer<string, string, KafkaOptions>("KafkaConsumer");
builder.Services.AddScoped<IProcessor<string, string>, KafkaProcessor>();
builder.Services.AddHostedService<Worker>();
```

### Offset management — at-least-once
- `EnableAutoOffsetStore = false`: an offset is stored **only after** the message is fully processed (or sent to retry/DLQ). With the default (`true`) the offset is stored on `Consume`, and a crash during processing loses the message.
- `EnableAutoCommit = true` then periodically commits only the stored offsets. Alternatively disable auto-commit and call `Commit` explicitly — never both auto-store and auto-commit.
- Storing offset N marks **all** earlier offsets of that partition as done. Therefore process messages of one partition **sequentially**. Scale with more partitions and more consumer instances, not with parallel processing inside a partition.
- On shutdown call `consumer.Close()` so the final offsets are committed and the group rebalances immediately.

```csharp
while (!ct.IsCancellationRequested)
{
    var result = consumer.Consume(ct);
    if (result is null || result.IsPartitionEOF)
    {
        continue;
    }

    await HandleWithRetryAsync(result, ct);   // processes, or forwards to retry/DLQ
    consumer.StoreOffset(result);             // only after the message is settled
}
```

### Errors, retry and DLQ
Never block a partition forever and never drop a message silently:
1. **Poison message** (cannot be deserialized, unknown `message-type` / unsupported `message-version`) → publish to `<topic>-dlq` immediately.
2. **Transient error** (DB timeout, HTTP 5xx) → retry in-process with exponential backoff (Polly, e.g. 3 attempts within seconds).
3. Still failing → publish to `<topic>-retry` (a delayed consumer re-processes it later) or to `<topic>-dlq` when retries are exhausted.
4. Store the offset of the original message only after step 1, 2 or 3 completed successfully.

DLQ/retry messages keep the original key, value and headers and add: `original-topic`, `original-partition`, `original-offset`, `error-type`, `error-message` (no stack traces with sensitive data), `attempts`. DLQ topics are monitored and alerted on (`messaging.consumer.messages{outcome=dead_lettered}`).

### Replaying a DLQ
Replay is a manual operation through a dedicated tool (console app / one-off job), after the cause is fixed:
- Target topic from the `original-topic` header (explicit `--target` only as a fallback).
- Keep key, value and `message-id`; drop the dead-letter headers (`original-*`, `error-*`, `attempts`); add `replayed-at`. Because `message-id` is kept, the consumer's inbox skips messages that were in fact processed.
- Publish with `Acks = All`, commit the DLQ offset only after the publish is confirmed (dedicated consumer group, e.g. `dead-letter-replay`).
- Replaying a single message by `message-id` scans the DLQ with a throw-away group **without** committing, so the remaining messages stay replayable.
- Support `--dry-run` and `--max`; log every replayed message id.

### Idempotency
Delivery is at-least-once (rebalances, retries, outbox re-publishes). Every processor deduplicates by `message-id` or by a business key with a unique constraint — see "Inbox" in `outbox.md`.

## Message Contracts & Versioning
- Payload: JSON of a DTO from `Contracts`; the DTO name and `message-type` are documented together.
- **Compatible changes** (keep `message-version`): add optional fields. Consumers ignore unknown fields (System.Text.Json default — do not enable `UnmappedMemberHandling.Disallow` on consumers).
- **Breaking changes** (remove/rename a field, change a type or meaning): bump `message-version`, publish both versions in parallel until every consumer has migrated, then stop the old one. Alternatively introduce a new topic. Never change the meaning of an existing version.
- A consumer that receives a version it does not support sends the message to the DLQ and raises an alert instead of guessing.

## Tracing
There is no built-in OpenTelemetry instrumentation for `Confluent.Kafka` in the shared defaults; propagate the trace context through headers manually.

Use an explicit W3C propagator. `Propagators.DefaultTextMapPropagator` is a no-op until the OpenTelemetry SDK is initialized in the process, so headers would silently stay empty (e.g. in tools, tests, or hosts without tracing):

```csharp
public static readonly TextMapPropagator Propagator = new CompositeTextMapPropagator(
    [new TraceContextPropagator(), new BaggagePropagator()]);
```

```csharp
// Producer side
var headers = new Headers();
Propagator.Inject(
    new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
    headers,
    (h, key, value) => h.Add(key, Encoding.UTF8.GetBytes(value)));

// Consumer side
var parent = Propagator.Extract(
    default,
    result.Message.Headers,
    (h, key) => h.TryGetLastBytes(key, out var bytes) ? [Encoding.UTF8.GetString(bytes)] : []);

using var activity = ActivitySource.StartActivity(
    $"{result.Topic} process", ActivityKind.Consumer, parent.ActivityContext);
```

## Known Gaps in Existing Code
When touching an existing Kafka consumer, check it against this reference: auto offset store left enabled, parallel processing inside one partition, exceptions only logged (message silently skipped), no DLQ. Fix these as part of the task or report them to the user.
