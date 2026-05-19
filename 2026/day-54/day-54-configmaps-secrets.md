# Day 54 – Kubernetes ConfigMaps and Secrets

## What I Learned Today

Applications need configuration — database URLs, feature flags, API keys, port numbers. Hardcoding these into your container image means rebuilding and redeploying every time a value changes. Kubernetes solves this with two dedicated resources:

- **ConfigMap** → non-sensitive config (env settings, port numbers, feature flags, config files)
- **Secret** → sensitive data (passwords, API keys, tokens)

---

## Task 1: Create a ConfigMap from Literals

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```

**Inspect it:**

```bash
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

Data is stored as **plain text** — no encoding, no encryption. All three key-value pairs are visible directly in the output.

---

## Task 2: Create a ConfigMap from a File

**Custom nginx config — `default.conf`:**

```nginx
server {
    listen 80;
    location /health {
        return 200 'healthy';
        add_header Content-Type text/plain;
    }
}
```

**Create the ConfigMap:**

```bash
kubectl create configmap nginx-config --from-file=default.conf=default.conf
```

The key name (`default.conf`) becomes the actual filename when mounted into a Pod.

```bash
kubectl get configmap nginx-config -o yaml
```

---

## Task 3: Use ConfigMaps in a Pod

### Inject all keys as environment variables (envFrom)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-env-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c", "echo APP_ENV=$APP_ENV APP_PORT=$APP_PORT APP_DEBUG=$APP_DEBUG && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
  restartPolicy: Never
```

### Mount a config file as a volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-config-pod
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: nginx-conf
      configMap:
        name: nginx-config
```

**Test the /health endpoint:**

```bash
kubectl exec nginx-config-pod -- curl -s http://localhost/health
# Output: healthy
```

---

## Task 4: Create a Secret

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cureP@ssw0rd
```

**Inspect it:**

```bash
kubectl get secret db-credentials -o yaml
```

Values appear base64-encoded in the output:

```yaml
data:
  DB_PASSWORD: czNjdXJlUEBzc3cwcmQ=
  DB_USER: YWRtaW4=
```

**Decode a value:**

```bash
echo 'czNjdXJlUEBzc3cwcmQ=' | base64 --decode
# Output: s3cureP@ssw0rd
```

> **base64 is encoding, not encryption.** Anyone with cluster access can decode it instantly. The real security benefits of Secrets are RBAC separation, tmpfs storage on nodes, and optional encryption at rest — not the encoding itself.

---

## Task 5: Use Secrets in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c", "echo DB_USER=$DB_USER && cat /etc/db-credentials/DB_PASSWORD && sleep 3600"]
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
      volumeMounts:
        - name: db-creds
          mountPath: /etc/db-credentials
          readOnly: true
  volumes:
    - name: db-creds
      secret:
        secretName: db-credentials
  restartPolicy: Never
```

**Each Secret key becomes a file inside the mount path. The file content is the decoded plaintext value — not base64.**

```bash
kubectl exec db-pod -- cat /etc/db-credentials/DB_PASSWORD
# Output: s3cureP@ssw0rd
```

---

## Task 6: ConfigMap Live Update (Volume Mount)

**Create the ConfigMap:**

```bash
kubectl create configmap live-config --from-literal=message=hello
```

**Pod that reads it in a loop:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-config-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c", "while true; do cat /etc/config/message; echo; sleep 5; done"]
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: live-config
```

**Update the ConfigMap:**

```bash
kubectl patch configmap live-config --type merge -p '{"data":{"message":"world"}}'
```

Wait 30–60 seconds — the volume-mounted value updates automatically **without restarting the Pod**.

> **Environment variables do NOT update.** They are set once at Pod startup and stay fixed until the Pod is restarted. Volume mounts reflect changes automatically.

---

## Task 7: Clean Up

```bash
# Delete Pods
kubectl delete pod app-env-pod nginx-config-pod db-pod live-config-pod

# Delete ConfigMaps
kubectl delete configmap app-config nginx-config live-config

# Delete Secret
kubectl delete secret db-credentials
```

---

## Key Concepts Summary

### ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| **Use for** | Non-sensitive config | Passwords, tokens, keys |
| **Storage** | Plain text | base64-encoded |
| **RBAC** | Standard | Can be restricted separately |
| **Node storage** | Standard | tmpfs (in-memory) |

### Environment Variables vs Volume Mounts

| | Environment Variables | Volume Mounts |
|---|---|---|
| **Use for** | Simple key-value pairs | Full config files |
| **Updates** | ❌ Set at Pod startup only | ✅ Auto-update within 60s |
| **Access** | `$ENV_VAR` | `cat /mount/path/key` |

### Why base64 is NOT encryption

base64 is a reversible encoding format — it is not a security mechanism. Any person or process with access to the Secret object can decode the values in seconds:

```bash
echo 'YWRtaW4=' | base64 --decode
# Output: admin
```

Real Secret security comes from:
- **RBAC** — restrict who/what can `get` or `list` secrets
- **Encryption at rest** — enable `EncryptionConfiguration` in the API server
- **External secret managers** — AWS Secrets Manager, HashiCorp Vault, etc.

---

## Commands Reference

```bash
# ConfigMap
kubectl create configmap <name> --from-literal=KEY=VALUE
kubectl create configmap <name> --from-file=key=filename
kubectl get configmap <name> -o yaml
kubectl describe configmap <name>
kubectl patch configmap <name> --type merge -p '{"data":{"key":"value"}}'

# Secret
kubectl create secret generic <name> --from-literal=KEY=VALUE
kubectl get secret <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 --decode

# Decode base64 (always use -n to avoid trailing newline)
echo -n 'value' | base64
echo '<encoded>' | base64 --decode
```
