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
kubectl logs <pod> -n
kubectl logs <pod> --previous -n
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
  
**Kid version:**  
Imagine you’re at lunch:  
- **Request** = you reserve a chair so you’re guaranteed a seat.  
- **Limit** = the cafeteria lady says you can’t eat more than two slices of pizza.
  
**Autoscaling:**
- HPA = scales replicas based on CPU/memory/custom metrics.
- VPA = adjusts requests/limits based on observed usage.
- Cluster Autoscaler = adds/removes nodes.
  
**Kid version:**  
- **HPA** = adding more chairs to the table when more kids show up to eat.  
- **VPA** = giving one kid a bigger plate if they’re hungrier than the rest.  

**Storage:**
- PV = actual disk resource.
- PVC = claim for storage, matched to PV by size, mode, SC.
- StorageClass = defines how PVs are dynamically provisioned.
  
**Kid version:**  
Think of it like borrowing a book at the library:  
- **PV** = the shelf of books (actual storage).  
- **PVC** = you filling out a slip asking for a book (the request).  
- **StorageClass** = the librarian who decides which shelf to grab the book from.

**Pod / Deployment / Node / Container:**
- Container = running app.
- Pod = smallest schedulable unit.
- Deployment = manages replica sets + rollout.
- Node = worker machine hosting pods.
  
**Kid version (LEGO set analogy):**  
- **Container** = a single LEGO brick (your app).  
- **Pod** = a small LEGO model built from a few bricks (the running unit).  
- **Deployment** = the instruction booklet that says “make 5 of these LEGO models and keep them standing.”  
- **Node** = the table where you build and keep all your LEGO sets.
  
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
