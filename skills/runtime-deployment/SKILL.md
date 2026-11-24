---
name: runtime-deployment
description: >-
  Use this skill when preparing a host to run in containers or Kubernetes: Dockerfile, health probes (/health, /alive), resource requests and limits, graceful shutdown, running database migrations before a deploy, replicas and PodDisruptionBudget, rolling updates without downtime. Triggers (RU): деплой, Dockerfile, Kubernetes, k8s, пробы, liveness, readiness, graceful shutdown, лимиты ресурсов, реплики.
---

# Runtime & Deployment

This skill covers draining, retries and recovery during rolling updates, scaling and restarts. In-flight requests may fail or have an unknown outcome; idempotent retries and durable messaging prevent duplicate business effects and recover unfinished work within the documented durability assumptions. Pipeline gates and artifact promotion belong to [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md); business acceptance/observation criteria to [reliability-and-performance](../reliability-and-performance/SKILL.md); restoring persistent state to [data-recovery](../data-recovery/SKILL.md).

## Verify the Affected Host

1. Identify each runnable host or Job affected by the change. For each, locate its project, Dockerfile, build context and deployment entry. For shared build changes, check the affected consumers of that shared build.
2. Use the project's image build command for each affected image. A repository-root context can use `docker build -f <host-Dockerfile> -t <local-test-tag> <repo-root>`. For Kubernetes changes, render the target overlay or chart with the project's configured tool and values. Compare the rendered image, probe paths and ports, resources and applicable Job dependencies with the host code and release contract.
3. In an authorized local or non-production environment, exercise the affected startup, probe or rollout behavior. For a changed migration path, verify that the release uses the single pre-deploy Job and that a failed Job blocks the new rollout.

**Example from `it-letter`** — WorkerHost and the EF migration runner have separate Dockerfiles, and WebHost declares `/alive` for liveness and `/health` for readiness. A successful WebHost build only covers that host. When changing WorkerHost or the migration Job, verify their own images and rendered workload entries.

## Core Rules
1. **One image per host**, multi-stage Dockerfile on the official `mcr.microsoft.com/dotnet/aspnet:10.0` runtime image, non-root user (`USER $APP_UID`), no SDK and no secrets in the final image. See [containers.md](./references/containers.md). Every runnable host, including the migrations runner, has its `Dockerfile` next to its `.csproj` from the MR that creates the host; when a solution is created from the template, Dockerfiles of kept hosts are kept and missing ones added.
2. **Probes** come from `MapDefaultEndpoints(port)` on a separate management port:
   - liveness → `/alive` (process is responsive; no dependency checks);
   - readiness → `/health` (dependencies the host cannot work without);
   - startup probe for slow starts instead of a long liveness delay.
3. **Graceful shutdown**: the host stops accepting work, finishes in-flight requests/messages, commits offsets/acks, and exits within `terminationGracePeriodSeconds`. `HostOptions.ShutdownTimeout` is shorter than the grace period.
4. **Migrations run once, before the new version starts** (a single pre-deploy Job with the migrations project, not one init container per application pod). They follow `database-migrations` zero-downtime rules, because old and new versions run side by side during a rollout. A failed Job blocks the rollout in `delivery-and-supply-chain`.
5. **Resources**: every container sets CPU/memory requests and a memory limit; the GC adapts to the container limit — do not hardcode heap sizes.
6. **Replicas**: stateless hosts (WebHost, RpcHost, consumers) run ≥ 2 replicas with a PodDisruptionBudget; WorkerHost follows its documented clustering choice (`worker-host-jobs`).
7. **Configuration** comes from environment variables and mounted secrets (`configuration-secrets`); the image is identical for all environments.
8. **Rollout acceptance** includes the documented business/SLI observation window, not only probes. Deploy the exact verified digest with recorded non-secret configuration; rollback requires a compatible schema and the prior verified digest. Restore/replay is a separate operator decision, never an automatic rollout fallback.

## Adopting in an Existing Solution
When bringing a solution to these rules, check every item; copying Dockerfiles from another repository is not enough:
- [ ] `global.json` pins the SDK version (`sdk.version` + `rollForward`), not only test-runner settings.
- [ ] The solution is a single `.slnx` (legacy `.sln` migrated with `dotnet sln migrate` and removed); all projects are SDK-style `.csproj` without `packages.config` (`solution-architecture`).
- [ ] `Directory.Build.props`: `RestorePackagesWithLockFile=true`; `RestoreLockedMode` in CI. Every project has a committed `packages.lock.json`.
- [ ] `Directory.Packages.props`: `CentralPackageTransitivePinningEnabled=true`.
- [ ] Each Dockerfile copies `global.json`, `Directory.Build.props`, `Directory.Packages.props`, shared configs and sources, then runs `dotnet restore ... --locked-mode` and `publish --no-restore`.
- [ ] Every runnable application has an image: all hosts, the migrations runner, tools run as Jobs, and the SPA frontend (see containers.md, "Frontend (SPA) image").
- [ ] A host image does not drag another runnable project's `appsettings*.json` in (publish fails with NETSDK1152); reference such projects from tests instead.
- [ ] Each affected image builds locally or in CI using its actual Dockerfile and build context.

## References
- [containers.md](./references/containers.md) — Dockerfile template, non-root, fonts/ICU, image tags.
- [kubernetes.md](./references/kubernetes.md) — probes, resources, graceful shutdown per host type, migrations Job, PDB, rolling update settings.

## Completion Evidence

- For each affected image, record the Dockerfile, build context, successful command or CI job, and resulting local image ID or CI digest. For manifest changes, link the rendered workload and record the affected fields checked against the host and its image.
- For a running service with a required dependency, verify that `/alive` and `/health` pass when healthy. When exercising a dependency outage that leaves the process running, `/health` must fail and `/alive` must still pass. For a one-shot Job, check its exit status and applicable rollout gating instead. [Kubernetes probe behavior](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- For a changed rollout, record the applicable migration result, rollout outcome and observation evidence for the same candidate. Report unavailable checks and the scope of any configuration-only verification.
