# RabbitMQ Integration Reference

Only **RabbitMQ.Client 7.x** is supported. Its API is fully asynchronous.

- Never use the 6.x API: `CreateModel`, `BasicPublish`, `BasicAck`, `CreateBasicProperties`, `consumer.Received`, `ConnectionFactory.DispatchConsumersAsync`.
- Before writing RabbitMQ code, check the `RabbitMQ.Client` version resolved in the solution (`Directory.Packages.props`, or the version pulled in by `Aspire.RabbitMQ.Client`). If it is 6.x, do not write 6.x code and do not upgrade the package on your own: stop and tell the user. The upgrade to 7.x is a separate task, because it changes every existing publisher and consumer.

## Configuration
Options class: `RabbitMqOptions` (properties: `Queue`, `Exchange`, `RoutingKey` — the routing key must match the exchange binding declared in the Topology table below).
Connection string: `rabbitMessaging` (`amqp://user:password@host:5672`).

## Setup
```csharp
builder.Services.AddOptionsWithValidation<RabbitMqOptions>(builder.Configuration.GetRequiredSection(nameof(RabbitMqOptions)).Bind);
builder.AddRabbitMQClient("rabbitMessaging"); // registers a singleton IConnection
```

## Publishing
Messages must be **persistent** and published with **publisher confirms**, otherwise they are lost on broker restart or network failure.

```csharp
await using var channel = await connection.CreateChannelAsync(
    new CreateChannelOptions(
        publisherConfirmationsEnabled: true,
        publisherConfirmationTrackingEnabled: true),
    ct);

var properties = new BasicProperties
{
    Persistent = true,                     // delivery_mode = 2
    ContentType = "application/json",
    MessageId = messageId.ToString(),      // used by consumers for idempotency
};

var body = JsonSerializer.SerializeToUtf8Bytes(messageDto);

// With confirmation tracking enabled, the call completes only after the broker confirms
// and throws if the message was nacked or returned (mandatory: true).
await channel.BasicPublishAsync(
    exchange: rabbitMqOptions.Exchange,
    routingKey: rabbitMqOptions.RoutingKey,   // matches the queue binding in the topology; empty only for a fanout binding
    mandatory: true,
    basicProperties: properties,
    body: body,
    cancellationToken: ct);
```

- Do not create a channel per message in hot paths; reuse a channel per publisher (channels are not thread-safe — one channel per concurrent publisher).
- Exchanges and queues must be declared `durable: true`.

## Consumer Host Setup (RabbitHost)

This example uses bounded in-process retries and a source quorum queue with verified at-least-once dead-lettering. Use the project's explicit transient-error classifier and backoff. Configure a broker delivery limit if the retry policy needs a bound across crashes. Use a separate confirmed-forwarding branch for delayed retries when a worker should not hold the delivery.

```csharp
await channel.BasicQosAsync(prefetchSize: 0, prefetchCount: options.PrefetchCount, global: false, ct);

var consumer = new AsyncEventingBasicConsumer(channel);
consumer.ReceivedAsync += async (_, ea) =>
{
    try
    {
        for (var attempt = 1; ; attempt++)
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var processor = scope.ServiceProvider.GetRequiredService<IProcessor<string, string>>();
                await processor.ExecuteAsync(ToConsumedMessageDto(ea), ct);
                break;
            }
            catch (PoisonMessageException ex)
            {
                logger.LogWarning(ex, "Rejected poison message {MessageId}", ea.BasicProperties.MessageId);
                await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: false, ct);
                return;
            }
            catch (Exception ex) when (IsTransient(ex) && !ct.IsCancellationRequested)
            {
                if (attempt >= options.MaxAttempts)
                {
                    logger.LogError(ex, "Retry budget exhausted for message {MessageId}", ea.BasicProperties.MessageId);
                    await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: false, ct);
                    return;
                }

                await Task.Delay(GetBackoff(attempt), ct);
            }
        }

        await channel.BasicAckAsync(ea.DeliveryTag, multiple: false, ct);
    }
    catch (OperationCanceledException) when (ct.IsCancellationRequested)
    {
        // Leave incomplete work unacknowledged. Closing the channel makes it deliverable again.
    }
};

await channel.BasicConsumeAsync(queue: rabbitMqOptions.Queue, autoAck: false, consumer: consumer, ct);
```

Unknown exceptions and consumer-channel failures are handled by the host's error supervisor: log and close/recover the channel so an unacknowledged delivery remains replayable. Never treat a failed ACK, NACK or forwarding publish as successful processing. A logged NACK with a verified DLX means submitted for dead-lettering. The DLQ delivery must be observed separately.

- Always `autoAck: false`. Acknowledge after successful processing or a confirmed and routable application-managed transfer.
- Never `requeue: true` for poison messages — it causes an infinite redelivery loop. Configure a dead-letter exchange (`x-dead-letter-exchange`) on every queue.
- Resolve a scoped `IProcessor<string, string>` per message.
- The processor must **throw** when processing failed. If it catches and only logs, the worker acknowledges the message and it is lost.

## Topology
Declared by infrastructure (Aspire `RabbitMqOptions.Exchanges` locally, IaC / definitions in deployed environments) and documented in `docs/contracts/rabbitmq/`.

| Object | Rule |
| ------ | ---- |
| Exchange | `durable: true`, `autoDelete: false` |
| Queue | `durable: true`; `x-queue-type: quorum` in deployed environments (replicated, survives node loss) |
| Dead-lettering | every work queue has `x-dead-letter-exchange: <exchange>-dlx`; the DLX routes to `<queue>-dlq` |
| Retry with delay | `<queue>-retry` with `x-message-ttl` (e.g. 30000) and `x-dead-letter-exchange` pointing back to the work exchange; the consumer republishes a failed message there with an incremented `attempts` header, and to the DLQ after `MaxAttempts` |

Classic non-durable queues are acceptable only in local and test environments.

For broker-managed durable dead-letter delivery, configure **each source quorum queue** with `dead-letter-strategy: at-least-once`, `overflow: reject-publish` and its DLX. This includes a delayed retry queue if TTL expiry dead-letters a message back to the work exchange. Verify the effective settings on the broker, including feature flags on supported versions. Quorum queues default to at-most-once dead-lettering. If the topology cannot retain deliveries as required, forward with `mandatory: true` and publisher confirmation tracking, and ACK the original only after confirming that it was routed. On a failed or returned publish, leave the original replayable and recover the channel. Use stable IDs because retry and DLQ delivery can duplicate. Test DLQ unavailability and recovery, and separately a retry queue expiring while its work target is unavailable. [RabbitMQ quorum dead-lettering](https://www.rabbitmq.com/docs/quorum-queues#activating-at-least-once-dead-lettering)

## Replaying a DLQ
- Read the DLQ with `BasicGetAsync(autoAck: false)`, at most as many messages as the queue had at start (requeued ones are not read again).
- Target exchange and routing key come from the `x-death` header that RabbitMQ adds on dead-lettering.
- Publish with confirms and `Persistent = true`, keep `MessageId`, drop `x-death`/`x-first-death-*`/`x-last-death-*`, add `replayed-at`; `BasicAckAsync` the DLQ message only after the publish is confirmed.
- Messages that do not match a filter are `BasicNackAsync(requeue: true)`.

## Publishing Through the Outbox
If the use-case also writes to the database, do not call `BasicPublishAsync` from the handler: write an outbox row (see `outbox.md`); the relay publishes it with the rules above.

## Headers & Tracing
- Set `MessageId` (= outbox id), `Type` (`message-type`), `ContentType`, and `Headers["message-version"]`, `Headers["correlation-id"]` when available.
- Header values arrive as `byte[]` on the consumer side; decode with `Encoding.UTF8.GetString`.
- Propagate the trace context in `BasicProperties.Headers` with the explicit W3C propagator (inject on publish, extract on consume) exactly as shown in `kafka.md` "Tracing".

## Versioning
Additive changes keep the version. For a breaking change, bump `message-version` and route the new representation through a separate routing key and queue binding, so the v1 queue receives only v1. If a logical consumer processes overlapping versions, deduplicate by the shared logical event identity as in [the version cutover](./kafka.md#breaking-change-cutover). Keep the unsupported-version DLQ rule for deliveries that violate the queue's contract.
