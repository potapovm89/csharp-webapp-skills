# Git Branching & Commits Reference

## Branch Types
- **Simple Task**:
  - `feature/<task-number>-<short-description>`
  - `bugfix/<task-number>-<short-description>`
- **Complex Epic**:
  - Epic parent branch: `feature/<epic-task-number>-<epic-name>`
  - Subtask branches: branched off the epic parent branch.

## Commit Message Format
- Must include the task identifier and description of the delta:
  ```
  PROJ-102: add endpoint for creating order batch
  PROJ-102: add validator tests for order batch request
  ```
- Make frequent, small, cohesive commits rather than monolithic dumps.

## Non-Negotiable Git Rules
- Direct push to `main` is prohibited.
- Modifying published history (`git rebase` on shared branches, `git push --force`) is strictly forbidden.

## Review & Merge Policy
- Every MR requires at least one approval from a person who is not the author. Changes touching `security`, migrations, release gates/pipeline, or the WorkerHost clustering choice additionally require the designated reviewer for that area (recorded in the project's review policy or `docs/delivery/pipeline.md`).
- Review SLA: first response (approval, change request or an explicit question) within one business day. Long MRs get an early draft review — open as `Draft` when the change exceeds roughly 400 lines.
- The merge method is chosen once per project (squash with a single task-referencing message, or a merge commit) and recorded in the project's git policy; whichever is chosen, the merged history keeps the task identifier.
- Rebase on your own unpublished branch is fine; rebasing a branch that is under review requires the reviewer's agreement, and published history is never rewritten (`AGENTS.md`).
