# C4 Architecture (Structurizr DSL) Reference

## Location: `docs/arch/workspace.dsl`
- Uses Structurizr DSL to model C4 System Context and Containers.
- Containers match actual solution host projects and infrastructure resources from AspireHost (e.g. `webhost`, `rpchost`, `db`, `broker`).
- Links define protocols: `HTTPS/JSON`, `gRPC`, `AMQP`, `PostgreSQL/TCP`.
- Images are not committed to git; images are compiled from the DSL by the Structurizr tooling (see the tooling note below).
- **Tooling note (2026-11)**: the classic Structurizr CLI has reached end of life (official announcement, January 2026); its successor is Structurizr vNext. Keep `workspace.dsl` as the source of truth; render with a pinned version of the existing tooling or migrate to vNext when the platform adopts it — the diagrams must not go unmaintained because the renderer changed.
- Whenever a host, database, broker, or external dependency is added or removed, update `docs/arch/workspace.dsl`.
