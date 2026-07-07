# Debugging Playbooks by Symptom Category

Use this as a diagnostic ordering reference, not a script to dump on the user. Pick the category matching the symptom, then walk the commands one at a time per the main SKILL.md protocol — state brief reasoning, one command, wait for output, adapt.

## Table of Contents
1. Pod stuck Pending (scheduling)
2. CrashLoopBackOff / container exits
3. ImagePullBackOff / ErrImagePull
4. Pod Running but not Ready
5. Networking: Service unreachable / DNS failures
6. Ingress / LoadBalancer routing issues
7. Storage: PVC Pending / mount failures
8. OOMKilled / resource pressure
9. RBAC / permission denied
10. Node-level issues
11. Rollout stuck / stale deployment
12. Control plane / API server issues

---

## 1. Pod stuck Pending (scheduling)

Diagnostic priority (cheapest, most-explains-first):
1. `kubectl describe pod <pod> -n <ns>` — the Events section usually names the exact reason (Insufficient cpu/memory, node(s) had taint, didn't match node selector/affinity, unbound PVC).
2. If resources: `kubectl top nodes` / `kubectl describe nodes` — check allocatable vs. requested.
3. If taints/affinity: `kubectl get nodes -o json | jq '.items[].spec.taints'` and compare against pod's tolerations/affinity.
4. If PVC: jump to storage playbook (#7).

Common root causes: insufficient cluster capacity, restrictive node affinity/anti-affinity, taints with no matching toleration, unbound PersistentVolumeClaim, pod requesting more than any single node has (won't ever schedule regardless of cluster size).

## 2. CrashLoopBackOff / container exits

Diagnostic priority:
1. `kubectl logs <pod> -n <ns> --previous` — the previous container's logs, since the current one may have just restarted with nothing logged yet.
2. `kubectl describe pod <pod> -n <ns>` — check exit code and reason in the container status (0 = clean exit but still crashlooping is often a misconfigured command; 137 = OOMKilled or SIGKILL; 1 = app error).
3. If exit code 137: go to OOM playbook (#8).
4. If no useful logs at all: check the container's entrypoint/command for a typo or missing config/env var causing immediate exit.

Common root causes: application error on startup (bad config, missing env var/secret, unreachable dependency), OOMKill, liveness probe killing a slow-starting app before it's ready, wrong command/entrypoint override in the manifest.

## 3. ImagePullBackOff / ErrImagePull

Diagnostic priority:
1. `kubectl describe pod <pod> -n <ns>` — Events will show the exact pull error (not found, unauthorized, manifest unknown, timeout).
2. If unauthorized: check `imagePullSecrets` on the pod/service account, and whether the secret exists in the right namespace and is valid (registry credentials can expire, especially ECR tokens which rotate every 12h).
3. If not found: check for a typo in image name/tag, or whether the tag was actually pushed.
4. If timeout: node-level network/egress issue reaching the registry — check node's ability to reach the registry (proxy, firewall, private registry DNS).

Common root causes: typo in image/tag, private registry auth expired or missing, image genuinely doesn't exist (CI didn't push it yet), node egress blocked to registry.

## 4. Pod Running but not Ready

Diagnostic priority:
1. `kubectl describe pod <pod> -n <ns>` — check readiness probe results/events.
2. `kubectl logs <pod> -n <ns>` — is the app actually up and listening on the expected port?
3. Manually verify the probe's target (path/port) matches what the app actually serves — a common bug is probe hitting the wrong port or a path that 404s.

Common root causes: readiness probe misconfigured (wrong port/path), app takes longer to become ready than `initialDelaySeconds` allows, app is up but a dependency it needs (DB, downstream service) isn't, missing startupProbe on slow-starting apps causing readiness/liveness to fight the startup window.

## 5. Networking: Service unreachable / DNS failures

Diagnostic priority:
1. `kubectl get endpoints <service> -n <ns>` — if empty, the Service has no matching Ready pods (selector mismatch or all pods unready — loop back to #4).
2. `kubectl exec` into a debug pod and `curl`/`nc` the ClusterIP directly — isolates Service-layer routing from DNS.
3. If ClusterIP works but DNS name doesn't resolve: `kubectl exec` into a pod, `nslookup <service>.<namespace>.svc.cluster.local` — isolates CoreDNS.
4. If CoreDNS itself is the problem: `kubectl get pods -n kube-system -l k8s-app=kube-dns` and check its logs.

Common root causes: Service selector doesn't match pod labels (most common — always check `endpoints` first), pods genuinely unready, NetworkPolicy blocking traffic, CoreDNS pods unhealthy or resource-starved, wrong namespace in the DNS query.

## 6. Ingress / LoadBalancer routing issues

Diagnostic priority:
1. `kubectl describe ingress <name> -n <ns>` — check backend resolution and any controller-specific events/annotations errors.
2. Confirm the Service behind the Ingress has Ready endpoints (loop to #5 if not).
3. Check the ingress controller's own logs (`kubectl logs -n <ingress-ns> <ingress-controller-pod>`) for the specific request path.
4. For cloud LoadBalancer Services: check cloud console/CLI for the actual LB's health check status — a Service `type: LoadBalancer` stuck `<pending>` almost always means a cloud-provider-side issue (quota, subnet tagging, IAM), not a Kubernetes-side one.

Common root causes: Ingress annotation typo (controller-specific, e.g. missing `kubernetes.io/ingress.class` or the newer `ingressClassName`), backend Service has no Ready endpoints, TLS secret missing/mismatched, cloud LB health check hitting the wrong port/path, security group/firewall blocking the LB's health check traffic.

## 7. Storage: PVC Pending / mount failures

Diagnostic priority:
1. `kubectl describe pvc <pvc> -n <ns>` — Events show binding failure reason.
2. `kubectl get storageclass` — confirm the requested StorageClass exists and has a provisioner (or `WaitForFirstConsumer` binding mode, which is normal and means it binds once a pod is scheduled — not itself a bug).
3. If mount failure on a scheduled pod: `kubectl describe pod` — check for multi-attach errors (RWO volume already attached elsewhere) or provisioner-side errors.

Common root causes: no default StorageClass and none specified, StorageClass provisioner unhealthy or misconfigured, RWO volume trying to attach to a second node (common after a node replacement without the old pod being cleaned up first), zone mismatch (PV in zone A, pod scheduled in zone B on cloud providers with zonal disks).

## 8. OOMKilled / resource pressure

Diagnostic priority:
1. `kubectl describe pod <pod> -n <ns>` — confirm `OOMKilled` in last state and check the memory limit set.
2. `kubectl top pod <pod> -n <ns>` (needs metrics-server) — compare actual usage trend against the limit.
3. Check if this is a genuine leak (steadily climbing) vs. a limit set too low for a legitimate spike (e.g. JVM heap sizing, batch job).

Common root causes: memory limit set too low for real usage, actual application memory leak, no limit set at all combined with node-level memory pressure evicting the pod, JVM/runtime not respecting container memory limits (older Java versions especially).

## 9. RBAC / permission denied

Diagnostic priority:
1. Read the exact error — it names the verb, resource, and namespace denied (e.g. `pods is forbidden: User "x" cannot list resource "pods" in API group ""`).
2. `kubectl auth can-i <verb> <resource> --as=<user-or-sa> -n <ns>` — reproduce the check directly rather than guessing.
3. `kubectl get rolebinding,clusterrolebinding -A -o wide | grep <subject>` — find what's actually bound to the subject in question.

Common root causes: missing RoleBinding/ClusterRoleBinding entirely, RoleBinding in the wrong namespace, ServiceAccount not what you assumed (pod using `default` SA, not the one with the role), ClusterRole itself missing the verb/resource.

## 10. Node-level issues

Diagnostic priority:
1. `kubectl get nodes` — check `Ready` status and `SchedulingDisabled`.
2. `kubectl describe node <node>` — check conditions (MemoryPressure, DiskPressure, PIDPressure) and recent events.
3. If node is NotReady: this is usually kubelet or container-runtime health on the node itself, not app-layer — may need node-level access (SSH, cloud console, node problem detector) beyond kubectl.

Common root causes: kubelet crashed or can't reach the API server, disk pressure (often from unpruned images/logs), underlying VM/instance issue on cloud providers, container runtime (containerd/CRI-O) unhealthy.

## 11. Rollout stuck / stale deployment

Diagnostic priority:
1. `kubectl rollout status deployment/<name> -n <ns>` — states what it's waiting on.
2. `kubectl get rs -n <ns> -l <selector>` — check if the new ReplicaSet is scaling up and old one scaling down as expected.
3. `kubectl describe rs <new-rs> -n <ns>` — the new ReplicaSet's pods will show the actual failure (often loops back to CrashLoopBackOff/Pending/ImagePull above).

Common root causes: new pods failing to become Ready (readiness probe or app-level failure — loop back to #2/#4), `maxUnavailable`/`maxSurge` configuration preventing progress given available capacity, PodDisruptionBudget blocking eviction of old pods.

## 12. Control plane / API server issues

Diagnostic priority:
1. `kubectl get componentstatuses` (deprecated in newer versions but still informative on some distros) or check control-plane pod health directly on self-managed clusters: `kubectl get pods -n kube-system`.
2. On managed clusters (EKS/GKE/AKS), the control plane isn't directly inspectable — check the cloud provider's cluster status/health dashboard and audit logs instead of chasing it via kubectl.
3. `kubectl get events -A --sort-by='.lastTimestamp'` — cluster-wide event stream often surfaces control-plane-adjacent issues (etcd, scheduler, controller-manager) even when you can't see those components directly.

Common root causes (self-managed): etcd disk pressure or quorum loss, API server certificate expiry, scheduler/controller-manager crash. On managed clusters, this category is rare — the cloud provider owns it — so if kubectl itself is timing out/erroring, check cluster reachability/auth (kubeconfig, IAM token expiry) before assuming a real control-plane outage.
