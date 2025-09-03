# Kubernetes Troubleshooting & Resource Mgmt – Simple Prep (Commented)

## 0) Quick cluster sanity
```bash
kubectl version --short                 # check client/server versions
kubectl config get-contexts             # confirm current context & namespace
kubectl get ns                          # list namespaces
kubectl get nodes -o wide               # list nodes + IPs/roles/status
kubectl describe node <node-name>       # detailed info: capacity, taints, conditions
```

## 1) Fast triage loop (pods → events → logs → exec)
```bash
kubectl get pods -A                     # list all pods across namespaces
kubectl get pods -n <ns> -o wide        # list pods in a specific namespace + IP/node

kubectl describe pod <pod> -n <ns>      # show pod details (events, conditions, status)

kubectl get events -A --sort-by=.lastTimestamp | tail -n 25  
# recent cluster-wide events, useful for scheduling/storage errors

kubectl logs <pod> -n <ns>              # view logs from main container
kubectl logs <pod> -n <ns> --previous   # view logs from previous crash/restart

kubectl exec -it <pod> -n <ns> -- sh    # open shell in container (if /bin/sh exists)
```

## 2) Common issues → what to check → simple fixes

### A) **Pending** pod
```bash
kubectl describe pod <pod> -n <ns>      # events show "insufficient resources" or "PVC not bound"
kubectl top nodes                       # check node CPU/mem usage (needs metrics-server)
kubectl get pvc -n <ns>                 # list PVCs and their status
kubectl get quota -n <ns>               # see ResourceQuota (limits per namespace)
kubectl describe node <node>            # check taints, allocatable resources
kubectl get storageclass                # verify available StorageClasses
```
- Fixes: lower requests, add nodes, bind PVC with correct storageClass, tolerate taints, adjust quota.

---

### B) **CrashLoopBackOff**
```bash
kubectl describe pod <pod> -n <ns>      # check for OOMKilled, probe failures
kubectl logs <pod> -n <ns> --previous   # logs from the container that just crashed
```
- Fixes: correct command/args, mount missing config, raise memory, fix readiness/liveness probes.

---

### C) **ImagePullBackOff**
```bash
kubectl describe pod <pod> -n <ns>      # shows registry/auth errors
```
- Fixes: use correct repo:tag, add imagePullSecrets for private registry.

---

### D) **PVC not bound**
```bash
kubectl get pvc -n <ns>                 # list claims
kubectl describe pvc <name> -n <ns>     # details on why it's pending
kubectl get pv                          # check available volumes
kubectl get storageclass                 # verify default/valid storageClass
```
- Fixes: correct `storageClassName`, adjust PVC size, confirm provisioner installed.

---

## 3) Resource management – key points + commands

### Requests & Limits
```bash
kubectl top pods -n <ns>                # current CPU/mem usage for pods
kubectl top nodes                       # current usage per node
```
- **Requests** = scheduler reservation (must fit on a node).  
- **Limits** = runtime cap (CPU throttled; memory beyond limit → OOMKill).  
- **QoS classes**:  
  - Guaranteed (req=limit), Burstable (req < limit), BestEffort (none set).

---

### Storage basics
```bash
kubectl get sc                          # list storage classes
kubectl describe pvc <pvc> -n <ns>      # see events and bound PV
kubectl describe pv <pv>                # see details of persistent volume
```
- PVC (what the pod wants) → matched to PV (actual disk) via StorageClass.

---

### Autoscaling
```bash
kubectl autoscale deployment <deploy> -n <ns> --min=2 --max=10 --cpu-percent=70
# create HPA targeting 70% CPU utilization

kubectl get hpa -n <ns>                 # check autoscalers
kubectl describe hpa <hpa> -n <ns>      # see scaling behavior/events
```
- HPA = scale replicas based on load.  
- VPA = adjust requests/limits for right-sizing containers.

---

### Object refresher
```bash
kubectl rollout status deployment/<name> -n <ns>   # see if rollout finished
kubectl rollout history deployment/<name> -n <ns>  # view revision history
kubectl rollout undo deployment/<name> -n <ns>     # rollback to previous version
```
- Container = image + env + ports.  
- Pod = group of containers.  
- Deployment = manages ReplicaSets + updates.  
- Node = worker machine.
