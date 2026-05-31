# Day 59 – Helm: The Kubernetes Package Manager

## What is Helm?

Helm is the package manager for Kubernetes — think of it like `apt` for Ubuntu or `brew` for macOS. Instead of writing and maintaining dozens of individual YAML files (Deployments, Services, ConfigMaps, Secrets, PVCs) for every application, Helm bundles them into a single reusable package called a **chart**.

---

## Three Core Concepts

| Concept | Description |
|---|---|
| **Chart** | A package of Kubernetes manifest templates. Like a `.deb` or `.zip` of your app's entire config. |
| **Release** | A specific installation of a chart in your cluster. You can install the same chart multiple times as different releases. |
| **Repository** | A collection of charts hosted online — like a package registry (e.g. Bitnami, ArtifactHub). |

---

## Installation

```bash
# macOS
brew install helm

# Linux (curl script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
helm env
```

---

## Working with Repositories

```bash
# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update local cache
helm repo update

# Search available charts
helm search repo nginx
helm search repo bitnami
```

---

## Installing a Chart

```bash
# One command deploys a Deployment, Service, and ConfigMap automatically
helm install my-nginx bitnami/nginx

# Check what was created in the cluster
kubectl get all

# Inspect the release
helm list
helm status my-nginx
helm get manifest my-nginx
```

A single `helm install` replaces writing 3–5 YAML files by hand.

---

## Customising with Values

Every chart exposes configurable defaults via a `values.yaml` file.

```bash
# View all configurable defaults
helm show values bitnami/nginx

# Override inline with --set
helm install my-nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# Or override with a values file
helm install my-nginx-file bitnami/nginx -f custom-values.yaml

# Check what overrides are active
helm get values my-nginx-file
```

### custom-values.yaml

```yaml
# custom-values.yaml
# Custom values for bitnami/nginx release

# Number of nginx pod replicas to run
replicaCount: 3

# Expose the service as NodePort for local cluster access
service:
  type: NodePort

# Resource limits to prevent a single pod consuming too much cluster capacity
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

---

## Upgrading and Rolling Back

```bash
# Upgrade a release — scales replicas to 5
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# View full revision history
helm history my-nginx

# Rollback to revision 1
helm rollback my-nginx 1

# Check history again — rollback creates revision 3, does NOT overwrite revision 2
helm history my-nginx
```

> This is the same concept as Deployment rollouts (`kubectl rollout undo`) but operating at the full application stack level — all resources managed by the chart are rolled back together.

After the rollback you will see **3 revisions** in history.

---

## Creating Your Own Chart

```bash
# Scaffold a new chart (generates the full directory structure)
helm create my-app
```

### Chart directory structure

```
my-app/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configurable values
└── templates/
    ├── deployment.yaml # Kubernetes Deployment template
    ├── service.yaml    # Kubernetes Service template
    ├── _helpers.tpl    # Reusable template helpers
    └── NOTES.txt       # Post-install usage notes
```

### Go template syntax

Helm uses Go templating to inject values into Kubernetes manifests at install time:

| Template expression | What it resolves to |
|---|---|
| `{{ .Values.replicaCount }}` | Value from `values.yaml` or `--set` override |
| `{{ .Chart.Name }}` | Chart name from `Chart.yaml` |
| `{{ .Release.Name }}` | The release name given at `helm install` |
| `{{ .Release.Namespace }}` | The Kubernetes namespace of the release |

Example from `templates/deployment.yaml`:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### Workflow for a custom chart

```bash
# Edit values.yaml — set replicaCount: 3, image: nginx:1.25

# Validate chart structure
helm lint my-app

# Preview rendered manifests without installing
helm template my-release ./my-app

# Install the chart
helm install my-release ./my-app

# Upgrade (scale to 5 replicas)
helm upgrade my-release ./my-app --set replicaCount=5
```

> `helm template` is the most useful debugging tool — it renders the final YAML without touching the cluster, so you can catch template errors before deploying.

---

## Useful Commands Reference

```bash
helm show values <chart>          # See all configurable values
helm get values <release>         # See active overrides for a release
helm get values <release> --all   # See all values including defaults
helm list                         # List all releases
helm history <release>            # Full revision history
helm template <release> <chart>   # Render manifests without installing
helm lint <chart-dir>             # Validate chart before installing
helm uninstall <release>          # Remove a release and all its resources
helm uninstall <release> --keep-history  # Remove but retain history for auditing
```

---

## Key Takeaways

- **Before Helm:** 5–10 YAML files per application, manually maintained per environment.
- **After Helm:** one chart, version-controlled, upgradeable, rollback-capable, reusable across environments.
- Rollbacks create a **new revision** — history is never destroyed.
- `helm template` + `helm lint` before every install is a production best practice.
- Nested value overrides use dot notation: `--set service.type=NodePort`.
