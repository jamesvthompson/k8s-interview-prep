# Kubernetes Interview Cheatsheet

> **Goal:** Speak confidently about how Kubernetes works, what each object does, and how to triage common issues with only basic `kubectl`.

---

## Mental Model: How Kubernetes Works
1. **You talk to the API server** → every config is a declarative object stored in **etcd** (cluster state).
2. **Admission & validation** → built-ins + webhooks can mutate/validate before storage.
3. **Controllers reconcile** → make actual state match desired (e.g., Deployment → ReplicaSets → Pods).
4. **Scheduler places Pods** → picks Nodes based on requests, constraints, policies.
5. **Kubelet runs containers** → pulls images, runs probes, reports status.
6. **Networking & exposure** → Services give stable virtual IPs; Ingress/Gateway routes external traffic.
7. **Storage abstraction** → PVCs bind to PVs via StorageClasses (often dynamically provisioned).

---

## Control Plane vs. Nodes (What’s What)
- **kube-apiserver**: front door; authn, authz, admission.
- **etcd**: source of truth (state DB).
- **kube-scheduler**: binds Pods to Nodes.
- **controller-manager(s)**: run control loops (Deployment, Job, Node, etc.).
- **cloud-controller-manager**: cloud integrations (LBs, volumes).
- **On each Node:** **kubelet** (agent), **kube-proxy** (Service routing), **container runtime** (containerd, etc.).

---

## Objects Cheat Sheet (Use the Right Thing)
### Workloads
- **Pod**: Smallest unit; 1+ containers share net/volumes.
- **ReplicaSet**: Ensures N identical Pods (usually owned by Deployments).
- **Deployment**: Rolling updates/rollbacks for stateless apps (manages ReplicaSets).
- **StatefulSet**: Stable IDs, ordered rollout, stable volumes (DBs/brokers).
- **DaemonSet**: One Pod per Node (agents/log shippers).
- **Job**: Run-to-completion batch.
- **CronJob**: Scheduled Jobs.

### Config & Identity
- **ConfigMap**: Non-secret config (env/files).
- **Secret**: Sensitive config (base64—treat as secret).
- **ServiceAccount**: Pod identity to API; RBAC ties permissions.

### Networking
- **Service**: Stable virtual IP & discovery.
  - `ClusterIP` (internal), `NodePort` (per-node port), `LoadBalancer` (cloud LB), **Headless** (no virtual IP, direct endpoints/DNS).
- **Endpoints/EndpointSlice**: Backends behind a Service (auto-managed).
- **Ingress / Gateway**: L7 HTTP(S) routing via controller.
- **NetworkPolicy**: Pod-level allow/deny rules.

### Storage
- **PersistentVolume (PV)**: Actual storage.
- **PersistentVolumeClaim (PVC)**: App’s request for storage.
- **StorageClass**: How to provision PVs (parameters/provisioner).

### Scheduling & Resilience
- **Requests/Limits**: Scheduler placement & caps; QoS classes.
- **Probes**: `readiness` (traffic gate), `liveness` (restart), `startup` (boot time).
- **HPA/VPA**: Auto-scale replicas vs. resources.
- **PodDisruptionBudget (PDB)**: Min available during maintenance.
- **(Anti)Affinity & TopologySpread**: Place/spread Pods.
- **PriorityClass**: Preemption & scheduling priority.
- **Taints/Tolerations**: Restrict Nodes to certain Pods (or vice versa).

### Security & Policy
- **RBAC**: Roles & Bindings define permissions.
- **Pod Security (PSA labels)**: Enforce baseline/restricted policies.
- **Admission Controllers/Webhooks**: Mutate/validate on create/update.

---

## Request Lifecycle (Why YAML “Sticks”)
`kubectl → API server (authn/ authz) → admission (mutate/validate) → etcd → controllers reconcile → scheduler binds Pod → kubelet runs → services/network expose`

---

## Which Object for Which Problem
- **Stateless web app with rollbacks** → Deployment + Service + Ingress.
- **Per-node agent** → DaemonSet.
- **DB with stable identity/storage** → StatefulSet + PVC + StorageClass.
- **Nightly script that must finish** → CronJob → Job.
- **Lock down Pod-to-Pod traffic** → NetworkPolicy.
- **Keep 2 replicas during maintenance** → PDB.
- **Land Pods on SSD nodes only** → node labels + nodeAffinity (or taints/tolerations).
- **Auto-scale on CPU** → HPA (set CPU requests!).
- **Mount config without rebuilding image** → ConfigMap/Secret as files/env.

---

## Resource Management Essentials
- **Requests** (what scheduler reserves) vs **Limits** (hard cap).  
  - QoS: **Guaranteed** (req=limit CPU+mem), **Burstable**, **BestEffort**.
- **Typical pitfalls**
  - **OOMKilled** → memory **limit** too low or leak.
  - **Pending** → requests too large; taints; unbound PVC; quota.
- **HPA math** needs CPU requests; Metrics Server must be working.
- **Storage**: `PVC (claim)` ↔ `PV (backing)` via `StorageClass` (dynamic).

---

## Triage with Only Basic `kubectl`
```bash
# Pods & Nodes
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl get nodes -o wide
kubectl describe node <node>

# Logs & Exec
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh

# Events & Services
kubectl get events --sort-by=.lastTimestamp | tail -n 20
kubectl get svc; kubectl describe svc <svc>
kubectl get endpointslices

# Storage
kubectl get pvc; kubectl describe pvc <pvc>
kubectl get pv;  kubectl describe pv <pv>

# Scale & Rollout
kubectl scale deploy <name> --replicas=3
kubectl rollout status deploy <name>
kubectl rollout undo deploy <name>
```

---

## Classic Troubleshooting → Likely Root Causes & Fixes
### A) **Pod Pending**
- **Common causes:** requests > capacity; PVC Pending; quota; taints.
- **Fix:** lower requests or add nodes; correct StorageClass/size; bind PVC; add toleration/labels; review quotas.

### B) **CrashLoopBackOff**
- **Common causes:** bad command/args; missing config; failing probes; OOMKilled.
- **Fix:** `kubectl logs ... --previous`; check probes; env & mounts; increase mem limit; fix entrypoint/args.

### C) **ImagePullBackOff**
- **Common causes:** wrong image/tag; missing registry auth; DNS/network issues.
- **Fix:** correct image; add `imagePullSecrets`; fix registry/DNS; confirm network egress.

---

## Admission Controllers (One-Liner)
> Admission is the **gate** where Kubernetes can mutate (inject sidecars/defaults) or validate (enforce policies) **before** persisting objects. Built-ins (e.g., `LimitRanger`, `ResourceQuota`) + webhooks (e.g., Gatekeeper, Kyverno, sidecar injectors).
