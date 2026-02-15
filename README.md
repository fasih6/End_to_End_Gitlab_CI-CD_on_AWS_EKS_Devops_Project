# 🚀 Full-Stack App on AWS EKS

A production-grade DevOps project deploying a full-stack Node.js + React user management application on **AWS EKS** with a complete CI/CD pipeline, infrastructure as code, container security scanning, and a full observability stack.

---

## 📸 Project Snapshot

| Layer | Technology |
|-------|------------|
| **Application** | Node.js (Express) + React |
| **Database** | MySQL 8 (StatefulSet) |
| **Containerization** | Docker |
| **Registry** | Docker Hub |
| **CI/CD** | GitLab CI/CD (self-hosted runner) |
| **Security** | Gitleaks, Trivy, SonarQube |
| **Infrastructure** | Terraform → AWS EKS (Kubernetes 1.33) |
| **Ingress & TLS** | NGINX Ingress + cert-manager + Let's Encrypt |
| **Observability** | Prometheus + Grafana + Loki + Promtail + Alertmanager |
| **Notifications** | Slack + Email (via Alertmanager) |

---

## 📐 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Developer Workstation                        │
│                                                                      │
│   git push → GitLab → CI/CD Pipeline                                │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      GitLab CI/CD Pipeline                           │
│                                                                      │
│  [install tools] → [security scan] → [test] → [code quality]        │
│       │                  │              │            │               │
│    Trivy              Gitleaks       Unit Tests   SonarQube          │
│    Gitleaks           Trivy FS                                       │
│                                                                      │
│  → [docker build + scan + push] → [update k8s manifests]            │
│         │                                  │                         │
│      Trivy image                    git commit + push                │
│      Docker Hub                     [IMAGE_TAG in yaml]              │
│                                                                      │
│  → [deploy to EKS]                                                   │
│         │                                                            │
│      kubectl apply                                                   │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     AWS (us-east-1)                                  │
│                                                                      │
│  ┌────────────────── EKS Cluster: devops1-cluster ──────────────┐   │
│  │                                                               │   │
│  │   prod namespace                                              │   │
│  │   ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │   │
│  │   │   backend    │  │   frontend   │  │  MySQL          │  │   │
│  │   │  (3 pods)    │  │  (3 pods)    │  │  StatefulSet    │  │   │
│  │   │  Node.js     │  │  React/Nginx │  │  (1 pod)        │  │   │
│  │   └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │   │
│  │          │                 │                    │            │   │
│  │   monitoring namespace     │               EBS 5Gi           │   │
│  │   ┌─────────────────────────────────────────────────────┐   │   │
│  │   │  Prometheus  │  Grafana  │  Loki  │  Alertmanager  │   │   │
│  │   └─────────────────────────────────────────────────────┘   │   │
│  │                                                               │   │
│  │   Ingress (NGINX) + cert-manager (Let's Encrypt TLS)         │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  VPC (10.0.0.0/16)  │  2 Public Subnets  │  EBS CSI Driver          │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    https://fasih.site
                    https://grafana.fasih.site
```

---

## 📁 Repository Structure

```
.
├── api/                          # Node.js Express backend
│   ├── app.js                    # Entry point, Prometheus metrics setup
│   ├── routes/
│   │   ├── ...
│   │   └── ...
│   ├── models/
│   │   └── ...               # MySQL connection pool
│   ├── Dockerfile
│   └── package.json
│
├── client/                       # React frontend
│   ├── src/
│   ├── Dockerfile
│   └── package.json
│
├── k8s-prod/                     # Kubernetes manifests (prod namespace)
│   ├── storageclass.yaml         # ebs-sc StorageClass
│   ├── cert-issuer.yaml          # Let's Encrypt ClusterIssuer
│   ├── mysql.yaml                # StatefulSet + Service + ConfigMaps
│   ├── backend.yaml              # Deployment + Service
│   ├── frontend.yaml             # Deployment + Service
│   └── ingress.yaml              # NGINX Ingress with TLS
│
├── observability/                # Monitoring stack configs
│   ├── prometheus-values.yaml    # kube-prometheus-stack Helm values
│   ├── loki-values.yaml          # Loki Helm values
│   ├── promtail-values.yaml      # Promtail DaemonSet values
│   ├── alertmanager-config.yaml  # Slack + Email alert routing
│   ├── alert-rules.yaml          # PrometheusRule definitions
│   ├── service-monitor.yaml      # ServiceMonitors for backend + MySQL
│   └── mysql-exporter.yaml       # MySQL Prometheus exporter
│
├── terraform/                    # Infrastructure as Code
│   ├── main.tf                   # VPC, EKS cluster, node group, IAM
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
│
├── .gitlab-ci.yml                # GitLab CI/CD pipeline definition
│
├── README.md                     # ← You are here
├── KUBERNETES.md                 # Kubernetes manifests deep-dive
├── MONITORING.md                 # Observability stack deep-dive
└── RBAC.md                       # GitLab CI/CD RBAC setup guide
```

---

## ⚙️ CI/CD Pipeline

The GitLab pipeline runs on a **self-hosted runner** and consists of 7 sequential stages:

```
install_tools → security_scan → test → code_quality
    → docker_build_scan_push → update_manifests → deploy_to_eks
```

| Stage | Job | What it does |
|-------|-----|-------------|
| `install_tools` | `install-tools-job` | Installs Trivy and Gitleaks on the runner |
| `security_scan` | `gitleaks-scan-job` | Scans git history for leaked secrets |
| `security_scan` | `trivy-fs-scan-job` | Scans filesystem/dependencies for CVEs |
| `test` | `unit-test-job` | Runs application unit tests |
| `code_quality` | `sonarqube-analysis-job` | Static code analysis (runs on `main` + MR) |
| `docker_build_scan_push` | `build-scan-push-job` | Builds images → Trivy scan → push to Docker Hub (blocks on CRITICAL CVEs) |
| `update_manifests` | `update-k8s-manifests-job` | Updates image tags in `k8s-prod/` YAML files and commits back to repo |
| `deploy_to_eks` | `deploy-to-eks-job` | `kubectl apply` to the EKS cluster via KUBECONFIG |

**Image tagging strategy:** `{branch-slug}-{short-SHA}` (e.g., `main-2b69588d`)

**Security gate:** The pipeline hard-fails at `build-scan-push-job` if any CRITICAL vulnerabilities are found in the container images. HIGH vulnerabilities are reported but do not block the pipeline.

---

## 🏗️ Infrastructure (Terraform)

Terraform provisions the full AWS infrastructure from scratch:

```
VPC (10.0.0.0/16)
  ├── 2 Public Subnets (us-east-1a, us-east-1b)
  ├── Internet Gateway + Route Table
  ├── Cluster Security Group
  └── Node Security Group

EKS Cluster: devops1-cluster (Kubernetes 1.33)
  ├── Node Group: 3 × c7i-flex.large
  ├── OIDC Provider (for IRSA)
  └── EBS CSI Driver Addon (v1.50.1) with dedicated IAM role
```

**Key Terraform design decisions:**

- OIDC provider enables **IRSA** (IAM Roles for Service Accounts) — the EBS CSI Driver uses a scoped IAM role rather than node-level credentials
- `WaitForFirstConsumer` volume binding prevents cross-AZ PVC/pod mismatches
- Node group uses `c7i-flex.large` instances — a flexible compute-optimized type offering good price/performance for mixed workloads
- All subnets tagged with `kubernetes.io/role/elb: "1"` for AWS Load Balancer Controller compatibility

To provision:

```bash
cd terraform/
terraform init
terraform plan
terraform apply
```

To update kubeconfig after provisioning:

```bash
aws eks update-kubeconfig --name devops1-cluster --region us-east-1
```
---

## ☸️ Kubernetes Stack

All application workloads run in the `prod` namespace. See [KUBERNETES.md](k8s-prod/README.md) for the full deep-dive.

| Workload | Kind | Replicas | Image |
|----------|------|----------|-------|
| `backend` | Deployment | 3 | `fasih6/backend:<tag>` |
| `frontend` | Deployment | 3 | `fasih6/frontend:<tag>` |
| `mysql` | StatefulSet | 1 | `mysql:8` |
| `mysql-exporter` | Deployment | 1 | `prom/mysqld-exporter:v0.15.1` |

**Zero-downtime deployments** via `RollingUpdate` with `maxUnavailable: 0`.

**Public endpoints:**

| URL | Service |
|-----|---------|
| `https://fasih.site` | React frontend |
| `https://fasih.site/api` | Node.js REST API |
| `https://grafana.fasih.site` | Grafana dashboards |

---

## 📊 Observability Stack

Full observability across metrics, logs, and alerts. See [MONITORING.md](monitoring/MONITORING.md) for the full deep-dive.

**Metrics pipeline:**

```
App pods (/metrics)  ──►  Prometheus  ──►  Grafana dashboards
MySQL Exporter        ──►  Prometheus  ──►  Alertmanager
Node Exporter         ──►  Prometheus       │
kube-state-metrics    ──►  Prometheus       ▼
                                      Slack / Email
```

**Log pipeline:**

```
All pod logs  ──►  Promtail (DaemonSet)  ──►  Loki  ──►  Grafana (Explore)
```

**Pre-built alerts:**

| Alert | Trigger | Severity |
|-------|---------|----------|
| `PodDown` | Pod not Running for 5m | critical |
| `HighErrorRate` | HTTP 5xx > 5% for 5m | critical |
| `MySQLDown` | mysql_up == 0 for 1m | critical |
| `HighCPUUsage` | CPU > 80% for 5m | warning |
| `HighMemoryUsage` | Memory > 80% for 5m | warning |
| `PersistentVolumeAlmostFull` | PV > 80% for 5m | warning |
| `PodRestartingTooOften` | Restart rate > 0 in 15m | warning |

---

## 🔐 Security

### Pipeline security

- **Gitleaks** scans the full git history for committed secrets on every pipeline run
- **Trivy filesystem scan** checks dependencies and OS packages for known CVEs
- **Trivy image scan** runs against built Docker images — CRITICAL CVEs block the push
- **SonarQube** performs static code analysis on `main` branch and merge requests

### Kubernetes security

- Dedicated `gitlab-cicd-sa` ServiceAccount with least-privilege RBAC (no `cluster-admin`)
- Namespace-scoped Role for workload management in `prod`
- Cluster-scoped ClusterRole grants read-only access to nodes, storageclasses, and PVs only
- Database passwords stored as Kubernetes Secrets, never in Git
- TLS enforced end-to-end via cert-manager + Let's Encrypt

See [RBAC.md](rbac/RBAC.md) for the complete RBAC setup guide.

### GitLab variable hygiene

All sensitive values are stored as **masked + protected** GitLab CI/CD variables:

```
KUBECONFIG_DATA     ← base64-encoded kubeconfig (masked + protected)
DOCKER_PASSWORD     ← Docker Hub access token (masked + protected)
DOCKER_USERNAME     ← Docker Hub access username
GITLAB_PUSH_TOKEN   ← GitLab PAT for manifest commits (masked + protected)
SONAR_TOKEN         ← SonarQube token (masked + protected)
SONAR_URL           ← SonarQube URL 
```

---

## 🚀 Getting Started

### Prerequisites

| Tool | Version |
|------|---------|
| Terraform | ≥ 1.0 |
| AWS CLI | ≥ 2.0 |
| kubectl | ≥ 1.28 |
| helm | ≥ 3.0 |
| Docker | ≥ 24 |

### 1 — Provision infrastructure

```bash
cd terraform/
terraform init
terraform apply

# Configure kubectl
aws eks update-kubeconfig --name devops1-cluster --region us-east-1
```

### 2 — Install cluster dependencies

```bash
# NGINX Ingress Controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager -n cert-manager \
  --create-namespace --set installCRDs=true
```

### 3 — Set up RBAC for GitLab CI/CD

Follow the full guide in [RBAC.md](rbac/RBAC.md). At minimum:

```bash
kubectl create namespace prod
kubectl apply -f -  # (ServiceAccount, Role, RoleBinding, ClusterRole, ClusterRoleBinding)

kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD='<strong-password>' \
  -n prod
```

### 4 — Apply Kubernetes manifests

```bash
kubectl apply -f k8s-prod/storageclass.yaml
kubectl apply -f k8s-prod/cert-issuer.yaml
kubectl apply -f k8s-prod/ -n prod
```

### 5 — Install the observability stack

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --values observability/prometheus-values.yaml

helm install loki grafana/loki \
  -n monitoring --values observability/loki-values.yaml

helm install promtail grafana/promtail \
  -n monitoring --values observability/promtail-values.yaml

kubectl apply -f observability/alert-rules.yaml
kubectl apply -f observability/alertmanager-config.yaml
kubectl apply -f observability/mysql-exporter.yaml
kubectl apply -f observability/service-monitor.yaml
```

### 6 — Configure GitLab CI/CD variables

Add these variables in **GitLab → Settings → CI/CD → Variables**:

```
KUBECONFIG_DATA      (base64-encoded kubeconfig)
DOCKER_USERNAME
DOCKER_PASSWORD
GITLAB_PUSH_TOKEN
SONAR_HOST_URL
SONAR_TOKEN
```

### 7 — Push to `main` and watch the pipeline

```bash
git add .
git commit -m "initial deployment"
git push origin main
```

Navigate to **GitLab → CI/CD → Pipelines** to monitor progress.

---

## 📖 Documentation

| File | Contents |
|------|----------|
| [KUBERNETES.md](k8s-prod/README.md) | All Kubernetes manifests explained — resources, routing, storage, rolling updates, troubleshooting |
| [MONITORING.md](monitoring/MONITORING.md) | Full observability stack — Prometheus, Grafana, Loki, Alertmanager, alert rules, dashboards |
| [RBAC.md](rbac/RBAC.md) | GitLab CI/CD RBAC setup — ServiceAccount, Role, kubeconfig generation, token rotation |

---

## 🔮 Potential Improvements

- Add Kubernetes **liveness and readiness probes** to backend and frontend deployments
- Restrict `/metrics` endpoint to internal cluster access only via NetworkPolicy
- Add **HorizontalPodAutoscaler** (HPA) for backend and frontend based on CPU/RPS
- Switch MySQL to a managed **Amazon RDS** instance for automated backups and failover
- Move Loki to **S3 backend** for production-scale log retention
- Add **Velero** for Kubernetes resource and PV backup and restore
- Replace long-lived ServiceAccount tokens with **EKS Pod Identity** or IRSA for GitLab runner
- Enforce **NetworkPolicies** to restrict pod-to-pod traffic to only necessary paths
- Add **staging environment** with separate namespace and pipeline branch rules

