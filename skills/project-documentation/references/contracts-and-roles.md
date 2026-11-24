# Contracts, Roles & Settings Documentation Reference

## API Contracts (`docs/contracts/api/<controller>.md`)
- One file per controller in kebab-case.
- Documents business scenarios, required headers, request DTO table, and HTTP status codes.

## External API Clients (`docs/contracts/api-clients/<client>.md`)
- Target URLs per environment, auth mechanism (no secret values!), rate limits, retry policies.

## gRPC Contracts (`docs/contracts/gRPC/<service>.md`)
- Documents service methods, proto message formats, and mapped status codes.

## Broker Contracts (`docs/contracts/kafka/` & `docs/contracts/rabbitmq/`)
- Topic/queue name, message DTO payload, key semantics, idempotency notes, and retry/dead-letter policy.
- Message types with their `message-version` history (what changed in each version and until when the old version is published).
- Kafka: partitions, replication factor, retention, consumer groups. RabbitMQ: exchange type, queue type (quorum/classic), DLX and retry queues.
- Producers (and whether they publish via the outbox) and consumers.

## Roles & Permissions (`docs/roles/<role>.md`)
- User/Admin roles, Keycloak realm mappings, and accessible endpoints.

## Realtime Contracts (`docs/contracts/realtime/<hub>.md`)
- Hub path and authorization policy; client→server methods with parameters; server→client message names with payload DTO and version; groups and the authorization check behind each; the broker event that produces each message (`realtime-signalr`).

## Settings Documentation (`docs/settings/<host>.md`)
- Documents all configuration sections, environment overrides, and secrets injection instructions.
