# Ecosystem & Managed-Variant Notes

Reach for this when the user's stack involves these tools/platforms — for specifics worth getting right rather than guessing at.

## Helm

- Chart review basics: check `values.yaml` defaults are actually safe to deploy as-is (resource limits, replica counts) — a chart that only works after heavy overriding is a footgun for anyone else who installs it.
- `helm template` before `helm install`/`upgrade` to see rendered manifests when debugging — much faster than trial-and-error installs.
- Common gotcha: `helm upgrade` doesn't recreate resources that don't support in-place update the way you'd expect for some fields (e.g. changing a StatefulSet's volumeClaimTemplates requires manual intervention, Helm won't do it for you).
- Subchart value overrides use the subchart's name as the key in the parent's `values.yaml` — a frequent source of "why didn't my override apply" confusion.
- Helm hooks (pre-install, post-upgrade, etc.) run as separate Jobs — failures there can leave a release in a weird state (`helm history` and `helm rollback` are the way out, not manual resource surgery).

## GitOps (ArgoCD / Flux)

- Core principle to check for: is the cluster state actually only reachable via git, or is there config drift from manual `kubectl apply`/`edit` that GitOps will silently revert (surprising people) or that will fight with the GitOps controller?
- ArgoCD: check sync policy — `automated` with `selfHeal: true` means manual changes get reverted, which is often desired but should be a known, not surprised-into, behavior.
- Sync waves / hooks matter for ordering (e.g. CRDs before the resources that use them) — a common failure is a fresh-install ordering problem that an existing cluster never surfaces.
- Flux vs ArgoCD: don't recommend switching an established one without a real reason — the migration cost usually isn't worth it just for preference.

## Service Mesh (Istio / Linkerd)

- The question to ask before recommending a mesh at all: what specific problem is it solving that isn't better solved more cheaply (mTLS via simpler cert tooling, retries/timeouts at the app or Ingress layer, observability via metrics + tracing without full mesh injection)? Meshes add real operational cost — sidecar resource overhead, upgrade complexity, another thing that can break traffic. This is often the most valuable "trade-off" callout in an architecture conversation.
- Istio: sidecar injection issues are one of the most common mesh-adjacent debugging categories — check `kubectl get pod -o jsonpath='{.spec.containers[*].name}'` for the `istio-proxy` container's presence when traffic behaves unexpectedly post-mesh-adoption.
- Linkerd is lighter-weight operationally than Istio — worth mentioning as an option when the team wants mTLS/observability without Istio's full feature surface and complexity.
- mTLS mode (permissive vs strict) is a common source of "service can't reach service" issues right after enabling a mesh — check this before assuming it's a NetworkPolicy or DNS problem.

## Vault (Secrets Management)

- Two common integration patterns: **Vault Agent Injector** (sidecar injects secrets into a shared volume via annotations on the pod) vs. **Vault CSI Provider** (secrets mounted as a volume via CSI driver). Know which one's in play before debugging a "secret not showing up" issue — the failure surface is different (sidecar init-container logs vs. CSI driver pod logs).
- Common failure: Vault Agent sidecar can't authenticate — usually a Kubernetes auth method misconfiguration (wrong role, ServiceAccount not bound to the expected Vault role, or the Vault-side Kubernetes auth backend pointed at the wrong cluster's API/JWKS endpoint).
- Secret rotation: Agent Injector can template and refresh secrets on a lease without pod restart; CSI provider's rotation behavior depends on `rotationPollInterval` — an app that only reads a secret at startup won't pick up rotation either way unless it re-reads the file or gets restarted (a common source of "I rotated it but nothing changed" confusion).
- Vault itself unsealed/unavailable is a distinct failure category from the injector/CSI layer — check Vault's own health endpoint before assuming the Kubernetes-side integration is at fault.

## Terraform (Provisioning)

- Relevant mainly for cluster-level and surrounding infra (node groups, VPC/networking, IAM roles for IRSA/Workload Identity, managed add-ons) rather than in-cluster application resources — flag it if you see Terraform being used to manage things Helm/kubectl would handle better (individual Deployments, ConfigMaps that change often), since that mixing tends to cause drift and slow iteration.
- Common drift issue: manual `kubectl` changes or a GitOps controller (ArgoCD/Flux) both mutating resources that Terraform also thinks it owns (e.g. an add-on's Helm release defined in Terraform via the `helm` provider, but also synced by ArgoCD) — pick one owner per resource.
- `terraform plan` showing unexpected diffs on managed node groups/IAM after a cloud console or `eksctl`-style manual change is drift, not a bug — worth checking who else might be touching the same resource.
- State lock or stale state on shared cluster infra is the most common "why won't my apply go through" — check for a stuck lock before assuming a real infra conflict.

## OpenSearch (Logging/Observability)

- Common in-cluster pattern: Fluent Bit/Fluentd DaemonSet shipping container logs to an OpenSearch cluster (self-managed in-cluster or managed service). If logs aren't showing up, check in this order: is the DaemonSet pod running on the node in question, is it actually tailing the right log path/container, and can it reach the OpenSearch endpoint (network policy, auth, TLS).
- Index management: without an ISM (Index State Management) policy, indices grow unbounded — a frequent root cause of OpenSearch cluster disk-pressure/yellow-red status incidents that looks like an app logging problem but isn't.
- OpenSearch cluster yellow/red health is its own diagnostic path (shard allocation, disk watermarks, node count vs. replica count) — distinct from whether logs are being shipped at all; don't conflate "I don't see my logs" (shipping/parsing issue) with "the cluster is unhealthy" (OpenSearch-side capacity/allocation issue).
- Resource requests/limits on the OpenSearch data nodes matter more than most workloads — JVM heap sizing (typically half of container memory, capped) is a common misconfiguration causing OOM or poor performance under load.

## Managed Kubernetes Variants

### EKS (AWS)
- IAM integration: IRSA (IAM Roles for Service Accounts) or EKS Pod Identity is how pods should get AWS permissions — never long-lived AWS credentials baked into Secrets.
- Networking: VPC CNI assigns pods real VPC IPs by default — IP exhaustion in the subnet is a real, distinct failure mode from anything in vanilla k8s (`kubectl describe pod` will show scheduling failures with a cause distinct to "no IPs available" that's easy to misread as a generic scheduling issue).
- Node groups: managed node groups vs. self-managed vs. Fargate profiles all have different failure/debug surfaces — Fargate especially removes node-level debugging entirely (no `kubectl exec` into a node, no DaemonSets).
- Ingress: typically AWS Load Balancer Controller (ALB/NLB) — its own set of annotations and IAM permissions to get right, and its own controller pod worth checking logs on when Ingress behaves oddly.

### GKE (Google Cloud)
- Autopilot vs Standard: Autopilot removes node-level control (no DaemonSets in the traditional sense, no direct node access, resource requests become semi-mandatory) — worth confirming which mode before giving node-level debugging advice, since half of it doesn't apply.
- Workload Identity is GKE's equivalent of IRSA — pods get GCP service account permissions without key files.
- GKE's default network policy and CNI (Dataplane V2 / Cilium-based on newer clusters) changes some NetworkPolicy behavior versus vanilla Calico — worth knowing if migrating policies from elsewhere.

### AKS (Azure)
- Azure AD Workload Identity is the IRSA/Workload-Identity equivalent for pod-level Azure permissions.
- AKS's default CNI (kubenet vs Azure CNI) materially changes pod networking model and IP consumption from the VNet — relevant to the same kind of IP-exhaustion issue as EKS's VPC CNI, but configured differently.
- Ingress commonly via AGIC (Application Gateway Ingress Controller) or a standard ingress-nginx — check which is in play before assuming annotation syntax.

### General managed-cluster debugging note
Across all three: control-plane-level issues (etcd, scheduler, API server internals) are the cloud provider's problem, not yours — if kubectl itself is failing/timing out, check auth/connectivity (kubeconfig, IAM token expiry, cluster endpoint reachability/private-endpoint VPN) before assuming a genuine control-plane outage. Trying to `kubectl describe` your way into a managed control plane's internals is usually a dead end — check the cloud console's cluster health/status page instead.
