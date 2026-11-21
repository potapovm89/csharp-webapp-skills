---
name: message-brokers
description: >-
  Use this skill when publishing or consuming messages with Apache Kafka or RabbitMQ, implementing message producers, KafkaHost/RabbitHost consumers, idempotent message processors (IProcessor), the transactional outbox/inbox, retries and dead-letter queues, or message contract versioning.
  Triggers (RU): Kafka, RabbitMQ, брокер, продюсер, консьюмер, outbox, inbox, DLQ, повторная публикация, идемпотентность.
---

# Message Brokers (Kafka & RabbitMQ)

This skill covers asynchronous messaging with Apache Kafka and RabbitMQ in the solution. The target is **at-least-once** delivery: durable outbox storage and confirmed publication allow recovery, and consumers tolerate duplicates. Guarantees depend on broker replication/retention, database durability and operational recovery; they do not promise survival of arbitrary data loss or exactly-once business effects.

## General Broker Rules

- **Topic & Queue Naming**: Always `kebab-case` (e.g. `order-created-events`); retry and dead-letter destinations are `<name>-retry` and `<name>-dlq`.
- **Contracts**: Message payloads are strongly typed DTOs from `Itl.<Project>.Contracts` serialized as JSON, with standard headers `message-id`, `message-type`, `message-version`, `traceparent`, `correlation-id`.
- **Transactional outbox (mandatory)**: if a use-case writes to the database and publishes a message, the message is written to the outbox table in the same transaction and published by a relay. Never publish directly inside or right after a DB transaction. See [outbox.md](./references/outbox.md).
- **Idempotency (mandatory)**: every consumer deduplicates — inbox table keyed by `message-id`, or a business key protected by a unique constraint, in the same transaction as the business change. A check-then-insert without a unique constraint is not idempotent.
- **No silent loss**: the processor throws on failure; the worker retries transient errors, then moves the message to a retry or dead-letter destination, and only then acknowledges / stores the offset. Poison messages and unsupported versions go to the DLQ immediately. DLQs are monitored.
- **Versioning**: additive changes keep `message-version`; breaking changes bump it with a period of parallel publishing. Consumers ignore unknown fields.
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

**Before implementing or changing a consumer or producer, read [kafka.md](./references/kafka.md) or [rabbitmq.md](./references/rabbitmq.md) and [handlers.md](./references/handlers.md).** Existing worker code (including the template's) is not proof of compliance: check it against the list below and fix or report every miss.

## Consumer Checklist (verify before finishing)

| Check | Kafka | RabbitMQ |
|---|---|---|
| Ack/offset only after the message is settled (processed, or sent to retry/DLQ) | `EnableAutoOffsetStore: false` in config + `consumer.StoreOffset(result)` after settle; `EnableAutoCommit` may stay `true` (it commits stored offsets only) | `autoAck: false` + `BasicAckAsync` after settle |
| Ordering | one partition is processed sequentially; parallelism only across partitions (one loop per assigned partition), never a semaphore over `Consume()` results of the same partition | `prefetchCount` bounds parallelism; ordering not guaranteed, consumers are idempotent |
| Transient failure | bounded retries with backoff, then retry topic / DLQ | bounded retries (`x-delivery-limit` on quorum queues or retry queue with TTL), then DLX |
| Poison message (malformed JSON, null payload, failed validation, unsupported `message-version`, permanent 4xx/`InvalidArgument` from a downstream call) | straight to `<topic>-dlq` with error headers | `BasicNackAsync(requeue: false)` → DLX → `<queue>-dlq` |
| Never | log-and-return, catch-all that swallows, `StoreOffset` before processing | `requeue: true` on failure (infinite loop), queue without `x-dead-letter-exchange` |

The processor's contract: **return = success, throw = failure**. A processor that logs a warning and returns on a bad message tells the worker "success" and the message is lost. Throw `PoisonMessageException` for permanent errors so the worker sends the message to the DLQ without retries.
