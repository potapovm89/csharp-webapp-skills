---
name: project-documentation
description: >-
  Use this skill when creating or updating project documentation in docs/, including C4 architecture (workspace.dsl), DBML database schemas (schema.dbml), API / gRPC contracts, broker topics, roles, or settings.
  Triggers (RU): документация, docs, C4, DBML, контракты, ADR, runbook, регламент.
---

# Project Documentation (`docs/`)

This skill defines the requirements for maintaining living system documentation in the `docs/` directory of the solution.

## Golden Principle
Changes to endpoints, database schemas, message broker events, application options, permissions, observable behavior or system architecture include corresponding updates to `docs/` in the same Merge Request. Purely cosmetic/internal edits with no documentation impact do not require an artificial docs change; record that assessment in the MR.

Concrete minimums that are easy to skip:
- `docs/contracts/api/` has **one file per controller**; a single README for the whole API is not enough.
- `docs/roles/` exists as soon as the first `[Authorize]` appears; `docs/adr/` gets a record for every user decision recorded in `AGENTS.md` (Stories mechanism, host topology, WorkerHost replicas); `docs/runbooks/` gets a DLQ replay runbook with the first consumer.
- Tool output (language-server logs, generated previews, `*.log`) is never committed; add it to `.gitignore`.

## Route a Change to Its Documentation

Review the task diff and select every row whose contract or behavior changed. Read the relevant reference before editing and compare the result against the stated source.

| Change | Source to read | Target and check |
|---|---|---|
| Product behavior | Accepted requirements and observable behavior | `docs/description.md`: verify supported user flows and label planned ones. |
| REST controller | Routing, policies, DTOs and OpenAPI | `docs/contracts/api/<controller>.md`: verify methods, routes, permissions and responses for that controller. |
| External API client | Client implementation and provider contract | `docs/contracts/api-clients/<client>.md`: verify operations, timeouts, errors and retry semantics. |
| gRPC or real-time contract | `.proto` files or hub methods and payload types | `docs/contracts/gRPC/` or `docs/contracts/realtime/`: verify methods, messages and access rules. |
| Broker contract | Producer/consumer code and topology | `docs/contracts/kafka/` or `docs/contracts/rabbitmq/`: verify destination, versions, keys and retry/DLQ routing. |
| Persistent schema | Migration, model and storage configuration | `docs/database/`: compare PostgreSQL DBML with changed tables, keys and indexes. For Mongo or Redis, compare the Markdown record with the actual data and retention rules. |
| Roles or security boundary | Authorization policies, claims mapping and applicable ADR | `docs/roles/` and affected `docs/security/` records: verify permissions, ownership and evidence. |
| Host setting | Options class, binding and effective configuration | `docs/settings/<host>.md`: verify keys, applicable defaults and validation. |
| Topology or architectural decision | Project references, wiring and accepted project decision | Configured C4 workspace under `docs/arch/` and applicable `docs/adr/NNNN-<title>.md`: verify relationships and decision status. Keep accepted ADRs immutable. |
| Delivery, reliability or recovery | CI and rendered manifests, telemetry and queries, or store inventory and restore evidence | `docs/delivery/pipeline.md`, `docs/reliability/<journey>.md` or `docs/recovery/`: verify applicable gates, measurement, objectives and evidence using the owning skill. |
| Alert or operator procedure | Alert rule, failure path and approved operating procedure | `docs/runbooks/`: verify the trigger, owner, actions and observable recovery outcome. |

**Example from `it-letter`** — `docs/contracts/api/README.md` summarizes several controllers. For a change in `MessagesController`, compare the controller and its OpenAPI operations with `docs/contracts/api/messages.md`. Create the per-controller file if missing and use the README as an index.

## Subtopic References
- [arch-dsl.md](./references/arch-dsl.md) — Structurizr C4 DSL modeling.
- [database-dbml.md](./references/database-dbml.md) — PostgreSQL DBML, Mongo collections, and Redis key docs.
- [contracts-and-roles.md](./references/contracts-and-roles.md) — REST API, gRPC, broker, roles, and settings documentation templates.
- [adr-and-runbooks.md](./references/adr-and-runbooks.md) — ADR template and rules, runbook template, DLQ replay runbook.

Delivery, reliability and recovery document content is defined by [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md), [reliability-and-performance](../reliability-and-performance/SKILL.md) and [data-recovery](../data-recovery/SKILL.md). Link their records and access-controlled evidence instead of maintaining conflicting targets/policies in multiple files. Create only the records applicable to the system; unknown targets and unperformed checks remain explicit.

## Completion Evidence

- For every changed public or operational contract, list the source and affected document in the MR. For a change without documentation impact, record the reason.
- Verify each affected REST controller has its own document and that the changed methods, routes, authorization and response codes match code and OpenAPI. For other rows, record the corresponding source comparison and run the project's existing format or link validator when available.
- Record discrepancies, open decisions and any check that could not run. Link the underlying evidence where the document describes a measured or executed result.
