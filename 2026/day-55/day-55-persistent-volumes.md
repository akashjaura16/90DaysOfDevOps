# Day 55 – Persistent Volumes (PV) and Persistent Volume Claims (PVC)

## Why Containers Need Persistent Storage

Containers are ephemeral by design. When a Pod is deleted, restarted, or rescheduled, everything written inside the container's filesystem is lost. This is fine for stateless apps but a serious problem for:

- Databases (PostgreSQL, MySQL, MongoDB)
- File uploads and user-generated content
- Application logs that must survive restarts
- Shared data between containers in a Pod

Kubernetes solves this with **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)** — a system that decouples storage from the Pod lifecycle.

---

## What Are PVs and PVCs?

### PersistentVolume (PV)
A PV is a piece of storage provisioned in the cluster. It exists independently of any Pod and survives Pod deletion. Think of it as a physical disk registered with Kubernetes.

- PVs are **cluster-wide** (not namespaced)
- Status lifecycle: `Available` → `Bound` → `Released`

### PersistentVolumeClaim (PVC)
A PVC is a request for storage made by a user or application. Kubernetes matches the PVC to a suitable PV based on capacity and access mode.

- PVCs are **namespaced**
- Once bound, a PVC gives a Pod a stable reference to storage

### How They Relate

```
Pod  ──uses──►  PVC  ──binds to──►  PV  ──backed by──►  actual storage
                     (Kubernetes                         (hostPath, NFS,
                      matches them)                       cloud disk, etc.)
```

---

## Access Modes

| Mode | Short | Meaning |
|------|-------|---------|
| ReadWriteOnce | RWO | Read-write by a single node |
| ReadOnlyMany  | ROX | Read-only by many nodes |
| ReadWriteMany | RWX | Read-write by many nodes |

---

## Reclaim Policies

| Policy | Behaviour |
|--------|-----------|
| `Retain` | PV survives PVC deletion. Data kept. Admin must clean up manually. |
| `Delete` | PV and underlying storage deleted automatically when PVC is deleted. |
| `Recycle` | Deprecated. Basic scrub then made available again. |

---

## Static vs Dynamic Provisioning

### Static Provisioning
An admin manually creates PVs ahead of time. Developers then create PVCs that bind to available PVs.

**Workflow:**
1. Admin creates PV
2. Developer creates PVC
3. Kubernetes matches them by capacity + access mode
4. Status becomes `Bound`

### Dynamic Provisioning
Developers only create PVCs. The **StorageClass** automatically provisions a matching PV on demand — no admin involvement needed.

**Workflow:**
1. Developer creates PVC with a `storageClassName`
2. StorageClass provisioner creates PV automatically
3. Status becomes `Bound` immediately

---

## Task 1: Ephemeral Storage with emptyDir

Demonstrates data loss on Pod deletion.

```yaml
# emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: writer
    image: busybox:latest
    command: ["sh", "-c", "echo $(date) > /data/message.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: temp-storage
  volumes:
  - name: temp-storage
    emptyDir: {}
```

```bash
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-pod -- cat /data/message.txt

kubectl delete pod emptydir-pod
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-pod -- cat /data/message.txt
# Timestamp is different — data was lost on Pod deletion
```

---

## Task 2: Create a PersistentVolume (Static Provisioning)

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volumes
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/k8s-pv-data
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
# STATUS: Available
```

> `hostPath` is fine for learning on a local cluster. Not suitable for production.

---

## Task 3: Create a PersistentVolumeClaim

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl get pv
# Both show STATUS: Bound
# VOLUME column in kubectl get pvc shows: volumes
```

---

## Task 4: Use the PVC in a Pod

```yaml
# pvcbinding.yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: kube-01
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: pvc
  containers:
  - image: busybox:latest
    name: test-container
    command: ["sh", "-c", "echo $(date) > /data/message.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: task-pv-storage   # must match volumes[].name exactly
```

```bash
kubectl apply -f pvcbinding.yaml
kubectl exec task-pv-pod -- cat /data/message.txt

# Delete and recreate
kubectl delete pod task-pv-pod
kubectl apply -f pvcbinding.yaml
kubectl exec task-pv-pod -- cat /data/message.txt
# File still exists — data survived Pod deletion
```

> **Common mistake:** `volumeMounts[].name` must match `volumes[].name`, NOT the Pod name or PVC name.

---

## Task 5: StorageClasses and Dynamic Provisioning

```bash
kubectl get storageclass
kubectl describe storageclass standard
```

Key fields to note:
- **Provisioner** — what creates the PV (e.g. `rancher.io/local-path`, `kubernetes.io/aws-ebs`)
- **Reclaim Policy** — usually `Delete` for dynamic classes
- **Volume Binding Mode** — `Immediate` or `WaitForFirstConsumer`

The StorageClass marked `(default)` is used when no `storageClassName` is specified in a PVC.

---

## Task 6: Dynamic Provisioning

```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: standard
```

```yaml
# dynamic-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  volumes:
    - name: dynamic-storage
      persistentVolumeClaim:
        claimName: dynamic-pvc
  containers:
  - name: test-container
    image: busybox:latest
    command: ["sh", "-c", "echo dynamic-$(date) > /data/message.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: dynamic-storage
```

```bash
kubectl apply -f dynamic-pvc.yaml
kubectl apply -f dynamic-pod.yaml
kubectl get pv
# Two PVs now visible:
#   volumes          — manual, Retain, 1Gi  (static)
#   pvc-11b534f1-... — standard, Delete, 500Mi (dynamic, auto-generated name)

kubectl exec dynamic-pod -- cat /data/message.txt
```

---

## Task 7: Clean Up

```bash
# 1. Delete pods first
kubectl delete pod task-pv-pod dynamic-pod

# 2. Delete PVCs
kubectl delete pvc pvc dynamic-pvc

# 3. Check PVs
kubectl get pv
# dynamic PV (pvc-11b534f1-...) is GONE — Delete reclaim policy
# manual PV (volumes) shows Released — Retain reclaim policy

# 4. Delete the manual PV
kubectl delete pv volumes

# 5. If stuck in Terminating, remove finalizers
# PowerShell syntax (escaped quotes required):
kubectl patch pv volumes -p '{\"metadata\":{\"finalizers\":null}}'
kubectl patch pvc pvc -p '{\"metadata\":{\"finalizers\":null}}'

# Bash/Linux syntax:
# kubectl patch pv volumes -p '{"metadata":{"finalizers":null}}'
# kubectl patch pvc pvc -p '{"metadata":{"finalizers":null}}'

# 6. Confirm clean
kubectl get pv    # No resources found
kubectl get pvc   # No resources found
```

**Why the difference?**

| PV | Reclaim Policy | Result after PVC deleted |
|----|---------------|--------------------------|
| `pvc-11b534f1-...` | `Delete` | Auto-deleted with PVC |
| `volumes` | `Retain` | Stays as `Released` — manual cleanup required |

> `Released` ≠ available. A Released PV still holds a reference to the old claim and cannot be automatically rebound to a new PVC.

---

## Key Takeaways

- Containers lose data on deletion — use PVs/PVCs for anything stateful
- PVs are cluster-scoped; PVCs are namespace-scoped
- Static provisioning = admin creates PV manually
- Dynamic provisioning = StorageClass creates PV automatically from a PVC
- `Retain` policy = safe but requires manual cleanup
- `Delete` policy = automatic cleanup, careful in production
- `volumeMounts[].name` must always match `volumes[].name` in the Pod spec
- Stuck `Terminating` resources → remove finalizers with `kubectl patch`
