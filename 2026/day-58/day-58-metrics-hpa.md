# Day 58 – Metrics Server and Horizontal Pod Autoscaler (HPA)

## What is the Metrics Server?
The Metrics Server is a cluster-wide aggregator of resource usage data. It collects
CPU and memory usage from kubelets every 15 seconds and exposes them via the
Kubernetes API. HPA queries this API to make scaling decisions — without it,
`kubectl top` returns nothing and HPA targets show `<unknown>`.

## How HPA Calculates Desired Replicas
HPA uses this formula:
desiredReplicas = ceil(currentReplicas × (currentUsage / targetUsage))

Example: 1 pod at 237% of a 50% target → ceil(1 × 237/50) = ceil(4.74) = 5 pods

## autoscaling/v1 vs v2
| Feature | v1 | v2 |
|---|---|---|
| CPU scaling | ✅ | ✅ |
| Memory scaling | ❌ | ✅ |
| Custom metrics | ❌ | ✅ |
| Behavior control | ❌ | ✅ |

`autoscaling/v2` adds fine-grained `behavior` blocks to control scale-up/down
speed and stabilization windows — critical for production to avoid flapping.

## What the behavior Section Controls
- **scaleUp.stabilizationWindowSeconds: 0** — scale up instantly on spike
- **scaleUp policy: 100% / 15s** — can double pod count every 15 seconds
- **scaleDown.stabilizationWindowSeconds: 300** — wait 5 min before scaling down
- **scaleDown policy: 1 pod / 60s** — remove pods gradually to avoid thrashing

## Results Observed
- Idle CPU: 0%/50% → 1 replica
- Under load: peaked at 237%/50% → HPA scaled to 6 replicas
- After load stopped: CPU dropped to 3%, cool-down period held at 6 replicas
- Scale-down began after 300s stabilization window

## Key Learnings
- `resources.requests.cpu` is mandatory for HPA — without it targets show `<unknown>`
- `kubectl top` shows actual usage; `kubectl describe pod` shows configured requests
- Scale-up is aggressive; scale-down is conservative by design
- HPA works on Deployments, StatefulSets, and ReplicaSets