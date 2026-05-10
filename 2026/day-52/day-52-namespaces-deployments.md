# Day 52 – Kubernetes Namespaces and Deployments

## What Are Namespaces?

A **Namespace** is a virtual cluster inside your Kubernetes cluster. It lets you isolate and organise resources so different teams, environments, or projects don't interfere with each other.

**Built-in namespaces:**

| Namespace | Purpose |
|---|---|
| `default` | Where resources go if you don't specify a namespace |
| `kube-system` | Kubernetes internal components (API server, scheduler, etc.) |
| `kube-public` | Publicly readable resources |
| `kube-node-lease` | Node heartbeat tracking |

**Why use namespaces?**
- Separate environments: `dev`, `staging`, `production`
- Apply resource quotas per namespace
- Control access with RBAC per namespace
- Avoid name collisions (two teams can each have a pod named `nginx`)

---

## What Are Deployments?

A **Deployment** is the recommended way to run applications in Kubernetes. Instead of creating individual Pods, you declare a desired state (e.g. "3 replicas of nginx") and Kubernetes continuously works to maintain that state.

**Deployment vs Standalone Pod:**

| | Standalone Pod | Deployment |
|---|---|---|
| Auto-restarts on crash | ❌ | ✅ |
| Scaling | ❌ Manual | ✅ `kubectl scale` |
| Rolling updates | ❌ | ✅ Zero-downtime |
| Rollback | ❌ | ✅ `kubectl rollout undo` |

---

## My Deployment Manifest

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
```

**Explanation of each section:**

- `apiVersion: apps/v1` — Deployments live in the `apps` API group, not the core `v1` group
- `kind: Deployment` — tells Kubernetes this is a Deployment resource
- `metadata.namespace: dev` — deploys into the `dev` namespace
- `spec.replicas: 3` — Kubernetes will always maintain 3 running pods
- `spec.selector.matchLabels` — how the Deployment finds and tracks its pods (must match the template labels)
- `spec.template` — the Pod blueprint; every pod the Deployment creates uses this spec
- `containers.image: nginx:1.24` — pinned image version (best practice over `latest`)

---

## Self-Healing Behaviour

When a Pod managed by a Deployment is deleted, the **ReplicaSet controller** detects the count dropped below desired (3 → 2) and immediately creates a replacement.

```bash
kubectl delete pod nginx-deployment-xxxxx -n dev
# Within seconds, a NEW pod with a different name appears
kubectl get pods -n dev
```

The replacement pod gets a **new name** — it's a brand new pod, not the same one restarted.

With a standalone pod: delete it and it's gone permanently.

---

## Scaling

**Imperative (quick, not tracked in Git):**
```bash
kubectl scale deployment nginx-deployment --replicas=5 -n dev
kubectl scale deployment nginx-deployment --replicas=2 -n dev
```

**Declarative (change YAML, apply — tracked in Git):**
```yaml
spec:
  replicas: 4
```
```bash
kubectl apply -f nginx-deployment.yaml
```

When scaling down, Kubernetes **terminates the extra pods gracefully** — it doesn't just kill them instantly.

---

## Rolling Updates and Rollbacks

**Rolling update** — Kubernetes replaces pods one by one. New pods must be healthy before old ones are removed. Zero downtime.

```bash
# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev

# Watch live
kubectl rollout status deployment/nginx-deployment -n dev

# Check history
kubectl rollout history deployment/nginx-deployment -n dev
```

**Rollback** — revert to the previous revision instantly:

```bash
kubectl rollout undo deployment/nginx-deployment -n dev

# Verify image rolled back
kubectl describe deployment nginx-deployment -n dev | grep Image
```

---

## Commands Reference

```bash
# Namespaces
kubectl get namespaces
kubectl create namespace dev
kubectl apply -f namespace.yaml
kubectl delete namespace dev

# Pods across namespaces
kubectl get pods -n dev
kubectl get pods -A          # all namespaces

# Deployments
kubectl apply -f nginx-deployment.yaml
kubectl get deployments -n dev
kubectl get pods -n dev
kubectl describe deployment nginx-deployment -n dev

# Scaling
kubectl scale deployment nginx-deployment --replicas=5 -n dev

# Rolling update & rollback
kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
kubectl rollout status deployment/nginx-deployment -n dev
kubectl rollout history deployment/nginx-deployment -n dev
kubectl rollout undo deployment/nginx-deployment -n dev

# Behind the scenes
kubectl get replicasets -n dev

# Clean up
kubectl delete deployment nginx-deployment -n dev
kubectl delete namespace dev staging production
```

---

## Key Takeaways

1. Namespaces = isolation. Always deploy into a named namespace, not `default`
2. Deployments = self-healing + scaling + zero-downtime updates
3. The `selector.matchLabels` MUST match `template.metadata.labels` — if not, the Deployment can't track its pods
4. Rolling updates replace pods one by one — old pod only dies after new one is healthy
5. Deleting a namespace deletes **everything** inside it

---

