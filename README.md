# eks-devops-gitops

GitOps repository for the EKS DevOps Platform. This is the **single source of truth** for everything running on the Kubernetes cluster — application deployments, observability stack, alerting rules, and platform configuration. ArgoCD watches this repository and continuously reconciles the cluster state to match.

---

## Table of Contents

- [GitOps Philosophy](#gitops-philosophy)
- [Repository Structure](#repository-structure)
- [App of Apps Pattern](#app-of-apps-pattern)
- [Applications](#applications)
- [Blue/Green Deployment](#bluegreen-deployment)
- [Observability Stack](#observability-stack)
- [Alerting](#alerting)
- [Logging Stack](#logging-stack)
- [How to Add a New Application](#how-to-add-a-new-application)
- [Workflow](#workflow)

---

## GitOps Philosophy

> **Git is the source of truth. Nobody touches the cluster directly.**

| Traditional Approach | GitOps Approach |
|---|---|
| `kubectl apply` manually | Push to Git → ArgoCD applies |
| Configuration drift over time | ArgoCD detects and reverts drift |
| No audit trail | Every change is a Git commit |
| Hard to reproduce | `git clone` + ArgoCD = identical cluster |
| Manual rollback | `git revert` = instant rollback |

**Key ArgoCD behaviours enabled across all applications:**

```yaml
syncPolicy:
  automated:
    prune: true       # removes resources deleted from Git
    selfHeal: true    # reverts any manual kubectl changes
```

---

## Repository Structure

```
eks-devops-gitops/
│
├── bootstrap/
│   └── root-app.yaml             # App of Apps — bootstraps everything
│
├── apps/                         # ArgoCD Application CRs
│   ├── argocd/
│   │   └── app.yaml              # ArgoCD self-management
│   ├── web-app/
│   │   └── app.yaml              # FoodRush application
│   ├── monitoring/
│   │   ├── app.yaml              # kube-prometheus-stack
│   │   └── alerting-rules-app.yaml  # Custom PrometheusRules
│   └── logging/
│       ├── loki-app.yaml         # Loki log aggregation
│       └── promtail-app.yaml     # Promtail log collector
│
├── argocd/
│   ├── ingress.yaml              # ArgoCD UI ingress (ALB)
│   └── argocd-cm-patch.yaml      # PVC health check customization
│
├── k8s/
│   ├── base/                     # Base Kubernetes manifests (Kustomize)
│   │   ├── blue-deployment.yaml
│   │   ├── green-deployment.yaml
│   │   ├── service-blue.yaml
│   │   ├── service-green.yaml
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   │
│   ├── overlays/
│   │   ├── dev/                  # Dev environment overrides
│   │   └── prod/                 # Production environment overrides
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       ├── configmap.yaml
│   │       ├── hpa-blue.yaml
│   │       ├── hpa-green.yaml
│   │       ├── pdb-blue.yaml
│   │       ├── pdb-green.yaml
│   │       ├── patch-blue-image.yaml    # Current blue image tag
│   │       ├── patch-green-image.yaml   # Current green image tag
│   │       ├── release.yaml             # Active color tracker
│   │       └── traffic/
│   │           ├── traffic-blue-100.yaml
│   │           └── traffic-green-100.yaml
│   │
│   ├── monitoring/
│   │   ├── values.yaml           # kube-prometheus-stack Helm values
│   │   └── rules/
│   │       └── alerting-rules.yaml  # Custom PrometheusRule CRs
│   │
│   └── logging/
│       ├── loki-values.yaml      # Loki Helm values (Simple Scalable + S3)
│       └── promtail-values.yaml  # Promtail Helm values
```

---

## App of Apps Pattern

The entire cluster is bootstrapped from a single entry point:

```
Terraform applies bootstrap/root-app.yaml
              ↓
ArgoCD reads apps/ folder recursively
              ↓
┌─────────────────────────────────────┐
│         Applications Created        │
│                                     │
│  argocd-infra    → argocd/          │
│  web-app-prod    → k8s/overlays/prod│
│  kube-prometheus → k8s/monitoring/  │
│  alerting-rules  → k8s/monitoring/  │
│  loki            → k8s/logging/     │
│  promtail        → k8s/logging/     │
└─────────────────────────────────────┘
              ↓
Each application syncs its own resources
              ↓
Cluster matches Git state completely
```

**Adding a new application is as simple as adding a YAML file to `apps/`** — ArgoCD picks it up automatically on the next sync cycle.

---

## Applications

### ArgoCD Self-Management (`apps/argocd/app.yaml`)
ArgoCD manages itself — its own ingress and ConfigMap patches are defined in `argocd/` and synced back by ArgoCD. This ensures ArgoCD configuration survives cluster rebuilds.

---

### Web Application (`apps/web-app/app.yaml`)

| Field | Value |
|---|---|
| Name | `web-app-prod` |
| Namespace | `prod-app` |
| Source | `k8s/overlays/prod` |
| Tool | Kustomize |
| Sync | Automated with prune + selfHeal |

---

### kube-prometheus-stack (`apps/monitoring/app.yaml`)

| Field | Value |
|---|---|
| Name | `kube-prometheus-stack` |
| Namespace | `monitoring` |
| Chart | `prometheus-community/kube-prometheus-stack` |
| Version | `65.1.1` (pinned) |
| Values | `k8s/monitoring/values.yaml` |

---

### Alerting Rules (`apps/monitoring/alerting-rules-app.yaml`)

| Field | Value |
|---|---|
| Name | `alerting-rules` |
| Namespace | `monitoring` |
| Source | `k8s/monitoring/rules/` |
| Tool | Raw manifests (PrometheusRule CRs) |

---

### Loki (`apps/logging/loki-app.yaml`)

| Field | Value |
|---|---|
| Name | `loki` |
| Namespace | `logging` |
| Chart | `grafana/loki` |
| Version | `6.18.0` (pinned) |
| Values | `k8s/logging/loki-values.yaml` |
| Storage | S3 bucket via IRSA |
| Mode | Simple Scalable (separate read/write paths) |

---

### Promtail (`apps/logging/promtail-app.yaml`)

| Field | Value |
|---|---|
| Name | `promtail` |
| Namespace | `logging` |
| Chart | `grafana/promtail` |
| Version | `6.16.6` (pinned) |
| Values | `k8s/logging/promtail-values.yaml` |
| Type | DaemonSet — one pod per node |

---

## Blue/Green Deployment

The application uses a blue/green deployment strategy with ALB weighted traffic routing.

### How It Works

```
Developer pushes code to app-repo
              ↓
Jenkins builds Docker image → pushes to ECR
              ↓
Jenkins reads release.yaml → determines active color
              ↓
Jenkins updates TARGET color image tag in gitops repo
              ↓
Jenkins updates traffic patch in kustomization.yaml
              ↓
Jenkins pushes to gitops repo
              ↓
ArgoCD detects change → syncs to cluster
              ↓
New color deployed with new image
              ↓
ALB shifts 100% traffic to new color
              ↓
Old color scales down
```

### Traffic Management

Traffic is controlled via ALB weighted target groups in `traffic/` patches:

**Blue active (`traffic/traffic-blue-100.yaml`):**
```
web-svc-blue  → weight: 100
web-svc-green → weight: 0
```

**Green active (`traffic/traffic-green-100.yaml`):**
```
web-svc-blue  → weight: 0
web-svc-green → weight: 100
```

### Color Tracking (`release.yaml`)

```yaml
activeColor: green    # updated by Jenkins on each deployment
```

Jenkins reads this file to determine which color is currently live and which is the deployment target.

### Image Patches

Each color has its own image patch file updated by Jenkins:

```yaml
# patch-green-image.yaml
spec:
  template:
    spec:
      containers:
        - name: web-app
          image: 608827180555.dkr.ecr.ap-south-1.amazonaws.com/web-app:<GIT_COMMIT_SHA>
```

### Reliability Features

| Feature | Configuration |
|---|---|
| HPA | Min 2, Max 5 replicas per color |
| PodDisruptionBudget | Min 1 available during disruptions |
| Readiness Probe | HTTP GET `/` port 8080 |
| Liveness Probe | HTTP GET `/` port 8080 |
| Resource Requests | CPU: 200m, Memory: 128Mi |
| Resource Limits | CPU: 500m, Memory: 256Mi |
| Security Context | `runAsNonRoot: true`, read-only root filesystem |

---

## Observability Stack

### Architecture

```
Your Pods (metrics on :8080/metrics)
Node Exporter (host metrics on :9100)
kube-state-metrics (k8s object metrics)
              ↓ scrape every 15s
         Prometheus
         (stores in EBS gp3, 7d retention)
              ↓ query
           Grafana
         (dashboards + alerts UI)
              ↓ fire alerts
         Alertmanager
         (routes to Slack)
```

### Prometheus Configuration

| Setting | Value |
|---|---|
| Retention | 7 days |
| Storage | 10Gi EBS gp3 |
| CPU Request | 200m |
| Memory Request | 400Mi |

### Grafana Configuration

| Setting | Value |
|---|---|
| Admin Password | Configured via values |
| Ingress | ALB internet-facing |
| Persistence | Disabled (stateless) |
| Datasources | Prometheus (default), Loki |

### Disabled Components (EKS Managed Control Plane)

These components are disabled because EKS manages the control plane — they are not accessible:

```yaml
kubeEtcd:             enabled: false
kubeControllerManager: enabled: false
kubeScheduler:        enabled: false
```

---

## Alerting

### Custom PrometheusRule (`k8s/monitoring/rules/alerting-rules.yaml`)

9 custom alerting rules across 4 categories:

#### Infrastructure Alerts

| Alert | Condition | Severity | For |
|---|---|---|---|
| `NodeMemoryHigh` | Node memory > 85% | warning | 5m |
| `NodeCPUHigh` | Node CPU > 80% | warning | 5m |
| `PodCrashLooping` | Pod restarts > 3 in 15m | critical | 5m |
| `PodNotReady` | Pod not ready | warning | 5m |

#### Deployment Alerts

| Alert | Condition | Severity | For |
|---|---|---|---|
| `HPAAtMaxReplicas` | HPA at maximum replicas | warning | 10m |

#### Application Alerts

| Alert | Condition | Severity | For |
|---|---|---|---|
| `HighHTTPErrorRate` | 5xx rate > 5% | critical | 5m |
| `HighResponseLatency` | p99 latency > 2s | warning | 5m |

#### Kubernetes Alerts

| Alert | Condition | Severity | For |
|---|---|---|---|
| `PVCAlmostFull` | PVC usage > 85% | warning | 5m |
| `NodeDiskPressure` | Root disk > 85% | critical | 5m |

### Alertmanager Routing

```
Alert fires
    ↓
Group by: alertname + namespace
    ↓
group_wait: 30s (collect related alerts)
    ↓
Send to #devops-alerts Slack channel
    ↓
critical → repeat every 1h
warning  → repeat every 6h
    ↓
Send ✅ RESOLVED when condition clears
```

### Slack Secret Management

The Slack webhook URL is stored as a Kubernetes Secret — never committed to Git:

```bash
kubectl create secret generic alertmanager-slack-webhook \
  --from-literal=webhook-url='https://hooks.slack.com/services/...' \
  --namespace monitoring
```

Alertmanager reads it via file mount — not environment variable — ensuring it never appears in pod specs or logs.

---

## Logging Stack

### Architecture

```
All pod stdout/stderr
       ↓
Promtail DaemonSet
(reads /var/log/pods/ on each node)
       ↓ ship logs with labels
Loki Gateway
       ↓
┌──────────────┐    ┌──────────────┐
│  Loki Write  │    │  Loki Read   │
│  (2 replicas)│    │  (2 replicas)│
└──────┬───────┘    └──────┬───────┘
       │                   │
       ▼                   ▼
    S3 Bucket (long-term storage)
       ↑
    Loki Backend (compaction)
```

### Loki Configuration

| Setting | Value |
|---|---|
| Mode | Simple Scalable |
| Storage | S3 (`{cluster}-loki-logs`) |
| Auth | IRSA (no credentials in config) |
| Schema | v13 / tsdb |
| Log Retention | 30 days (S3 lifecycle rule) |
| Write Replicas | 2 |
| Read Replicas | 2 |

### Log Labels Applied by Promtail

| Label | Source |
|---|---|
| `namespace` | Pod namespace |
| `pod` | Pod name |
| `container` | Container name |
| `app` | Pod `app` label |
| `node` | Node name |
| `job` | `namespace/pod` |

### Querying Logs in Grafana

Go to **Grafana → Explore → Select Loki datasource**

Example LogQL queries:

```logql
# All logs from prod-app namespace
{namespace="prod-app"}

# Error logs from web-app
{namespace="prod-app", app="web-app"} |= "ERROR"

# ArgoCD sync logs
{namespace="argocd"} |= "sync"

# Logs from specific pod
{pod="web-app-blue-xxxxxxxxx"}
```

---

## How to Add a New Application

1. Create an ArgoCD Application CR in `apps/<your-app>/app.yaml`
2. Add your Kubernetes manifests to `k8s/<your-app>/`
3. Push to `main` branch
4. ArgoCD `root-app` detects the new file and creates the Application automatically
5. New Application syncs your manifests to the cluster

No manual `kubectl apply` required.

---

## Workflow

### Normal Deployment Flow

```
Developer pushes to app-repo (eks-devops-app)
              ↓
Jenkins pipeline triggered
              ↓
Docker image built → pushed to ECR
              ↓
Jenkins updates this repo:
  - k8s/overlays/prod/patch-{color}-image.yaml  (new image tag)
  - k8s/overlays/prod/release.yaml              (active color)
  - k8s/overlays/prod/kustomization.yaml        (traffic patch)
              ↓
Git push to main
              ↓
ArgoCD detects change (polls every 3 minutes)
              ↓
ArgoCD syncs web-app-prod application
              ↓
New color deployed, traffic switched
```

### Rollback Flow

```
Jenkins pipeline triggered with ROLLBACK=true
              ↓
git revert HEAD in this repo
              ↓
ArgoCD detects revert commit
              ↓
Previous image tag restored
              ↓
Cluster rolled back automatically
```

### Infrastructure Change Flow

```
Change Terraform in aws-eks-devops-platform repo
              ↓
terraform apply
              ↓
If ArgoCD config changed → update this repo
              ↓
ArgoCD syncs cluster
```

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [aws-eks-devops-platform](https://github.com/gitsohel1030/aws-eks-devops-platform) | Terraform infrastructure — EKS, VPC, IAM, ArgoCD install |
| [eks-devops-app](https://github.com/gitsohel1030/eks-devops-app) | Application source — Dockerfile, Jenkinsfile, app code |
