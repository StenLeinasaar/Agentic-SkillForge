# Architecture & Review Checklists

Pull specific items from here during design reviews or manifest/chart reviews. Don't recite the whole list — match it against what's actually in front of you and flag what's missing or wrong, grouped by severity.

## Production-Readiness Checklist

**Scheduling & availability**
- [ ] Resource `requests` set on every container (scheduling depends on this; omitting it causes bin-packing problems and noisy-neighbor risk)
- [ ] Resource `limits` set deliberately — especially memory (CPU limits are more debatable; unthrottled CPU is often fine, but memory limits without OOM awareness cause surprise kills)
- [ ] `replicas` > 1 for anything meant to survive a node loss
- [ ] `PodDisruptionBudget` set for workloads that must maintain availability during voluntary disruptions (node drains, cluster upgrades)
- [ ] Topology spread constraints or anti-affinity so replicas aren't all on one node/zone
- [ ] `readinessProbe` present and actually reflects app readiness (not just "process is running")
- [ ] `livenessProbe` present but not so aggressive it kills slow-but-healthy pods — pair with `startupProbe` for slow-starting apps
- [ ] Graceful shutdown handled: app responds to SIGTERM and `terminationGracePeriodSeconds` is long enough for in-flight requests to drain

**Images & deployment**
- [ ] No `:latest` tag in production manifests — pin to a specific tag or digest
- [ ] Image pull policy appropriate for the tagging strategy (`IfNotPresent` is wrong if you reuse mutable tags)
- [ ] Rollout strategy (`maxUnavailable`/`maxSurge`) matches the workload's tolerance for reduced capacity during deploys
- [ ] Rollback path is known/tested, not just theoretical

**Security**
- [ ] Containers not running as root unless genuinely required (`runAsNonRoot`, explicit `runAsUser`)
- [ ] `readOnlyRootFilesystem` where the app doesn't need to write to its own filesystem
- [ ] No privileged containers / unnecessary `hostNetwork`, `hostPID`, `hostPath` mounts
- [ ] Secrets mounted, not baked into images or plaintext env vars in manifests committed to git
- [ ] `NetworkPolicy` restricting traffic to what's actually needed, not relying on flat-network-by-default
- [ ] RBAC scoped to least privilege — service accounts shouldn't default to broad ClusterRoles
- [ ] `automountServiceAccountToken: false` on pods that don't need to talk to the API server

**Observability**
- [ ] Structured logs going somewhere durable (not just relying on `kubectl logs` retention)
- [ ] Metrics exposed and actually scraped (Prometheus annotations/ServiceMonitor wired up, not just present in code)
- [ ] Alerting tied to symptoms users would notice (latency, error rate, saturation), not just infra-level noise
- [ ] Distributed tracing if the request path crosses multiple services and debugging latency matters

## Scaling Checklist

- [ ] HPA configured on the right metric (CPU is a poor proxy for I/O-bound or queue-driven workloads — consider custom metrics)
- [ ] HPA min/max bounds sane for actual traffic patterns and cost tolerance
- [ ] Cluster autoscaler (or Karpenter on AWS) present if node-level scaling is needed, not just pod-level
- [ ] Resource requests accurate enough that autoscaling decisions aren't wildly wrong (over-requested = wasted capacity triggers scale-out too early; under-requested = real pressure hidden)
- [ ] Stateful components (databases, queues) scaling strategy considered separately — HPA on a StatefulSet needs real thought, not a default assumption

## Architecture Decision Prompts

Use these as the "ask before recommending" list — only the ones that actually change the answer for the situation at hand:
- Expected traffic pattern: steady, bursty, or spiky-predictable (e.g. batch jobs, cron-like load)?
- Team's current Kubernetes maturity — will they operate a service mesh, or is that overhead not worth it yet?
- Multi-tenancy: shared cluster with namespace isolation, or dedicated clusters per team/environment?
- Compliance/security requirements that mandate specific isolation (network policies, dedicated node pools, encryption at rest)?
- Existing GitOps/CI-CD tooling already in place — don't recommend introducing ArgoCD if Flux is already standard there.
- Cost sensitivity — managed services (RDS instead of in-cluster Postgres, managed ingress) trade cost for reduced operational burden; worth surfacing the trade explicitly.
