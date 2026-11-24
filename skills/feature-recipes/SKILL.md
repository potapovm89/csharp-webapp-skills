---
name: feature-recipes
description: >-
  Use this skill for cross-layer feature checklists (REST, consumers, jobs, entities, external APIs and gRPC), delivery/recovery/reliability integration recipes, or reference project-helper signatures. Triggers (RU): добавь эндпоинт, новый консьюмер, новая джоба, новая сущность, новая интеграция, сделай фичу целиком, сквозной рецепт, чеклист.
---

# Feature Recipes (End to End)

Before starting, use [task-lifecycle](../task-lifecycle/SKILL.md) to confirm scope and inputs. Select the recipe that matches the requested outcome. Follow its steps in order and mark a conditional step as skipped only when its condition is false. If an applicable step depends on a missing project decision or check, record it as pending.

Reference helper signatures are in [project-helpers.md](./references/project-helpers.md). Confirm the current solution's API before implementing a call; code determines signatures, while these skills determine required behavior. An older API or template omission does not waive security, transaction or delivery requirements: adapt the implementation or report the needed change.

## Choose a Recipe

| Requested outcome | Recipe and conditions |
|---|---|
| REST endpoint | [REST endpoint and outbox](./references/recipes.md#1-rest-endpoint-write-to-the-db--publish-an-event). Use the outbox and broker contract steps only when a write must publish an event. |
| Consume a broker message | [Kafka or RabbitMQ consumer](./references/recipes.md#2-broker-consumer-kafka--rabbitmq). Use the configured broker and the existing deduplication scheme. |
| Scheduled work | [Scheduled job](./references/recipes.md#3-scheduled-job-on-several-replicas). Use the documented replica strategy and its concurrency test. |
| New persisted entity | [Entity and migration](./references/recipes.md#4-new-entity--migration). A DTO-only change has no migration step. |
| External REST API | [External API integration](./references/recipes.md#5-external-rest-api-integration). Verify the effective retry policy for state-changing calls. |
| New gRPC method | [gRPC method](./references/recipes.md#6-new-grpc-method). Use the project's service boundary and wire-compatibility rules. |
| Change pipeline or promotion | [Release pipeline](./references/recipes.md#7-release-pipeline-and-artifact-promotion). Use the affected stages and checks. |
| Add or materially change persistent state | [Recovery](./references/recipes.md#8-persistent-state-and-recovery). Reuse the existing plan when it already covers the change. |
| Change a critical journey, its capacity or failure behavior | [Reliability](./references/recipes.md#9-critical-journey-reliability). Reuse applicable objectives and verify the changed behavior. |
| Frontend feature | [React or Blazor feature](./references/recipes.md#10-frontend-feature-react--blazor). Regenerate the API client when the API operation changes. |

**Example from `it-letter`** — the delivery processor in WorkerHost calls a Story through `IStoryExecutor`. The Story calls the domain service that claims rows with `SKIP LOCKED`. For this recipe, verify that two workers claim disjoint batches and that retrying does not duplicate a business effect.

## Completion Evidence

Record the selected recipe, why each skipped step does not apply, and the test command and result for the requested behavior and its relevant negative case. For example, a scheduled-job test for `SKIP LOCKED` must show that concurrent claims have no overlapping IDs. Follow `task-lifecycle` steps 5–8 for build, tests, review and docs. Check these conditions when applicable:
- [ ] New or changed access boundary: authorization policy chosen explicitly and resource owner derived from the authenticated principal (`security`).
- [ ] New I/O or alert: structured logs and relevant metrics/traces, plus a runbook for a new alert (`observability`, `project-documentation`).
- [ ] New configuration: options validated on start and required local/Aspire values added without real secrets (`configuration-secrets`).
- [ ] When the change affects a critical journey, its applicable SLI/SLO, load/failure criteria and release signals are updated ([reliability-and-performance](../reliability-and-performance/SKILL.md)). New or materially changed persistent state has a recovery path/objectives and applicable drill evidence ([data-recovery](../data-recovery/SKILL.md)).
- [ ] Delivery-impacting changes update the gate/manifest/promotion contract ([delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md)); documentation-only changes do not need an image build or unrelated operational exercise.
