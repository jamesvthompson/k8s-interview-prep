# Kubernetes Interview Troubleshooting Scenarios (Namespace-only)

## Setup (namespace + optional default)
```bash
kubectl create namespace tech-int-prep
# optional: set your current context to use the namespace by default
kubectl config set-context --current --namespace=tech-int-prep
```

---

## 1) PENDING

### A) Pending due to no resources

**Create the problem**
```bash
kubectl create deployment too-big --image=nginx:1.25 -n tech-int-prep
kubectl set resources deploy/too-big --requests=cpu=1000,memory=500Gi -n tech-int-prep
```

**Triage**
```bash
kubectl get pods -n tech-int-prep -o wide
kubectl describe pod -l app=too-big -n tech-int-prep     # read Events for "Insufficient ..."
kubectl get events -n tech-int-prep --sort-by=.lastTimestamp | tail -n 20
```

**Fix (re-apply sane requests)**
```bash
kubectl set resources deploy/too-big --requests=cpu=200m,memory=256Mi -n tech-int-prep
kubectl rollout status deploy/too-big -n tech-int-prep
```

---

### B) Pending due to PVC not bound

**Create a bad PVC + a pod that needs it**
```bash
kubectl create pvc bad-pvc --storage=1Gi --access-modes=ReadWriteOnce --storage-class=doesnt-exist -n tech-int-prep

kubectl create deployment pvc-pending-demo --image=nginx:1.25 -n tech-int-prep
kubectl set volume deploy/pvc-pending-demo --add --name=data --type=persistentVolumeClaim --claim-name=bad-pvc --mount-path=/data -n tech-int-prep
```

**Triage**
```bash
kubectl get pvc -n tech-int-prep
kubectl describe pvc bad-pvc -n tech-int-prep           # Events show storage class issue
kubectl describe deploy pvc-pending-demo -n tech-int-prep
kubectl get pods -n tech-int-prep
kubectl get sc                                           # see valid StorageClasses
```

**Fix (create a good PVC and point the workload at it)**
```bash
# replace "standard" with a real class from your cluster
kubectl create pvc good-pvc --storage=1Gi --access-modes=ReadWriteOnce --storage-class=standard -n tech-int-prep

# switch the deployment to the good claim
kubectl set volume deploy/pvc-pending-demo --remove --name=data -n tech-int-prep
kubectl set volume deploy/pvc-pending-demo --add --name=data --type=persistentVolumeClaim --claim-name=good-pvc --mount-path=/data -n tech-int-prep

kubectl get pods -n tech-int-prep
```

---

## 2) CRASHLOOPBACKOFF

### A) Bad command/args

**Create the problem**
```bash
kubectl run crash-bad-cmd --image=busybox:1.36 --restart=Never -n tech-int-prep --command -- sh -c 'echo starting && exit 1'
```

**Triage**
```bash
kubectl get pod crash-bad-cmd -n tech-int-prep
kubectl describe pod crash-bad-cmd -n tech-int-prep
kubectl logs crash-bad-cmd -n tech-int-prep --previous
```

**Fix (recreate with a healthy command)**
```bash
kubectl delete pod crash-bad-cmd -n tech-int-prep
kubectl run crash-bad-cmd --image=busybox:1.36 --restart=Never -n tech-int-prep --command -- sh -c 'echo healthy; sleep 3600'
```

---

### B) OOMKilled

**Create the problem**
```bash
kubectl run crash-oom --image=python:3.11-alpine --restart=Never -n tech-int-prep   --requests=cpu=100m,memory=64Mi   --limits=cpu=200m,memory=128Mi   -- sh -c "python -c "a='x'*300*1024*1024; import time; time.sleep(600)""
```

**Triage**
```bash
kubectl get pod crash-oom -n tech-int-prep
kubectl describe pod crash-oom -n tech-int-prep          # Reason: OOMKilled
kubectl get events -n tech-int-prep --sort-by=.lastTimestamp | tail -n 20
```

**Fix (recreate with higher limits)**
```bash
kubectl delete pod crash-oom -n tech-int-prep
kubectl run crash-oom --image=python:3.11-alpine --restart=Never -n tech-int-prep   --requests=cpu=100m,memory=256Mi   --limits=cpu=200m,memory=512Mi   -- sh -c "python -c "import time; print('ok'); time.sleep(600)""
```

---

## 3) IMAGEPULLBACKOFF

### A) Bad tag

**Create the problem**
```bash
kubectl run bad-image --image=nginx:doesnotexist --restart=Never -n tech-int-prep
```

**Triage**
```bash
kubectl describe pod bad-image -n tech-int-prep          # pull error in Events
kubectl get events -n tech-int-prep --field-selector involvedObject.name=bad-image --sort-by=.lastTimestamp
```

**Fix (set a valid image)**
```bash
kubectl set image pod/bad-image app=nginx:1.25 -n tech-int-prep
kubectl get pod bad-image -n tech-int-prep -w

# Or recreate:
kubectl delete pod bad-image -n tech-int-prep
kubectl run bad-image --image=nginx:1.25 --restart=Never -n tech-int-prep
```

---

## Cleanup
```bash
kubectl delete namespace tech-int-prep
```
