---
name: git-workflow
description: >-
  Use this skill when creating or naming Git branches, drafting commit messages, preparing Merge Request (MR) descriptions, or reviewing Git procedures and safety rules.
  Triggers (RU): ветка, коммит, merge request, MR, git.
---

# Git Workflow & Pull Requests

This skill outlines Git branching conventions, commit standards, and merge request requirements for the solution.

## Core Principles
1. **Never Rebase Published Commits / Never Force Push**: Rewriting public Git history is strictly forbidden.
2. **Branch Naming**:
   - `feature/<task-id>-<slug>`
   - `bugfix/<task-id>-<slug>`
3. **Commit Messages**: Must reference `<task-id>: <summary of changes>`. Small, logical commits are preferred over bulk commits.
4. **Merge Requests**: Every merge request requires peer code review and passing CI tests before merging into `main`.

## Subtopic References
- [branching.md](./references/branching.md) — Branch hierarchies for simple and complex tasks, commit message standards.
- [mr-templates.md](./references/mr-templates.md) — Merge request template and pre-submission checklist.
