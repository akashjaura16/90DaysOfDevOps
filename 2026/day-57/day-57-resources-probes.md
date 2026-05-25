# Day 57 – Resource Requests, Limits, and Probes

## Overview

Today covered two critical Kubernetes production concepts:
- **Resource requests and limits** — how Kubernetes schedules and enforces CPU/memory boundaries
- **Liveness, readiness, and startup probes** — how Kubernetes detects and recovers from failures automatically

---

## Task 1: Resource Requests and Limits

### Concepts

| Term | Role | Who uses it |
|------|------|-------------|
| `requests` | Guaranteed minimum | Scheduler (for pod placement) |
| `limits` | Maximum allowed | Kubelet (enforced at runtime) |

- **CPU** is measured in millicores: `100m` = 0.1 CPU, `1` = 1 full core, `1000m` = 1 core
- **Memory** is measured in mebibytes/gibibytes: `128Mi`, `1Gi`
- CPU has **no** `Mi`/`Gi` suffix — only `m` (millicores) or whole numbers (cores)

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: resource-demo-ctr
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "250m"
        memory: "256Mi"
```

### QoS Classes

| Class | Condition |
|-------|-----------|
| `Guaranteed` | requests == limits for all containers |
| `Burstable` | requests < limits (our pod above) |
| `BestEffort` | no requests or limits set at all |

**Verified QoS Class: `Burstable`** — because requests (100m/128Mi) differ from limits (250m/256Mi).

---

## Task 2: OOMKilled — Exceeding Memory Limits

### Key Concept
- **CPU** is *compressible* → throttled when over limit, never killed
- **Memory** is *incompressible* → container is **killed immediately** with no warning

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-oom
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"       # hard ceiling
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    #                         ↑ tries to allocate 200M > 100Mi limit
```

### Observed Output

```
NAME              READY   STATUS             RESTARTS   AGE
memory-demo-oom   0/1     OOMKilled          0          4s
memory-demo-oom   1/1     Running            1          6s   ← k8s restarted it
memory-demo-oom   0/1     OOMKilled          1          7s   ← killed again
memory-demo-oom   0/1     CrashLoopBackOff   1          8s   ← backoff kicks in
```

### From `kubectl describe pod`:

```
State:      Terminated
  Reason:   OOMKilled
  Exit Code: 137
Restart Count: 3
QoS Class: Burstable
```

**Verified exit code: `137`** = 128 + SIGKILL (signal 9)

---

## Task 3: Pending Pod — Requesting Too Much

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-big-pod
spec:
  containers:
  - name: big
    image: nginx
    resources:
      requests:
        cpu: "100"      # 100 full cores — no node has this
        memory: "128Gi" # 128 gibibytes — also impossible
```

### Key Insight
`cpu: "100"` means **100 whole cores** — not millicores, not Mi/Gi (those are memory-only units).
There is no `Mi` or `Gi` for CPU. The scheduler cannot find any node with 100 cores, so the pod stays `Pending` forever.

### Observed Output

```
Status:      Pending
PodScheduled: False
```

### Scheduler Event Message

```
Warning  FailedScheduling  0/2 nodes are available: 1 Insufficient cpu,
1 Insufficient memory, 1 node(s) had untolerated taint(s).
preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```

**Verified:** The scheduler explicitly names the reason — insufficient CPU and memory on every available node.

---

## Task 4: Liveness Probe

### Concept
A liveness probe detects **stuck or broken containers**. If it fails `failureThreshold` times consecutively, Kubernetes **restarts** the container.

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  labels:
    test: liveness
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/busybox:1.27.2
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

### How it works
1. Container starts → creates `/tmp/healthy`
2. Liveness probe runs `cat /tmp/healthy` every 5s → passes ✓
3. After 30s → `/tmp/healthy` is deleted
4. Probe fails 3 consecutive times (15s) → Kubernetes **restarts** the container
5. Cycle repeats

### From `kubectl describe pod`:

```
Liveness: exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=3
Restart Count: 1+
```

**Verified:** Container restarts automatically after 3 consecutive liveness failures.

---

## Task 5: Readiness Probe

### Concept
A readiness probe controls **traffic routing**. Failure removes the Pod from Service endpoints but **does NOT restart** the container.

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: readiness-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Steps and Results

```powershell
# Expose as service
kubectl expose pod readiness-pod --port=80 --name=readiness-svc

# Endpoints with pod healthy
kubectl get endpoints readiness-svc
# NAME            ENDPOINTS        AGE
# readiness-svc   10.244.1.25:80   9s   ← pod IP listed

# Break the probe
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html

# Pod status after probe fails
kubectl get pod readiness-pod
# NAME            READY   STATUS    RESTARTS   AGE
# readiness-pod   0/1     Running   0          100s  ← 0/1 but RESTARTS still 0

# Endpoints after probe fails
kubectl get endpoints readiness-svc
# NAME            ENDPOINTS   AGE
# readiness-svc               46s   ← empty, pod removed from traffic
```

### Liveness vs Readiness

| Probe | On failure | Use case |
|-------|-----------|----------|
| Liveness | Restarts container | Detect deadlocks, crashes |
| Readiness | Removes from endpoints | Temporary unavailability, warm-up |

**Verified:** When readiness failed — **No**, the container was NOT restarted. `RESTARTS = 0` throughout.

---

## Task 6: Startup Probe

### Concept
A startup probe gives **slow-starting containers** time to initialise. While the startup probe is running, **liveness and readiness probes are completely disabled** — preventing premature kills.

### Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-pod
spec:
  containers:
  - name: slow-start
    image: registry.k8s.io/busybox:1.27.2
    command: ["/bin/sh", "-c", "sleep 20 && touch /tmp/started && sleep 3600"]
    startupProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 5
      failureThreshold: 12      # 5 × 12 = 60 second budget
    livenessProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 10
      failureThreshold: 3       # only activates after startup succeeds
```

### Observed Timeline

```
NAME                READY   STATUS    RESTARTS   AGE
startup-probe-pod   0/1     Running   0          5s   ← startup probe checking
startup-probe-pod   0/1     Running   0          25s  ← sleep 20 done, file created
startup-probe-pod   1/1     Running   0          25s  ← startup passed, liveness active ✓
```

### Probe Budget Calculation

```
budget = periodSeconds × failureThreshold
       = 5 × 12 = 60 seconds
```

The container takes 20 seconds to start — well within the 60 second budget.

### Verified: What if `failureThreshold` were 2 instead of 12?

| Setting | Budget | Outcome |
|---------|--------|---------|
| `failureThreshold: 12` | 5 × 12 = 60s | Pod starts fine — 20s sleep fits within budget |
| `failureThreshold: 2` | 5 × 2 = 10s | Probe gives up at 10s — `/tmp/started` doesn't exist yet — container gets **killed and enters CrashLoopBackOff** |

With `failureThreshold: 2`, the startup probe exhausts its budget (10s) before `sleep 20` finishes. Kubernetes kills the container before it ever becomes healthy.

---

## Summary: Probe Comparison

| Probe | Trigger | On failure | Liveness active? |
|-------|---------|-----------|-----------------|
| `startupProbe` | Container initialising | Kill + restart | No (disabled until startup passes) |
| `livenessProbe` | Container stuck/crashed | Kill + restart | Yes |
| `readinessProbe` | App temporarily unavailable | Remove from endpoints | Yes |

## Summary: Resource Limits

| Resource | Over limit behaviour | Unit |
|----------|---------------------|------|
| CPU | Throttled (never killed) | `m` millicores or whole cores |
| Memory | OOMKilled instantly (exit 137) | `Mi`, `Gi` |

---

## Cleanup

```powershell
kubectl delete pod memory-demo memory-demo-oom too-big-pod
kubectl delete pod liveness-exec readiness-pod startup-probe-pod
kubectl delete service readiness-svc
```
