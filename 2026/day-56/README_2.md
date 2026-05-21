# Day 56 — Kubernetes StatefulSets

> **90 Days of DevOps Challenge** | #DevOpsKaJosh by TrainWithShubham

---

## What I Learned

StatefulSets manage pods that need **stable identity, stable storage, and ordered deployment** — unlike Deployments where pods are interchangeable. This is essential for databases, message brokers, and any workload where each instance has a distinct role.

---

## Key Concepts

| Concept | StatefulSet | Deployment |
|---|---|---|
| Pod names | `web-0`, `web-1`, `web-2` (stable) | `web-abc12`, `web-xyz99` (random) |
| DNS per pod | ✅ Yes | ❌ No |
| Storage per pod | ✅ Individual PVCs | ❌ Shared or none |
| Creation order | Sequential (0 → 1 → 2) | Parallel |
| Deletion order | Reverse (2 → 1 → 0) | Parallel |

---

## Lab Tasks Completed

### Task 1 — Headless Service

A Headless Service (`clusterIP: None`) creates individual DNS entries per pod instead of load-balancing to one IP. StatefulSets require this.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  clusterIP: None
```

**Verified:** `CLUSTER-IP` column shows `None`

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
my-service   ClusterIP   None         <none>        80/TCP    64s
```

---

### Task 2 — StatefulSet with Persistent Storage

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: MyApp
  serviceName: "my-service"
  replicas: 3
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: MyApp
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.24
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 100Mi
```

**Bugs fixed during lab:**
- Label mismatch: pod template had `app: nginx` but selector expected `app: MyApp`
- `serviceName` pointed to `"nginx"` instead of `"my-service"`
- `storageClassName: "my-storage-class"` didn't exist — changed to `"standard"` (Docker Desktop default)

**Observed ordered creation:**
```
web-0   Running   ← starts first
web-1   Running   ← starts only after web-0 is Ready
web-2   Running   ← starts only after web-1 is Ready
```

---

### Task 3 — Stable Network Identity (DNS)

Each pod gets a DNS entry in the format:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

**Tested with busybox:**
```bash
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- sh
```

```
nslookup web-0.my-service.default.svc.cluster.local
→ 10.244.1.13

nslookup web-1.my-service.default.svc.cluster.local
→ 10.244.1.15

nslookup web-2.my-service.default.svc.cluster.local
→ 10.244.1.17
```

> **Note:** Use `busybox:1.28` specifically — newer versions have a broken `nslookup` that fails on valid DNS entries.

**After deleting and recreating `web-0`:** DNS name stayed the same (`web-0.my-service...`), only the IP changed. This proves stable identity — the name persists even when the pod is replaced.

---

### Task 4 — Stable Storage (Data Survives Pod Deletion)

Wrote unique data to each pod's PVC:

```bash
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"
```

Deleted `web-0`, waited for recovery, then read the file:

```bash
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# Output: Data from web-0  ✅
```

The new pod reconnected to `www-web-0` (same PVC) automatically — data survived.

---

### Task 5 — Clean Up

```bash
kubectl delete statefulset web
kubectl delete service my-service
kubectl get pvc        # PVCs are still there — must delete manually
kubectl delete pvc www-web-0 www-web-1 www-web-2
```

**Finding:** PVCs are NOT auto-deleted when a StatefulSet is deleted. This is intentional — Kubernetes treats storage as more valuable than the workload to prevent accidental data loss.

---

## Bugs & Fixes

| Bug | Cause | Fix |
|---|---|---|
| `StatefulSet invalid: selector does not match labels` | Pod template had `app: nginx`, selector had `app: MyApp` | Aligned both to `app: MyApp` |
| `serviceName: "nginx"` | Pointed to container name, not Service name | Changed to `serviceName: "my-service"` |
| PVC stuck in `Pending` | `storageClassName: "my-storage-class"` doesn't exist | Changed to `standard` (Docker Desktop default) |
| Markdown instructions embedded in YAML | Copy-paste from lab notes | Removed — YAML and instructions are separate files |

---

## Commands Reference

```bash
# Apply manifests
kubectl apply -f service.yaml

# Watch ordered pod creation
kubectl get pods -l app=MyApp -w

# Check PVCs
kubectl get pvc

# Check available StorageClasses
kubectl get storageclass

# DNS testing
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- sh

# Exec into a StatefulSet pod
kubectl exec -it web-0 -- sh

# Write data to a pod's volume
kubectl exec web-0 -- sh -c "echo 'hello' > /usr/share/nginx/html/index.html"

# Read data from a pod's volume
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html

# Clean up
kubectl delete statefulset web
kubectl delete service my-service
kubectl delete pvc www-web-0 www-web-1 www-web-2
```

---

## Files

```
day-56/
├── service.yaml        # Headless Service + StatefulSet (combined manifest)
└── README.md
```

---

## Connect

- GitHub: [SinghAkashdeep16](https://github.com/SinghAkashdeep16)
- Docker Hub: [akashjaura16](https://hub.docker.com/u/akashjaura16)
- Challenge: #90DaysOfDevOps #DevOpsKaJosh
