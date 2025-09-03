# Kubernetes Interview Troubleshooting & Resource Management Prep

## 1. Core kubectl for triage

### Pods & Nodes
```bash
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl get nodes -o wide
kubectl describe node <node>
```

### Logs
```bash
kubectl logs <pod>
kubectl logs <pod> --previous
```

### Exec into pod
```bash
kubectl exec -it <pod> -- sh
```

### Events
```bash
kubectl get events --sort-by=.lastTimestamp | tail -n 20
```

## 2. Common Issues to Troubleshoot

### A) Pending Pod
Causes: no resources, PVC not bound, quota, taints.

```bash
kubectl describe pod <pod>   # check Events
kubectl get pvc              # PVC bound?
kubectl describe node <node> # taints, resources
kubectl get quota            # quota violations
```

Fix: adjust requests/limits, bind PVC with valid StorageClass, add tolerations, raise quota.

### B) CrashLoopBackOff
Causes: bad command, missing config, failed probes, OOMKilled.

```bash
kubectl describe pod <pod>   # look for State + Reason
kubectl logs <pod> --previous
```

Fix: correct image/command, ensure config/secrets are mounted, tune readiness/liveness, raise limits if OOMKilled.

### C) ImagePullBackOff
Causes: wrong image tag, private registry auth.

```bash
kubectl describe pod <pod>   # Events show pull errors
```

Fix: correct tag, add imagePullSecrets, check registry connectivity.

### D) PVC not bound
Causes: missing StorageClass, wrong access mode.

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get sc
```

Fix: request supported size/class/mode, or create a valid PV.

## 3. Resource Management Discussion

**Requests & Limits:**
- Requests = scheduler guarantees.
- Limits = runtime ceiling.
- Best practice: set requests near actual usage; set limits to prevent runaway.

**Autoscaling:**
- HPA = scales replicas based on CPU/memory/custom metrics.
- VPA = adjusts requests/limits based on observed usage.
- Cluster Autoscaler = adds/removes nodes.

**Storage:**
- PV = actual disk resource.
- PVC = claim for storage, matched to PV by size, mode, SC.
- StorageClass = defines how PVs are dynamically provisioned.

**Pod / Deployment / Node / Container:**
- Container = running app.
- Pod = smallest schedulable unit.
- Deployment = manages replica sets + rollout.
- Node = worker machine hosting pods.

## 4. Handy Commands

### Requests/limits table
```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,REQCPU:.spec.containers[*].resources.requests.cpu,REQMEM:.spec.containers[*].resources.requests.memory,LIMCPU:.spec.containers[*].resources.limits.cpu,LIMMEM:.spec.containers[*].resources.limits.memory
```

### Tail all containers logs
```bash
kubectl logs <pod> --all-containers -f
```

### DNS test inside pod
```bash
kubectl exec -it <pod> -- sh -c 'nslookup kubernetes.default'
```
