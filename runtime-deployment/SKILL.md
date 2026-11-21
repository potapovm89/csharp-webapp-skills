---
name: runtime-deployment
description: >-
  Use this skill when preparing a host to run in containers or Kubernetes: Dockerfile, health probes (/health, /alive), resource requests and limits, graceful shutdown, running database migrations before a deploy, replicas and PodDisruptionBudget, rolling updates without downtime. Triggers (RU): деплой, Dockerfile, Kubernetes, k8s, пробы, liveness, readiness, graceful shutdown, лимиты ресурсов, реплики.
---

# Runtime & Deployment

This skill covers draining, retries and recovery during rolling updates, scaling and restarts. In-flight requests may fail or have an unknown outcome; idempotent retries and durable messaging prevent duplicate business effects and recover unfinished work within the documented durability assumptions. Pipeline gates and artifact promotion belong to [delivery-and-supply-chain](../delivery-and-supply-chain/SKILL.md); business acceptance/observation criteria to [reliability-and-performance](../reliability-and-performance/SKILL.md); restoring persistent state to [data-recovery](../data-recovery/SKILL.md).

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
- [ ] `docker build` of at least one image passes locally or in CI.

## References
- [containers.md](./references/containers.md) — Dockerfile template, non-root, fonts/ICU, image tags.
- [kubernetes.md](./references/kubernetes.md) — probes, resources, graceful shutdown per host type, migrations Job, PDB, rolling update settings.
