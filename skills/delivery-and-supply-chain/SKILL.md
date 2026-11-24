---
name: delivery-and-supply-chain
description: >-
  Use this skill when creating or changing CI/CD pipelines, release gates, dependency lock files, SBOM, build provenance, artifact signing or verification, and promotion or rollback of release artifacts. Triggers (RU): CI/CD, пайплайн, сборка релиза, поставка, цепочка поставки, SBOM, подпись образа, provenance, продвижение артефакта.
---

# Delivery & Software Supply Chain

Define a verifiable path from reviewed source to a released artifact. Keep the project's CI platform, .NET 10 architecture and package/license policy; do not select a new platform or signing service without a reason and the required authorization. Writing pipeline configuration does not authorize publishing, deploying or accessing production; the prohibition in `AGENTS.md` remains in force.

## When to Use (and When Not)
- **Use** when changing CI/CD configuration, release gates, lock files, SBOM/provenance/signing or promotion and rollback rules.
- **Skip** for ordinary feature code: it only has to pass the existing gates.

## Quick Checklist
1. Restore with committed lock files. Use `dotnet restore --locked-mode` for .NET and the frozen install matching the frontend lockfile ([react-frontend](../react-frontend/SKILL.md)).
2. Build and test once; build one image per host and record its digest.
3. Scan dependencies and the image; a missing or stale required result is a failure, not a pass.
4. Promote the same digest between environments; never rebuild for an environment.
5. Keep the previous verified digest and a compatible schema for rollback.

## Workflow
1. Inspect the existing pipeline, protected branches/environments, runner trust boundaries, SDK/tool versions, registries and package sources. Identify the release owner and the source revision being built; do not infer that an MR pipeline has release permissions.
2. Before changing gates, read [pipeline-gates.md](./references/pipeline-gates.md). For each changed stage or gate, map the entry in `docs/delivery/pipeline.md` to the actual CI job or script and any rendered deployment manifest. Record its trigger, dependencies, command, blocking condition and evidence artifact. If an operator manages the stage externally, link the authoritative configuration or record it as unverified. Mark a planned stage explicitly.
3. Restore a reviewed dependency graph, then build/test against that graph. Build the release image once per host, including migration runners; scan and promote the same digest through environments. A rebuild, even from the same commit, creates a new candidate requiring verification.
4. Produce a release manifest binding source revision, pipeline identity, artifact digests, dependency inventory/SBOM, scan results and provenance. Select and verify an authenticity policy appropriate to the threat model; before implementing signing or promotion, read [artifact-trust.md](./references/artifact-trust.md).
5. In an authorized non-production environment, verify the artifact and its gates, run the migration Job once, roll out and observe functional checks and the signals defined by `reliability-and-performance`. Promotion stops on failed or unavailable required evidence; do not mask a skipped/failed gate as success.
6. Record the prior verified digest and compatible configuration/schema for rollback. Rolling back code is distinct from restoring data (`database-migrations`, `data-recovery`). Changes to a release candidate's content or configuration require the applicable checks again.

**Example from `it-letter`** — the pipeline document calls `/health` liveness and `/alive` readiness. The WebHost manifest uses liveness → `/alive` and readiness → `/health`. When reviewing the contract, compare both probe fields and their ports against the rendered manifest and host configuration. Record the discrepancy and make the scoped correction at its source.

## Required Boundaries
- `git-workflow` owns branch/MR conventions; `testing` owns test design; `security` and `csharp-code-style` own vulnerability/license policy. This skill wires their results into enforceable gates.
- `runtime-deployment` owns Docker/Kubernetes behavior; this skill owns artifact identity, stage dependencies and promotion decisions.
- `configuration-secrets` owns secret handling. Untrusted MR code never receives signing, production or release-registry write credentials; any approved dependency-download credentials are separately scoped/read-only. Deployment jobs consume verified artifacts instead of rebuilding repository code with privileged credentials.
- Release checks have bounded timeouts/retries. A failed migration, authenticity check, expired exception or failing rollout stops progression; choose a diagnosis/rollback path instead of repeatedly rerunning a privileged action.

## Completion Evidence
- [ ] Every affected documented gate matches the effective CI stage and deployment configuration, or has an explicit planned/unverified status. Record the source paths and the fields compared.
- [ ] For an enforcement change, record an isolated passing candidate and the rejection cases for the affected gates. For promotion, verify that missing required evidence and evidence for a different digest both block promotion. For dependency changes, verify the relevant lock-file or policy rejection.
- [ ] Link the CI run and exact candidate digest to the test and scan results, release manifest and any authorized non-production rollout. Record rollback conditions, approvers and current time-bound exceptions. Keep credentials and personal data out of evidence.
- [ ] For a documentation-only change, verify the affected statements against configuration and label unavailable execution evidence. Report every applicable check that could not run.

Design references, reviewed 2026-11-14: [NIST SSDF 1.1](https://csrc.nist.gov/pubs/sp/800/218/final) and [SLSA 1.2](https://slsa.dev/spec/v1.2/tracks). These are reference frameworks, not a claim of certification or a mandatory SLSA level.
