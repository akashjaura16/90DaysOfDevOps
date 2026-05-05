# Day 50 – Kubernetes Architecture and Cluster Setup

## Kubernetes History (In My Own Words)

Kubernetes was created by Google in 2014, inspired by their internal container orchestration system called **Borg**, which Google had used for over a decade to run workloads at massive scale. The name "Kubernetes" comes from the Greek word for **"helmsman"** or "pilot" — the person who steers a ship. Google open-sourced it and donated it to the **Cloud Native Computing Foundation (CNCF)**. It solves the core problem Docker alone cannot: orchestrating hundreds or thousands of containers across multiple machines, with automatic scheduling, self-healing, scaling, and load balancing.

---

## Kubernetes Architecture

### Control Plane (Master Node)

```
┌──────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                         │
│                                                              │
│  ┌──────────────┐  ┌────────┐  ┌───────────────────────────┐ │
│  │  API Server  │  │  etcd  │  │        Scheduler          │ │
│  │ (front door) │  │  (DB)  │  │  (assigns pods to nodes)  │ │
│  └──────────────┘  └────────┘  └───────────────────────────┘ │
│                                                              │
│              ┌────────────────────────────┐                  │
│              │     Controller Manager     │                  │
│              │  (watches & enforces       │                  │
│              │   desired state)           │                  │
│              └────────────────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

### Worker Node

```
┌──────────────────────────────────────────────────────────────┐
│                         WORKER NODE                          │
│                                                              │
│  ┌──────────┐   ┌─────────────┐   ┌───────────────────────┐  │
│  │  kubelet │   │  kube-proxy │   │  Container Runtime    │  │
│  │ (manages │   │ (networking │   │  (containerd / CRI-O) │  │
│  │   pods)  │   │   & rules)  │   │                       │  │
│  └──────────┘   └─────────────┘   └───────────────────────┘  │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │                      PODS                            │   │
│   │   [Container]    [Container]    [Container]          │   │
│   └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### What Happens When You Run `kubectl apply -f pod.yaml`?

1. **kubectl** sends the YAML to the **API Server** via HTTP/REST
2. **API Server** authenticates, validates, and writes desired state to **etcd**
3. **Scheduler** notices an unscheduled pod, picks the best worker node, updates etcd
4. **kubelet** on the chosen node sees the assignment, tells the **Container Runtime** to pull and start the container
5. **kube-proxy** updates networking rules so the pod is reachable
6. **Controller Manager** continuously watches to ensure actual state matches desired state

### What If Components Go Down?

| Scenario | Impact |
|----------|--------|
| API Server goes down | No new commands work — existing pods keep running |
| etcd goes down | Cluster loses its brain — no state changes possible |
| Worker node goes down | Controller Manager detects it; pods are rescheduled to healthy nodes |

---

## Tool Choice: kind

**I chose kind (Kubernetes in Docker)** because:
- Runs entirely inside Docker containers — no VM overhead
- Fast to spin up and tear down (great for local dev/testing)
- Works perfectly on Windows with Docker Desktop
- Already familiar with Docker from Days 29–41 of the challenge

---

## Cluster Setup

### kubectl Installation (Windows PowerShell)

```powershell
$version = (Invoke-WebRequest -Uri "https://dl.k8s.io/release/stable.txt" -UseBasicParsing).Content.Trim()
Invoke-WebRequest -Uri "https://dl.k8s.io/release/$version/bin/windows/amd64/kubectl.exe" -OutFile "kubectl.exe" -UseBasicParsing
.\kubectl.exe version --client
# Client Version: v1.36.0
```

> Note: On Windows, `curl` is an alias for `Invoke-WebRequest` — Linux-style flags like `-L`, `-O` don't work.

### kind Cluster Setup

```powershell
kind create cluster --name devops-cluster
kubectl cluster-info --context kind-devops-cluster
kubectl get nodes
```

---

## Output

### `kubectl get nodes`
```
NAME                           STATUS   ROLES           AGE   VERSION
devops-cluster-control-plane   Ready    control-plane   65s   v1.35.0
```

### `kubectl get pods -A`
```
NAMESPACE            NAME                                                   READY   STATUS
kube-system          coredns-7d764666f9-nqz4h                               1/1     Running
kube-system          coredns-7d764666f9-v2pt4                               1/1     Running
kube-system          etcd-devops-cluster-control-plane                      1/1     Running
kube-system          kindnet-d2ktq                                          1/1     Running
kube-system          kube-apiserver-devops-cluster-control-plane            1/1     Running
kube-system          kube-controller-manager-devops-cluster-control-plane   1/1     Running
kube-system          kube-proxy-rjxlx                                       1/1     Running
kube-system          kube-scheduler-devops-cluster-control-plane            1/1     Running
```

---

## kube-system Pods Explained

| Pod | What It Does |
|-----|--------------|
| `etcd-*` | Key-value database storing ALL cluster state |
| `kube-apiserver-*` | Front door — every kubectl command hits this |
| `kube-controller-manager-*` | Watches cluster, enforces desired state |
| `kube-scheduler-*` | Decides which node a new pod runs on |
| `kube-proxy-*` | Manages networking rules for pod communication |
| `coredns-*` | DNS for the cluster — pods find each other by name |
| `kindnet-*` | kind's CNI plugin — handles pod networking |

---

## kubeconfig

**What is a kubeconfig?** A YAML file that tells `kubectl` how to connect to clusters — stores cluster addresses, auth credentials, and context names.

**Where is it stored?** `C:\Users\akash\.kube\config`

```bash
kubectl config current-context   # which cluster am I on?
kubectl config get-contexts      # list all clusters
kubectl config view              # full config file
```

---

## Cluster Lifecycle

```bash
kind delete cluster --name devops-cluster
kind create cluster --name devops-cluster
kubectl get nodes
```

---

*Day 50 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
