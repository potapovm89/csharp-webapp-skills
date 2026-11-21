---
name: feature-recipes
description: >-
  Use this skill for cross-layer feature checklists (REST, consumers, jobs, entities, external APIs and gRPC), delivery/recovery/reliability integration recipes, or reference project-helper signatures. Triggers (RU): добавь эндпоинт, новый консьюмер, новая джоба, новая сущность, новая интеграция, сделай фичу целиком, сквозной рецепт, чеклист.
---

# Feature Recipes (End to End)

Each recipe is an ordered checklist that links the detailed skills. Follow it top to bottom; a step is skipped only if it does not apply, never because it is tedious. Before starting, run the `task-lifecycle` steps (plan, confidence check).

Reference helper signatures are in [project-helpers.md](./references/project-helpers.md). Confirm the current solution's API before implementing a call; code determines signatures, while these skills determine required behavior. An older API or template omission does not waive security, transaction or delivery requirements: adapt the implementation or report the needed change.

## Recipes
- [recipes.md](./references/recipes.md):
  1. REST endpoint: write to the DB + publish an event (outbox).
  2. Broker consumer (Kafka / RabbitMQ) with inbox and DLQ.
  3. Scheduled job on several replicas.
  4. New entity + migration.
  5. External REST API integration.
  6. New gRPC method.
  7. New/changed release pipeline and artifact promotion.
  8. New persistent state or changed recovery behavior.
  9. Critical journey reliability/load acceptance.
  10. Frontend feature end to end (React / Blazor): contract → generated client → screens → i18n → tests.

## Definition of Done (every recipe)
Tests, build/test run, review and docs follow `task-lifecycle` steps 5–8. In addition, every recipe checks:
- [ ] Security: authorization policy chosen explicitly, resource owner taken from claims (`security`).
- [ ] Observability: structured logs, metrics/traces for new I/O, runbook for new alerts (`observability`, `project-documentation`).
- [ ] Configuration: new options class validated on start, local/Aspire values added, no real secrets (`configuration-secrets`).
- [ ] When the change affects a critical journey, its applicable SLI/SLO, load/failure criteria and release signals are updated ([reliability-and-performance](../reliability-and-performance/SKILL.md)). New or materially changed persistent state has a recovery path/objectives and applicable drill evidence ([data-recovery](../data-recovery/SKILL.md)).
- [ ] Delivery-impacting changes update the gate/manifest/promotion contract ([delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md)); documentation-only changes do not need an image build or unrelated operational exercise.
