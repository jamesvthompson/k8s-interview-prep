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

## 5. Practical Scenarios You Can Reproduce

### Pending (no resources)
```bash
kubectl create deployment too-big --image=nginx:1.25
kubectl set resources deploy/too-big --requests=cpu=1000,memory=500Gi
```

### Pending (PVC not bound)
```bash
kubectl create pvc bad-pvc --storage=1Gi --access-modes=ReadWriteOnce --storage-class=doesnt-exist
kubectl create deployment pvc-pending-demo --image=nginx:1.25
kubectl set volume deploy/pvc-pending-demo --add --name=data --type=persistentVolumeClaim --claim-name=bad-pvc --mount-path=/data
```

### CrashLoopBackOff (bad command)
```bash
kubectl run crash-bad-cmd --image=busybox:1.36 --restart=Never --command -- sh -c 'exit 1'
```

### CrashLoopBackOff (OOMKilled)
```bash
kubectl run crash-oom --image=python:3.11-alpine --restart=Never   --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi   -- sh -c "python -c \"a='x'*300*1024*1024; import time; time.sleep(600)\""
```

### ImagePullBackOff
```bash
kubectl run bad-image --image=nginx:doesnotexist --restart=Never
```

## 6. Namespace Simplification (so you don’t need -n every time)
```bash
kubectl create namespace tech-int-prep
kubectl config set-context --current --namespace=tech-int-prep
```

Now all commands default to that namespace:
```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
```

When finished:
```bash
kubectl delete namespace tech-int-prep
```

---

#  Kubernetes Troubleshooting Cheat Sheet

## Setup
```bash
kubectl create ns tech-int-prep
kubectl config set-context --current --namespace=tech-int-prep
```

##  Pending (no resources)

**Create**
```bash
kubectl create deploy too-big --image=nginx:1.25
kubectl set resources deploy/too-big --requests=cpu=1000,memory=500Gi
```

**Triage**
```bash
kubectl get pods
kubectl describe pod -l app=too-big   # Events → Insufficient resources
```

**Fix**
```bash
kubectl set resources deploy/too-big --requests=cpu=200m,memory=256Mi
kubectl rollout status deploy/too-big
```

##  Pending (PVC not bound)

**Create**
```bash
kubectl create pvc bad-pvc --storage=1Gi --access-modes=ReadWriteOnce --storage-class=doesnt-exist
kubectl create deploy pvc-demo --image=nginx:1.25
kubectl set volume deploy/pvc-demo --add --name=data --type=persistentVolumeClaim --claim-name=bad-pvc --mount-path=/data
```

**Triage**
```bash
kubectl get pvc
kubectl describe pvc bad-pvc   # Events → no SC found
```

**Fix**
```bash
kubectl delete pvc bad-pvc
kubectl create pvc good-pvc --storage=1Gi --access-modes=ReadWriteOnce --storage-class=standard
kubectl set volume deploy/pvc-demo --remove --name=data
kubectl set volume deploy/pvc-demo --add --name=data --type=persistentVolumeClaim --claim-name=good-pvc --mount-path=/data
```

##  CrashLoopBackOff (bad command)

**Create**
```bash
kubectl run crash-bad --image=busybox:1.36 --restart=Never -- sh -c 'exit 1'
```

**Triage**
```bash
kubectl describe pod crash-bad   # State: CrashLoopBackOff
kubectl logs crash-bad --previous
```

**Fix**
```bash
kubectl delete pod crash-bad
kubectl run crash-bad --image=busybox:1.36 --restart=Never -- sh -c 'sleep 3600'
```

##  CrashLoopBackOff (OOMKilled)

**Create**
```bash
kubectl run crash-oom --image=python:3.11-alpine --restart=Never   --requests=cpu=100m,memory=64Mi --limits=cpu=200m,memory=128Mi   -- sh -c "python -c \"a='x'*300*1024*1024; import time; time.sleep(600)\""
```

**Triage**
```bash
kubectl describe pod crash-oom   # Reason: OOMKilled
```

**Fix**
```bash
kubectl delete pod crash-oom
kubectl run crash-oom --image=python:3.11-alpine --restart=Never   --requests=cpu=100m,memory=256Mi --limits=cpu=200m,memory=512Mi   -- sh -c "python -c \"import time; time.sleep(600)\""
```

##  ImagePullBackOff

**Create**
```bash
kubectl run bad-image --image=nginx:doesnotexist --restart=Never
```

**Triage**
```bash
kubectl describe pod bad-image   # Events → image not found
```

**Fix**
```bash
kubectl delete pod bad-image
kubectl run bad-image --image=nginx:1.25 --restart=Never
```

## Cleanup
```bash
kubectl delete ns tech-int-prep
```
