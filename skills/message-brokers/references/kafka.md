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
| `message-id` | Physical message ID. For an outbox publication this is the row ID and stays the same on its retries. |
| `event-id` | Stable logical event ID. Required on both representations during the breaking-change cutover. |
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
- Keep the key, value, `message-id` and `event-id` when present. Drop the dead-letter headers (`original-*`, `error-*`, `attempts`) and add `replayed-at`. This preserves both physical and logical deduplication.
- Publish with `Acks = All`, commit the DLQ offset only after the publish is confirmed (dedicated consumer group, e.g. `dead-letter-replay`).
- Replaying a single message by `message-id` scans the DLQ with a throw-away group **without** committing, so the remaining messages stay replayable.
- Support `--dry-run` and `--max`; log every replayed message id.

### Idempotency
Delivery is at-least-once (rebalances, retries, outbox re-publishes). Use `message-id` for physical redelivery or a unique business key for the same operation. During a version cutover with overlapping representations, deduplicate by `(logical_consumer, event_id)` inside the transaction that applies the business effect. The logical consumer identity stays the same when the Kafka group or topic changes. See [inbox handling](./outbox.md#inbox-consumer-idempotency).

## Message Contracts & Versioning
- Payload: JSON of a DTO from `Contracts`; the DTO name and `message-type` are documented together.
- **Compatible changes** (keep `message-version`): add optional fields. Consumers ignore unknown fields (System.Text.Json default — do not enable `UnmappedMemberHandling.Disallow` on consumers).
- **Breaking changes** (remove/rename a field, change a type or meaning): increment `message-version` and introduce a separate versioned topic. Preserve the old topic's contract for its existing consumers. Follow the cutover below.
- Keep the unsupported-version DLQ rule for deliveries that violate the topic's contract. Route normal v2 traffic to the new topic.

### Breaking Change Cutover

1. Inventory the v1 groups and their logical consumer identities. Provision the v2 topic and capture start offsets per v2 partition at or before the first dual publication. Retain enough history for each group to cut over and roll back.
2. Assign one `event-id` per business event and persist it on both publications. With outbox, write both destination rows in the same transaction as the business change. Each row has its own `message-id`. Legacy groups continue to consume v1 only.
3. Before a group can process overlapping traffic, deploy logical-event deduplication on its v1 and v2 paths with the same logical consumer identity. Use the same transactional inbox or shared unique business key. A shadow group can validate v2 separately with business effects disabled and with its own checkpoint, leaving the active inbox untouched.
4. For each group, record a settled v1 boundary once v2 coverage is verified from its captured start. Drain v1 through that boundary and stop its active workers. Start the group's v2 path from the captured v2 offsets. Compare coverage using event IDs and per-topic evidence. Kafka offsets are topic- and partition-specific, so copying v1 offsets or starting at `latest` can skip events. Keep the v1 checkpoint and dual publishing during the agreed rollback window.
5. End v1 publishing only after every group has switched, the final drain is accounted for and rollback no longer depends on it.

**Verification:** deliver v1 and v2 for the same `event-id` with distinct `message-id` values. Assert a v1-only group sees only v1 and produces no unsupported-version DLQ entry. Switch one group across an overlapping boundary and replay across the switch. Assert every expected event arrives and each logical consumer applies it once. Verify rollback from the recorded v1 checkpoint while both topics are retained.

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
