# Day 53 – Kubernetes Services

## What Problem Do Services Solve?

Every Pod gets its own IP address, but there are two problems:
1. Pod IPs are **not stable** — when a Pod restarts, it gets a new IP
2. A Deployment runs **multiple Pods** — which IP do you connect to?

A Service solves both problems by providing:
- A **stable IP and DNS name** that never changes
- **Load balancing** across all Pods matching its selector

```
[Client] --> [Service (stable IP)] --> [Pod 1]
                                   --> [Pod 2]
                                   --> [Pod 3]
```

---

## Task 1: Deployment

`app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

> Note: Used `nginx:alpine` instead of `nginx:1.25` because the local cluster had no internet access. Images were pre-loaded into the cluster using `kind load docker-image`.

---

## Task 2: ClusterIP Service (Internal Access Only)

`clusterip-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

**Key fields:**
- `selector: app: web-app` — routes traffic to all Pods with matching label
- `port: 80` — the port the **Service** listens on (what clients connect to)
- `targetPort: 80` — the port on the **Pod** that traffic is forwarded to

> `port` and `targetPort` don't have to match. For example, your Service can listen on port 8080 but forward to a Pod running on port 80.

**Test from inside cluster:**
```bash
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
wget -qO- http://web-app-clusterip
```

Output: nginx welcome page — Service successfully load-balanced to one of the 3 Pods.

---

## Task 3: Kubernetes DNS for Service Discovery

Every Service gets a DNS entry automatically:
```
<service-name>.<namespace>.svc.cluster.local
```

**Short name** (same namespace):
```bash
wget -qO- http://web-app-clusterip
```

**Full DNS name** (cross-namespace):
```bash
wget -qO- http://web-app-clusterip.default.svc.cluster.local
```

Both resolve to the same ClusterIP `10.96.234.218`.

**Rule:**
- Use short name when communicating **within the same namespace**
- Use full DNS name when reaching **across namespaces**

```bash
nslookup web-app-clusterip
# Returns: web-app-clusterip.default.svc.cluster.local → 10.96.234.218
```

---

## Task 4: NodePort Service (External via Node)

`nodeport-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

**Traffic flow:**
```
<NodeIP>:30080 --> Service:80 --> Pod:80
```

- `nodePort: 30080` — port opened on every node (must be 30000–32767)
- Accessible from outside the cluster via node IP

---

## Task 5: LoadBalancer Service (Cloud External Access)

`loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

**Why is EXTERNAL-IP `<pending>`?**

On a local cluster there is no cloud provider (AWS/GCP/Azure). Kubernetes is waiting for a cloud provider to provision a real external load balancer and assign a public IP — but no one responds, so it stays `<pending>` forever.

In a real cloud cluster:
```
web-app-loadbalancer   LoadBalancer   10.96.183.106   52.14.23.101   80:32442/TCP
#                                                     ^^^^^^^^^^^^
#                                                     real public IP from AWS/GCP
```

---

## Task 6: Service Types Comparison

```bash
kubectl get services -o wide
```

| Type | Accessible From | Use Case |
|------|----------------|----------|
| ClusterIP | Inside cluster only | Internal communication between services |
| NodePort | Outside via `<NodeIP>:<NodePort>` | Development, testing |
| LoadBalancer | Outside via cloud load balancer | Production in cloud environments |

**They stack on top of each other:**
> LoadBalancer → creates NodePort → creates ClusterIP

Verified via `kubectl describe service web-app-loadbalancer`:

| Layer | Value |
|-------|-------|
| ClusterIP | 10.96.183.106 |
| NodePort | 32442 |
| LoadBalancer | pending (no cloud provider) |

---

## What Are Endpoints?

Endpoints are the actual Pod IPs a Service routes traffic to. When a Pod becomes Ready, it gets added to the Endpoints list. If a Pod crashes, it's removed automatically.

```bash
kubectl get endpoints web-app-clusterip
# Shows: 10.244.1.8:80, 10.244.1.10:80, 10.244.1.11:80
```

This is why the Service selector must **exactly match** Pod labels — if they don't match, Endpoints will be empty and the Service routes to nothing.

---

## Key Takeaways

- Services give Pods a **stable identity** regardless of restarts
- `port` = what clients connect to on the Service; `targetPort` = what the Pod listens on
- ClusterIP for internal traffic, NodePort for node-level access, LoadBalancer for cloud production
- Kubernetes DNS auto-creates entries for every Service
- Use short DNS names within the same namespace, full names across namespaces
- Always check `kubectl get endpoints` if a Service isn't routing traffic

---

