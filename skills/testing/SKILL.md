---
name: testing
description: >-
  Use this skill when writing, running, or fixing unit tests (xUnit, Moq, FluentAssertions up to 7.2.0, BaseTestCasesClass) or integration tests with Testcontainers and WebApplicationFactory.
  Triggers (RU): тесты, юнит-тесты, интеграционные тесты, Testcontainers, архитектурные тесты, контрактные тесты, флаки-тесты.
---

# Testing (Unit & Integration Tests)

This skill covers the testing practices and infrastructure for the solution.

## Core Rules
1. **Verify Changed Behavior**: Add or update meaningful tests for changed behavior and risks; existing tests may already provide the evidence. Documentation/formatting-only edits need appropriate validation, not tests that mirror implementation. Run required project gates.
2. **Library Version Cap**: `FluentAssertions` is strictly restricted to **≤ 7.2.0**.
3. **Naming Convention**: `[TestedMethod]_[Condition]_[ExpectedResult]`.
4. **Arrange / Act / Assert**: Every test must clearly follow this three-phase structure.

## Unit Tests (`tests/Itl.<Project>.UnitTests`)
- Fast, in-memory, zero network or DB dependencies.
- Table-driven validator tests inherit from `BaseTestCasesClass` and `BaseInvalidRequest` with a clear `Because` field.
- Cover all validation rules (valid, invalid and boundary inputs) and material business branches, with particular attention to authorization, money, concurrency and failure handling. Coverage metrics reveal gaps but do not replace scenario assertions or require tests of trivial implementation details.
- See [unit-testing.md](./references/unit-testing.md).

## Integration Tests (`tests/Itl.<Project>.IntegrationTests`)
- Run against real containerized services via Testcontainers (PostgreSQL, Kafka, RabbitMQ, Redis, RustFS for S3, MongoDB).
- Spin up hosts via `WebApplicationFactory<Program>`.
- Tests verify end-to-end side effects (DB state, emitted queue messages, cached items).
- Broker-related test cases (duplicate delivery, poison message → DLQ, outbox row in the same transaction) are listed in the checklists of `message-brokers` (handlers.md, outbox.md).
- See [integration-testing.md](./references/integration-testing.md).

## Determinism
- Tests never depend on the machine: time comes from `FakeTimeProvider`, culture is pinned (module initializer), random data is seeded or built by test data builders.
- Database state is reset between tests (Respawn or a fresh database per fixture), never by test order.
- A flaky test is quarantined with an owner and fixed at the root — never retried-until-green; the policy is in [test-data-and-determinism.md](./references/test-data-and-determinism.md).
- See [test-data-and-determinism.md](./references/test-data-and-determinism.md).

## Architecture & Contract Tests
- Layer dependencies are checked by tests (`tests/Itl.<Project>.ArchitectureTests`) so violations fail the build. The project is mandatory: if the solution does not have it, create it in the first MR that adds or changes a project reference.
- Minimum rule set: the dependency rules of `solution-architecture`; hosts do not reference `DataAccess` types outside DI registration; only one Stories mechanism is referenced (`cqrs-stories`); no prohibited packages; mapping classes only in `Common/Mappers`.
- The minimum set also covers package license caps from committed lock files (NuGet versions, transitive included) and direct system clock reads; every rule has a negative fixture that must fail. Tests copied from another repository are re-checked against the list, not trusted.
- Public contracts (REST OpenAPI, `.proto`, broker messages) have change-detection snapshots plus compatibility verification; updating a snapshot is not proof of compatibility.
- See [architecture-and-contract-tests.md](./references/architecture-and-contract-tests.md).

## Delivery, Recovery and Reliability Verification
- [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md) wires tests/scans into gates bound to the candidate's source and artifact digest. Test gate rejection with isolated fixtures; a report that exists or a scanner that exited successfully is not automatically a passed policy.
- [reliability-and-performance](../reliability-and-performance/SKILL.md) owns the load/failure acceptance contract, SLI query fixtures and burn-rate alert tests. Use approved non-production targets and bounded fault/load injection; Testcontainers correctness tests do not prove production capacity.
- [data-recovery](../data-recovery/SKILL.md) owns restore/PITR and cross-store reconciliation drills, including measured RPO/RTO. Run on the agreed cadence and material recovery changes, separately from ordinary MR tests; a successful migration on an empty DB is not a restore test. Label tabletop/fixture/realistic-volume evidence accurately.
