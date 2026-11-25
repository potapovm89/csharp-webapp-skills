---
name: message-brokers
description: >-
  Use this skill when publishing or consuming messages with Apache Kafka or RabbitMQ, implementing message producers, KafkaHost/RabbitHost consumers, idempotent message processors (IProcessor), the transactional outbox/inbox, retries and dead-letter queues, or message contract versioning.
  Triggers (RU): Kafka, RabbitMQ, брокер, продюсер, консьюмер, outbox, inbox, DLQ, повторная публикация, идемпотентность.
---

# Message Brokers (Kafka & RabbitMQ)

This skill covers asynchronous messaging with Apache Kafka and RabbitMQ in the solution. The target is **at-least-once** delivery: durable outbox storage and confirmed publication allow recovery, and consumers tolerate duplicates. Guarantees depend on broker replication/retention, database durability and operational recovery; they do not promise survival of arbitrary data loss or exactly-once business effects.

## Choose the Workflow

Read the existing topic or queue contract, broker configuration and failure path. Before implementing, read the reference for the broker in use ([Kafka](./references/kafka.md) or [RabbitMQ](./references/rabbitmq.md)) and the additional reference for the applicable workflow:

| Change | Workflow |
|---|---|
| Consumer | Read [handlers.md](./references/handlers.md). Resolve the idempotency key, transient retry limit, terminal DLQ route and settlement point before wiring the processor. |
| Publish without a business DB write | Use confirmed publication. Verify the destination, message key/id, version and trace headers in the contract. |
| Publish together with a business DB write or change a relay | Read [outbox.md](./references/outbox.md). Commit the business change and outbox row together. The relay marks the row published after confirmed delivery. |

## General Broker Rules

- **Topic & Queue Naming**: Always `kebab-case` (e.g. `order-created-events`); retry and dead-letter destinations are `<name>-retry` and `<name>-dlq`.
- **Contracts**: Message payloads are strongly typed DTOs from `Itl.<Project>.Contracts` serialized as JSON, with standard headers `message-id`, `message-type`, `message-version`, `traceparent`, `correlation-id`.
- **Transactional outbox (mandatory)**: if a use-case writes to the database and publishes a message, the message is written to the outbox table in the same transaction and published by a relay. Never publish directly inside or right after a DB transaction. See [outbox.md](./references/outbox.md).
- **Idempotency (mandatory)**: deduplicate physical redelivery by `message-id` or use a uniquely constrained key for the same business operation. Apply deduplication in the transaction that changes business state. Across paired contract versions, use a shared logical event key as described in the cutover.
- **No silent loss**: the processor throws on failure; the worker retries transient errors, then moves the message to a retry or dead-letter destination, and only then acknowledges / stores the offset. Poison messages and unsupported versions go to the DLQ immediately. DLQs are monitored.
- **Versioning**: additive changes keep `message-version`. Consumers ignore unknown fields. Breaking changes increment the version and route it separately from older consumers. For Kafka, follow the [versioned-topic cutover](./references/kafka.md#breaking-change-cutover).
- **Tracing**: propagate W3C trace context through message headers.
- **Contract Documentation**: every topic/queue is documented in `docs/contracts/kafka/` or `docs/contracts/rabbitmq/`: key semantics, message types and versions, partitions/queue type, retry and DLQ destinations, producers and consumers.

## Producers & Consumers Overview

- **Kafka**:
  - Auto-creation of topics is prohibited (`AllowAutoCreateTopics: false`).
  - Producers use `IProducer<string, string>` with `Acks = All` and `EnableIdempotence = true`; the key is the aggregate id.
  - Consumers: `EnableAutoOffsetStore = false`, offset stored only after the message is settled; messages of one partition are processed sequentially.
  - See [kafka.md](./references/kafka.md).
- **RabbitMQ**:
  - Connection strings: `rabbitMessaging`.
  - Only RabbitMQ.Client 7.x is supported; the 6.x API (`CreateModel`, `BasicPublish`, `DispatchConsumersAsync`) is prohibited. If the solution is on 6.x, stop and tell the user.
  - Consumers use `AsyncEventingBasicConsumer` (`ReceivedAsync`) with manual `BasicAckAsync`/`BasicNackAsync`.
  - Publish persistent messages (`Persistent = true`) with publisher confirms; durable (quorum in deployed environments) queues, each with a dead-letter exchange.
  - See [rabbitmq.md](./references/rabbitmq.md).
- **Processors**:
  - Both brokers feed into `IProcessor<TKey, TMessage>` via `ConsumedMessageDto` (key, payload, headers).
  - See [handlers.md](./references/handlers.md) for the lifecycle and the new-consumer checklist.

Check existing worker code against the following rules, including code inherited from the template. Fix gaps in scope and report the rest.

## Consumer Checklist (verify before finishing)

| Check | Kafka | RabbitMQ |
|---|---|---|
| Ack/offset only after the message is settled (processed, or sent to retry/DLQ) | `EnableAutoOffsetStore: false` in config + `consumer.StoreOffset(result)` after settle; `EnableAutoCommit` may stay `true` (it commits stored offsets only) | `autoAck: false` + `BasicAckAsync` after settle |
| Ordering | one partition is processed sequentially; parallelism only across partitions (one loop per assigned partition), never a semaphore over `Consume()` results of the same partition | `prefetchCount` bounds parallelism; ordering not guaranteed, consumers are idempotent |
| Transient failure | bounded retries with backoff, then retry topic / DLQ | bounded retries (`x-delivery-limit` on quorum queues or retry queue with TTL), then DLX |
| Poison message (malformed JSON, null payload, failed validation, unsupported `message-version`, permanent 4xx/`InvalidArgument` from a downstream call) | confirmed publication directly to `<topic>-dlq` with error headers | confirmed, routable publication then ACK, or `BasicNackAsync(requeue: false)` with verified at-least-once dead-lettering |
| Never | log-and-return, catch-all that swallows, `StoreOffset` before processing | `requeue: true` on failure (infinite loop), queue without `x-dead-letter-exchange` |

The processor's contract: **return = success, throw = failure**. A processor that logs a warning and returns on a bad message tells the worker "success" and the message is lost. Throw `PoisonMessageException` for permanent errors so the worker sends the message to the DLQ without retries.

**Example** — Send a valid message, a duplicate and a malformed message through each configured broker. Assert the expected business records and read each actual DLQ to match the malformed fixture by a known ID or source position. Verify its payload and diagnostic metadata.

## Completion Evidence

Run the applicable cases against an isolated broker and report the commands and observed results:

- **Consumer:** Duplicate input produces one business effect. A poison input produces no business effect and reaches the actual DLQ. When the DLQ transfer fails, Kafka must preserve the original offset for replay, including protection against a later offset skipping it. With application-managed RabbitMQ forwarding, keep the delivery unacknowledged until a routed publish is confirmed. With broker-managed at-least-once dead-lettering, verify that the source retains the message during target failure and delivers it after recovery.
- **Producer:** Confirmed publication reaches the configured destination with the expected key/id, version and headers. Failed confirmation, or an unroutable mandatory RabbitMQ publication, is reported as a failure to the caller.
- **Outbox:** Rollback leaves neither the business change nor its outbox row. Commit leaves both. Failed publication keeps the row retryable or retained for intervention, and confirmation allows the relay to mark it published. A replay after confirmation with a failed relay commit retains the same message id.
