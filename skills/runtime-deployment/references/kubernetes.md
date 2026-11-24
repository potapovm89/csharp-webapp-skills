# Kubernetes Runtime Reference

## Probes

```yaml
ports:
  - name: http
    containerPort: 8080
  - name: management
    containerPort: 8081          # port passed to MapDefaultEndpoints(port)
startupProbe:
  httpGet: { path: /alive, port: management }
  periodSeconds: 5
  failureThreshold: 30          # up to 150 s to start
livenessProbe:
  httpGet: { path: /alive, port: management }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /health, port: management }
  periodSeconds: 5
  failureThreshold: 3
```

- `/alive` checks only the process (the `live` tag). A failing database must not restart the pod — that only adds load.
- `/health` includes hard dependencies (database, broker connection). Optional dependencies (cache) are reported as `Degraded`, not `Unhealthy`.
- Probe endpoints are anonymous and served on the management port, which is not exposed through the ingress.

## Resources

```yaml
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { memory: 512Mi }        # CPU limit optional; throttling hurts latency
```

- Size from load test results and memory metrics, not guesses; revisit after big changes.
- .NET respects the memory limit (GC heap hard limit ≈ 75% of it). If memory is tight, try `DOTNET_GCConserveMemory` or workstation GC (`<ServerGarbageCollection>false</ServerGarbageCollection>`) — only with measurements.

## Security Context

```yaml
securityContext:
  runAsNonRoot: true                  # matches the image's USER $APP_UID
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
  readOnlyRootFilesystem: true
volumeMounts:
  - { name: tmp, mountPath: /tmp }
volumes:
  - { name: tmp, emptyDir: {} }
```

- .NET needs a writable `/tmp` (culture/ICU caches, Data Protection key generation) — mount an `emptyDir` instead of opening the root filesystem.
- Data Protection keys belong on a persistent or secret-backed key ring, not only in the container filesystem (`configuration-secrets`).
- These are the host defaults; if the platform enforces an admission policy (PSA/PSS baseline or restricted), reconcile with it instead of maintaining a parallel scheme.

## Network Policy (default deny)

```yaml
kind: NetworkPolicy
spec:
  podSelector: { matchLabels: { app: webhost } }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:                                   # public traffic through the ingress controller
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: ingress } }
      ports: [ { port: 8080 } ]
    - from:                                   # metrics scraping on the management port
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: monitoring } }
      ports: [ { port: 8081 } ]
  egress:
    - to:                                     # DNS: without it every other rule fails on name resolution
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports: [ { port: 53, protocol: UDP }, { port: 53, protocol: TCP } ]
    - to: [ { podSelector: { matchLabels: { app: postgres } } } ]
      ports: [ { port: 5432 } ]
    - to: [ { podSelector: { matchLabels: { app: kafka } } } ]
      ports: [ { port: 9092 } ]
    # Keycloak, Redis, S3 — allow-list per dependency
```

- The example is for WebHost. Each host gets its own policy: RpcHost accepts traffic only from the calling hosts' pods (`podSelector`), consumers and WorkerHost usually need no ingress besides metrics.
- Ingress reaches only the application port and, from the monitoring namespace, the management port; the management port is never exposed through the ingress. Label selectors (`ingress`, `monitoring`, `kube-dns`) differ between platforms — take them from the cluster.
- Egress is allow-listed per dependency: deny-by-default mirrors the application's fallback authorization policy (`security`).
- Start from the platform's baseline policy if one exists; extend it per host instead of inventing a second scheme.

## Graceful Shutdown
Kubernetes sends SIGTERM and removes the pod from endpoints at the same time, so requests can still arrive for a few seconds.

```yaml
terminationGracePeriodSeconds: 45
lifecycle:
  preStop:
    sleep: { seconds: 5 }   # native sleep action: no sleep binary needed in the image
```

- The `sleep` action (`PodLifecycleSleepAction`) is enabled by default since Kubernetes 1.30 (beta) and is stable in the KEP since 1.34. On older clusters use `exec: { command: ["sleep", "5"] }` — it needs a `sleep` binary, which chiseled/distroless .NET images do not have. Record the actual cluster version in `docs/delivery/pipeline.md`.

```csharp
builder.Services.Configure<HostOptions>(options => options.ShutdownTimeout = TimeSpan.FromSeconds(30));
```

Per host type:
- **WebHost / RpcHost**: Kestrel stops accepting connections and drains in-flight requests; long streaming calls observe `ApplicationStopping`.
- **Kafka consumer**: stop consuming, finish the current message, store/commit its offset, then `Close()` the consumer so the group rebalances immediately.
- **RabbitMQ consumer**: cancel the consumer (`BasicCancel`), finish in-flight deliveries and ack them; unacked messages return to the queue — safe because consumers are idempotent.
- **WorkerHost**: Quartz `WaitForJobsToComplete = true`; jobs check the cancellation token between items.
- **Outbox relay**: finish the current batch; unsent rows stay in the table.

## Migrations Before Deploy
- A Kubernetes Job (or Helm pre-upgrade hook) runs the migrations project image with the target connection string; the rollout starts only after it succeeds.
- The Job is idempotent (applied migrations are skipped) and has `backoffLimit` ≤ 2; a failed Job stops the release — follow the failed migration runbook.
- Hosts never call `Database.Migrate()` on startup in shared environments.
- Expand/contract rules (`database-migrations`) guarantee that the previous version keeps working against the new schema during the rollout and on rollback.

## Replicas & Rollouts

```yaml
replicas: 2
strategy:
  rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 1
  selector: { matchLabels: { app: <host> } }
```

- Spread replicas across nodes explicitly:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: <host> } }
```

- Rollback selects the prior verified image digest and compatible non-secret configuration/schema; never assume an old tag still resolves to the tested bytes. `delivery-and-supply-chain` owns verification and promotion/rollback gates, while `reliability-and-performance` defines the bounded observation and hold criteria. A failed release does not authorize a data restore or destructive migration rollback (`data-recovery`).

## Horizontal Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: webhost }
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

- Autoscale stateless hosts (WebHost, RpcHost) on CPU or a custom latency/queue metric; for consumers prefer lag/queue depth (KEDA or an equivalent scaled-object metric) — Kafka replicas beyond the partition count add nothing (`worker-host-jobs`, clustering.md).
- Every autoscaling decision re-checks the WorkerHost choice: Quartz in-memory store with `replicas > 1` is the configuration `worker-host-jobs` forbids.
- Scaling limits and their load-test evidence are recorded with the capacity notes of the affected journey (`reliability-and-performance`).
