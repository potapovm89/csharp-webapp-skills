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

## Documentation Structure
- `docs/description.md`: High-level business overview of the solution.
- `docs/arch/`: C4 architecture diagrams in Structurizr DSL (`workspace.dsl`).
- `docs/contracts/api/`: REST API documentation (one file per controller).
- `docs/contracts/api-clients/`: External REST API clients documentation.
- `docs/contracts/gRPC/`: gRPC service contracts documentation.
- `docs/contracts/kafka/` & `docs/contracts/rabbitmq/`: Message broker topic and queue contracts.
- `docs/contracts/realtime/`: SignalR hub contracts (one file per hub, `realtime-signalr`).
- `docs/database/`: Database schemas (`schema.dbml` for PostgreSQL, Markdown for Mongo and Redis).
- `docs/roles/`: Roles and authorization permissions.
- `docs/security/`: Threat model, scoped ASVS/control evidence, identity/MFA/step-up mapping, audit guarantees and vulnerability-response policy (`security`).
- `docs/settings/`: Host configuration files documentation.
- `docs/delivery/pipeline.md`: CI/CD stage/gate contract, artifact identity/evidence, trust/exception policy and promotion/rollback decisions (`delivery-and-supply-chain`).
- `docs/reliability/<journey>.md`: Applicable NFR/SLI/SLO, measurement/window, load/failure acceptance, alert/budget policy and evidence (`reliability-and-performance`).
- `docs/recovery/plan.md` and `docs/recovery/drills/`: Store/side-effect inventory, approved RPO/RTO/retention, restore/reconciliation procedure and measured drill results (`data-recovery`).
- `docs/adr/`: Architecture Decision Records (`NNNN-<title>.md`), immutable after acceptance.
- `docs/runbooks/`: Runbooks for every alert, DLQ replay, outbox backlog, failed migration, secret rotation.

## Subtopic References
- [arch-dsl.md](./references/arch-dsl.md) — Structurizr C4 DSL modeling.
- [database-dbml.md](./references/database-dbml.md) — PostgreSQL DBML, Mongo collections, and Redis key docs.
- [contracts-and-roles.md](./references/contracts-and-roles.md) — REST API, gRPC, broker, roles, and settings documentation templates.
- [adr-and-runbooks.md](./references/adr-and-runbooks.md) — ADR template and rules, runbook template, DLQ replay runbook.

Delivery, reliability and recovery document content is defined by [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md), [reliability-and-performance](../reliability-and-performance/SKILL.md) and [data-recovery](../data-recovery/SKILL.md). Link their records and access-controlled evidence instead of maintaining conflicting targets/policies in multiple files. Create only the records applicable to the system; unknown targets and unperformed checks remain explicit.
