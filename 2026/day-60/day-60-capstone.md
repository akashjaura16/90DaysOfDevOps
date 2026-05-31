# Day 60 – Capstone: Deploy WordPress + MySQL on Kubernetes

## What I Built Today

Ten days of Kubernetes learning — all in one deployment. Today I combined every major concept into a real production-style WordPress + MySQL stack running in a dedicated `capstone` namespace on Kind. Namespaces, Secrets, ConfigMaps, StatefulSets, Deployments, Services, PVCs, resource limits, health probes, HPA, and Helm — twelve concepts, one application.

---

## Project Structure

```
day-60/
├── namespace.yaml
├── secrets.yaml
├── service.yaml          # Headless MySQL service
├── deployment.yaml       # MySQL StatefulSet
├── configmap.yaml        # WordPress DB config
├── Nodeport.yaml         # WordPress NodePort service
├── hpa.yaml              # Horizontal Pod Autoscaler
└── README.md
```

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │        capstone namespace        │
                        │                                  │
  Browser               │  ┌──────────────────────┐       │
  :8080 ──port-forward──►  │  WordPress Deployment │       │
                        │  │  (2 replicas)         │       │
                        │  │  liveness + readiness │       │
                        │  │  probe: /wp-login.php │       │
                        │  └──────────┬───────────┘       │
                        │             │ envFrom ConfigMap  │
                        │             │ secretKeyRef       │
                        │  ┌──────────▼───────────┐       │
                        │  │  HPA (2-10 replicas) │       │
                        │  │  CPU target: 50%      │       │
                        │  └──────────────────────┘       │
                        │                                  │
                        │  ┌──────────────────────┐       │
                        │  │  MySQL StatefulSet    │       │
                        │  │  mysql-0              │       │
                        │  │  PVC: 1Gi             │       │
                        │  └──────────────────────┘       │
                        │             ▲                    │
                        │  Headless Service (ClusterIP:None│
                        │  mysql-0.mysql.capstone.svc...   │
                        └─────────────────────────────────┘
```

---

## Task 1: Namespace + Secret

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capstone
```

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app
  namespace: capstone
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: test@123
  MYSQL_DATABASE: wordpress
  MYSQL_USER: admin
  MYSQL_PASSWORD: test@123
```

**Why `stringData`?** Plain text values — Kubernetes handles base64 encoding automatically. No manual encoding needed.

---

## Task 2: MySQL StatefulSet + Headless Service

```yaml
# service.yaml (Headless)
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: capstone
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

```yaml
# deployment.yaml (StatefulSet)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: capstone
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          envFrom:
            - secretRef:
                name: app
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

**Why Headless?** `clusterIP: None` gives each pod a stable DNS entry (`mysql-0.mysql.capstone.svc.cluster.local`) instead of a virtual IP — essential for StatefulSets.

**Verify MySQL:**
```bash
kubectl exec -it mysql-0 -n capstone -- mysql -u admin -ptest@123 -e "SHOW DATABASES;"
```

---

## Task 3: WordPress Deployment

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: capstone
data:
  WORDPRESS_DB_HOST: "mysql-0.mysql.capstone.svc.cluster.local:3306"
  WORDPRESS_DB_NAME: "wordpress"
```

```yaml
# wordpress deployment (key sections)
envFrom:
  - configMapRef:
      name: wordpress-config
env:
  - name: WORDPRESS_DB_USER
    valueFrom:
      secretKeyRef:
        name: app
        key: MYSQL_USER
  - name: WORDPRESS_DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app
        key: MYSQL_PASSWORD
livenessProbe:
  httpGet:
    path: /wp-login.php
    port: 80
  initialDelaySeconds: 60
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /wp-login.php
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Why `initialDelaySeconds: 60`?** WordPress needs time to connect to MySQL and initialise. Too low causes crash loops.

---

## Task 4: NodePort Service

```yaml
# Nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: capstone
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30081
```

**Access on Kind:**
```bash
kubectl port-forward svc/wordpress 8080:80 -n capstone
# Open http://localhost:8080
```

---

## Task 5: Self-Healing and Persistence

### Test 1 — WordPress pod deletion
```bash
kubectl delete pod wordpress-7864f9479c-7wq2g -n capstone
# Deployment recreated it within seconds — 0 downtime
```

### Test 2 — MySQL pod deletion
```bash
kubectl delete pod mysql-0 -n capstone
# pod "mysql-0" deleted
# StatefulSet recreated mysql-0 in 5 seconds
```

### Result
```
NAME                          READY   STATUS    RESTARTS   AGE
mysql-0                       1/1     Running   0          5s     ← recreated
wordpress-7864f9479c-7wq2g    1/1     Running   0          3m6s
wordpress-7864f9479c-jglwf    1/1     Running   0          3m6s
```

**Blog post survived** — data lives in the PVC, not the pod. Deleting the pod never touches the volume.

```
NAME                       STATUS   CAPACITY   ACCESS MODES
mysql-data-mysql-0         Bound    1Gi        RWO
```

---

## Task 6: HPA

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
  namespace: capstone
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**Verify:**
```
NAME            REFERENCE              TARGETS      MINPODS   MAXPODS   REPLICAS
wordpress-hpa   Deployment/wordpress   cpu: 1%/50%  2         10        2
```

Metrics-server was already running — HPA showed real CPU data immediately.

---

## Task 7: Helm Comparison

```bash
helm install wp-helm ./wordpress-31.0.2.tgz --namespace helm-wordpress
```

### Manual vs Helm — Resource Count

| Resource | Manual (capstone) | Helm (helm-wordpress) |
|---|---|---|
| Pods | 3 | 2 |
| Deployments | 1 | 1 |
| StatefulSets | 1 (MySQL 8.0) | 1 (MariaDB) |
| Services | 2 | 3 |
| ConfigMaps | 1 | 2 |
| Secrets | 2 | 3 |
| PVCs | 1 × 1Gi | 2 × 18Gi total |
| ServiceAccounts | 1 (default) | 3 (dedicated per component) |
| HPA | ✅ configured | ❌ not included |
| **Total** | **~11** | **~15** |

### Key Differences

**Database:** Manual used `mysql:8.0` (my choice). Helm defaulted to MariaDB — compatible but not identical.

**Storage:** Manual provisioned 1Gi. Helm provisioned 8Gi (MariaDB) + 10Gi (WordPress media) = 18Gi.

**Networking:** Manual used NodePort. Helm created a LoadBalancer with HTTP + HTTPS (ports 80 + 443).

**Verdict:**
- Manual = full control, deep understanding, ideal for learning
- Helm = one command, production-ready defaults, ideal for speed

---

## Task 8: Cleanup

```bash
helm uninstall wp-helm -n helm-wordpress
kubectl delete namespace helm-wordpress
kubectl delete namespace capstone
kubectl config set-context --current --namespace=default
```

Deleting the namespace removed everything — pods, services, PVCs, secrets, configmaps, HPA. Clean slate confirmed.

---

## Concepts Used — Day Reference

| Concept | Day Learned | Used For |
|---|---|---|
| Namespaces | Day 52 | Isolating the capstone stack |
| Secrets | Day 54 | MySQL credentials |
| ConfigMaps | Day 54 | WordPress DB host + name |
| StatefulSets | Day 55 | MySQL with stable identity |
| Headless Service | Day 55 | Stable DNS for mysql-0 |
| PersistentVolumeClaims | Day 55 | MySQL data persistence |
| Deployments | Day 52 | WordPress replicas |
| NodePort Service | Day 53 | Exposing WordPress |
| Resource Limits | Day 57 | CPU + memory guardrails |
| Liveness + Readiness Probes | Day 57 | WordPress health checks |
| HPA | Day 58 | Auto-scaling WordPress |
| Helm | Day 59 | Comparing managed deployments |

---

## What Was Hardest

**StatefulSet DNS** — getting `WORDPRESS_DB_HOST` exactly right (`mysql-0.mysql.capstone.svc.cluster.local:3306`) took a few tries. One typo and WordPress can't connect to MySQL at all.

**Probe timing** — `initialDelaySeconds` needed tuning. WordPress takes longer to boot than a simple API — too low causes crash loops before the app is ready.

**NodePort conflict** — port 30080 was already allocated. Switched to 30081. Always check `kubectl get svc -A` before picking a nodePort.

## What Clicked

The StatefulSet + PVC relationship finally made sense. The pod is ephemeral — it can die and come back. The data lives independently in the PVC. That's why `mysql-0` coming back in 5 seconds with all data intact works — the volume was never touched.

## What I Would Add for Production

- TLS via cert-manager + Ingress (not NodePort)
- Namespace-level NetworkPolicies to restrict pod-to-pod traffic
- External Secrets Operator instead of plain Kubernetes Secrets
- Velero for PVC backup and disaster recovery
- Proper resource quotas on the namespace

---

## Final State — `kubectl get all -n capstone`

```
NAME                             READY   STATUS    RESTARTS
pod/mysql-0                      1/1     Running   0
pod/wordpress-7864f9479c-7wq2g   1/1     Running   0
pod/wordpress-7864f9479c-jglwf   1/1     Running   0

NAME                TYPE        CLUSTER-IP      PORT(S)
service/mysql       ClusterIP   None            3306/TCP
service/wordpress   NodePort    10.96.184.180   80:30081/TCP

NAME                        READY   UP-TO-DATE   AVAILABLE
deployment.apps/wordpress   2/2     2            2

NAME                     READY
statefulset.apps/mysql   1/1

NAME                                                TARGETS       MINPODS   MAXPODS
horizontalpodautoscaler.autoscaling/wordpress-hpa   cpu: 1%/50%   2         10
```

---
