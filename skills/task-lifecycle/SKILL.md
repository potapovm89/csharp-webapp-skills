---
name: task-lifecycle
description: >-
  Use this skill when receiving, planning, executing, testing, reviewing, or completing any coding or refactoring task in the project. Covers confidence check, execution plan, sub-agent delegation, testing verification, code review, and final summary.
  Triggers (RU): задача, план, порядок работы, ревью, проверка готовности.
---

# Task Lifecycle Workflow

This skill defines the end-to-end process for tackling any task in the project.

## Step-by-Step Task Plan

1. **Understand & Confidence Check**
   - Read the task requirements and verify acceptance criteria.
   - Identify which domain areas are touched (architecture, DB, API, brokers, etc.) and consult the corresponding skills.
   - For a changed critical journey, establish its reliability/load/failure acceptance criteria (`reliability-and-performance`); for new or materially changed persistent state, check recovery scope/objectives (`data-recovery`). Reuse approved objectives; unresolved material targets require a decision before claiming acceptance.
   - Ask before dependent work when a material decision cannot be resolved from the task, code or existing authorization:
     - a missing, ambiguous or contradictory acceptance criterion changes the observable outcome;
     - the change alters a public contract (REST, gRPC, Kafka/RabbitMQ message, DB schema) in a way that may break existing consumers;
     - data deletion, a non-reversible migration or a new dependency requires authorization that has not already been given;
     - alternatives differ materially in public behavior, architecture, cost, security or licensing, and no existing decision selects one;
     - you could not find the existing code or pattern the task refers to.

2. **Formulate the Plan**
   - Decompose complex tasks into subtasks.
   - Present alternative solutions and tradeoffs when applicable.
   - Always verify the plan against project architecture and strict rules.
   - Share a concise plan for substantial work. Proceed with authorized, reversible changes; ask for approval only for a material decision or action that needs it. An approved report or explicit implementation request already authorizes its scoped corrections; do not ask for the same approval again. Choose routine implementation details consistently with the project.

3. **Branching & Git Preparation**
   - Inspect the current branch/worktree and local changes first; preserve unrelated work. Reuse a suitable checkout. When a new branch is needed, create it from the agreed base after a safe update; never switch/reset a dirty checkout or overwrite another task's work:
     - Simple task: branch `feature/<task-number>-<short-description>` or `bugfix/<task-number>-<short-description>`.
     - Complex epic: create parent branch `feature/<task-number>-<epic-name>`, subtasks branch from it.
   - See `git-workflow` skill for naming details.

4. **Implementation**
   - Adhere strictly to the layer boundaries:
     - Hosts are thin entry points (controllers, consumers, jobs, gRPC services).
     - Business logic resides strictly in `Stories` (CQRS / MediatR or Executor Stories).
     - Data access is encapsulated in `DataAccess` domain services (`I<Entity>DomainService`).
     - Entities, DbContext, configurations, and DTOs reside in `Domain` and `Contracts`.
   - Any database schema changes must be done via migrations (never edit existing applied migrations; never alter DB manually).
   - Keep changes in small, logical steps. Commit/push only when authorized by the task or workflow; commit messages contain the task number when one exists.

5. **Write Tests**
   - Match verification to changed behavior and risk. Reuse existing coverage where it proves the outcome; documentation-only or cosmetic changes need relevant validation rather than implementation-mirroring tests.
   - Unit tests: in `tests/Itl.<Project>.UnitTests` covering all FluentValidation validators, mapping logic, and business branches using table-driven test cases (`BaseTestCasesClass`).
   - Integration tests: in `tests/Itl.<Project>.IntegrationTests` covering endpoints and consumers against Testcontainers.

6. **Verify & Run**
   - For application code changes, run `dotnet build --no-incremental` — ensure 0 errors and 0 warnings. Incremental builds may skip unchanged projects and their analyzers, so a clean-looking incremental result is insufficient.
   - Run the relevant tests and required repository gates; report what actually ran. Broaden testing when failures, shared-code changes or unresolved risks justify it. For skill/documentation-only edits, validate structure, links and changed examples; do not claim application tests ran.
   - If UI components are created or modified, verify visual appearance and functional flow.
   - If tests or runs fail: analyze the root cause, fix issues, or ask for clarifications if external factors are involved.
   - Pipeline/release changes follow `delivery-and-supply-chain`: validate stage dependencies and rejection cases, record candidate identity/evidence and check release observation criteria. Recovery/performance changes use the relevant isolated drill/load evidence; they do not authorize production execution or require a full drill for every unrelated edit.

7. **Review & Audit**
   - Conduct a code review of the changes (or spawn a subagent in a clean session) to check against code-style, prohibited libraries, XML-documentation, and the review checklist of the `security` skill.
   - Migrations: check the MR checklist in `database-migrations` (zero-downtime). Jobs: check how they behave with several WorkerHost replicas (`worker-host-jobs`).

8. **Documentation & Merge Request**
   - Update documentation in `docs/` when observable behavior, contracts, schema, settings, permissions or architecture change. A cosmetic/internal edit with no documentation impact does not require an artificial docs change:
     - `docs/contracts/` for API, gRPC, Kafka, RabbitMQ changes.
     - `docs/database/` (.dbml) for database changes.
     - `docs/settings/` for new configuration options.
     - `docs/roles/` for permissions/endpoints.
     - `docs/delivery/`, `docs/reliability/` and `docs/recovery/` for affected release gates, SLI/SLO/capacity or recovery policies/evidence.
   - Prepare a Merge Request description with the mandatory template:
     - What was done
     - Task reference
     - How to test / verification steps
     - Non-code changes (migrations, contracts, settings, docs)
   - Prepare a final summary report for the user detailing the accomplished work.
