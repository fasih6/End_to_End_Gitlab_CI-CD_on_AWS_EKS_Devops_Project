# 📊 Monitoring Stack

Full observability setup for the `devops1-cluster` EKS deployment using the **kube-prometheus-stack**, **Loki**, **Promtail**, and **Alertmanager**.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Components](#components)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
  - [Prometheus](#prometheus)
  - [Grafana](#grafana)
  - [Loki & Promtail](#loki--promtail)
  - [Alertmanager](#alertmanager)
  - [Alert Rules](#alert-rules)
- [Application Metrics](#application-metrics)
  - [Backend Service Monitor](#backend-service-monitor)
  - [MySQL Exporter](#mysql-exporter)
- [Grafana Dashboards](#grafana-dashboards)
- [Accessing the Stack](#accessing-the-stack)
- [Directory Structure](#directory-structure)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    prod namespace                       │
│                                                         │
│  ┌──────────┐   /metrics   ┌──────────────────────┐     │
│  │ Backend  │◄────────────►│  ServiceMonitor      │     │
│  │ (Node.js)│              │  (backend-monitor)   │     │
│  └──────────┘              └──────────┬───────────┘     │
│                                        │                │
│  ┌──────────┐   port 9104  ┌──────────▼───────────┐     │
│  │  MySQL   │◄────────────►│  ServiceMonitor      │     │
│  │ Exporter │              │  (mysql-exporter)    │     │
│  └──────────┘              └──────────┬───────────┘     │
└───────────────────────────────────────┼─────────────────┘
                                         │ scrape
┌───────────────────────────────────────▼─────────────────┐
│                 monitoring namespace                    │
│                                                         │
│  ┌──────────────┐    ┌──────────┐    ┌───────────────┐  │
│  │  Prometheus  │───►│ Grafana  │    │ Alertmanager  │  │
│  │  (50Gi EBS)  │    │(10Gi EBS)│    │  (10Gi EBS)   │  │
│  └──────────────┘    └──────────┘    └───────────────┘  │
│                                                         │
│  ┌──────────────┐    ┌──────────────────────────────┐   │
│  │     Loki     │◄───│  Promtail (DaemonSet)        │   │
│  │  (50Gi EBS)  │    │  (runs on every node)        │   │
│  └──────────────┘    └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Components

| Component | Purpose | Mode |
|-----------|---------|------|
| **Prometheus** | Metrics collection & storage | 15-day retention, EBS-backed |
| **Grafana** | Visualization & dashboards | TLS via cert-manager |
| **Alertmanager** | Alert routing (Slack + Email) | EBS-backed |
| **Loki** | Log aggregation | SingleBinary, filesystem storage |
| **Promtail** | Log shipping (pod logs → Loki) | DaemonSet on all nodes |
| **MySQL Exporter** | MySQL metrics → Prometheus | Sidecar-style deployment |
| **Node Exporter** | Host-level metrics | Managed by kube-prometheus-stack |
| **kube-state-metrics** | Kubernetes object metrics | Managed by kube-prometheus-stack |

---

## Prerequisites

- EKS cluster (`devops1-cluster`) running in `us-east-1`
- `kubectl` configured with cluster access
- `helm` v3+ installed
- `ebs-sc` StorageClass deployed (see `k8s-prod/storageclass.yaml`)
- NGINX Ingress Controller deployed
- cert-manager deployed with `letsencrypt-prod` ClusterIssuer

---

## Installation

### 1. Create the monitoring namespace

```bash
kubectl create namespace monitoring
```

### 2. Add Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 3. Install kube-prometheus-stack

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values observability/prometheus-values.yaml \
  --wait
```

### 4. Install Loki

```bash
helm install loki grafana/loki \
  --namespace monitoring \
  --values observability/loki-values.yaml \
  --wait
```

### 5. Install Promtail

```bash
helm install promtail grafana/promtail \
  --namespace monitoring \
  --values observability/promtail-values.yaml \
  --wait
```

### 6. Apply observability manifests

```bash
# Alert rules
kubectl apply -f observability/alert-rules.yaml

# Alertmanager config (update webhook URLs first!)
kubectl apply -f observability/alertmanager-config.yaml

# Application monitors (from prod namespace)
kubectl apply -f observability/service-monitor.yaml

# MySQL exporter
kubectl apply -f observability/mysql-exporter.yaml
```

### 7. Verify all pods are running

```bash
kubectl get pods -n monitoring
kubectl get pods -n prod -l app=mysql-exporter
```

---

## Configuration

### Prometheus

Configured via `observability/prometheus-values.yaml`.

**Key settings:**

| Setting | Value |
|---------|-------|
| Retention | 15 days |
| Storage | 50Gi EBS (`ebs-sc`) |
| CPU request/limit | 250m / 500m |
| Memory request/limit | 512Mi / 1Gi |
| Namespace monitoring | All namespaces (`serviceMonitorSelectorNilUsesHelmValues: false`) |

Prometheus auto-discovers pods with these annotations on their PodSpec:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "5000"
  prometheus.io/path: "/metrics"
```

### Grafana

Accessible at `https://grafana.fasih.site` (TLS via Let's Encrypt).
**Note**: 
```
#if "kubectl get certificate -n monitoring" grafana-tls is false, then do this 
# Delete TLS secret (forces regeneration)
kubectl delete secret grafana-tls -n monitoring --ignore-not-found
# Delete old CertificateRequest
kubectl delete certificaterequest -n monitoring --all
```

| Setting | Value |
|---------|-------|
| Default password | `admin123` — **change after first login** |
| Storage | 10Gi EBS (`ebs-sc`) |
| TLS secret | `grafana-tls` |

Pre-configured data sources: **Prometheus** and **Loki** (add Loki manually in UI or via additional values).

### Loki & Promtail

Loki runs in **SingleBinary** mode (single replica, filesystem storage on 50Gi EBS). Suitable for dev/staging; for production scale consider switching to distributed mode with S3 backend.

Promtail runs as a **DaemonSet** on every node, shipping container logs to:

```
http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push
```

### Alertmanager

Before applying `alertmanager-config.yaml`, update the placeholder values:

```yaml
# observability/alertmanager-config.yaml
receivers:
  - name: 'critical'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'   # ← Replace
        channel: '#alerts-critical'
    email_configs:
      - to: 'your-email@example.com'        # ← Replace
        auth_password: 'your-app-password'  # ← Replace

  - name: 'warning'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'   # ← Replace
        channel: '#alerts-warning'
```

Alertmanager routes:

```
All alerts
├── severity: critical  →  Slack #alerts-critical + Email
└── severity: warning   →  Slack #alerts-warning
```

Inhibition rule: `critical` alerts suppress `warning` alerts for the same `alertname/cluster/service`.

### Alert Rules

Defined in `observability/alert-rules.yaml` as a `PrometheusRule` resource (namespace: `monitoring`, label: `release: kube-prometheus-stack`).

| Alert | Condition | Severity | For |
|-------|-----------|----------|-----|
| `PodDown` | Pod not in Running phase | critical | 5m |
| `HighCPUUsage` | CPU > 80% of limit | warning | 5m |
| `HighMemoryUsage` | Memory > 80% of limit | warning | 5m |
| `HighErrorRate` | HTTP 5xx rate > 5% | critical | 5m |
| `MySQLDown` | `mysql_up == 0` | critical | 1m |
| `PersistentVolumeAlmostFull` | PV usage > 80% | warning | 5m |
| `PodRestartingTooOften` | Restart rate > 0 in 15m | warning | 5m |

---

## Application Metrics

### Backend Service Monitor

The Node.js backend exposes Prometheus metrics at `/metrics` on port `5000` using `prom-client`.

**Metrics exposed:**

- Default Node.js metrics (CPU, memory, event loop lag, GC)
- `http_request_duration_seconds` — histogram tracking request latency by `method`, `route`, and `status_code`

The `ServiceMonitor` in `observability/service-monitor.yaml` scrapes `backend-svc` every 30 seconds:

```yaml
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

### MySQL Exporter

`mysql-exporter` uses the official `prom/mysqld-exporter:v0.15.1` image and connects to MySQL at `mysql:3306`.

It is deployed in the `prod` namespace alongside a `ServiceMonitor` that scrapes port `9104` every 30 seconds.

Credentials are sourced from the existing `mysql-secret` Kubernetes Secret:

```yaml
env:
  - name: MYSQLD_EXPORTER_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: MYSQL_ROOT_PASSWORD
```

---

## Grafana Dashboards

The following dashboards are pre-imported via Helm values:

| Dashboard | Grafana ID | Description |
|-----------|------------|-------------|
| Kubernetes Cluster | `7249` | Cluster-wide resource overview |
| Node Exporter Full | `1860` | Per-node CPU, memory, disk, network |
| Kubernetes Pods | `6417` | Pod-level resource metrics |
| NGINX Ingress | `9614` | Ingress request rates and latency |

To add a Loki log exploration panel, configure a **Loki** data source in Grafana pointing to:

```
http://loki-gateway.monitoring.svc.cluster.local
```

---

## Accessing the Stack

| Service | URL | Notes |
|---------|-----|-------|
| Grafana | `https://grafana.fasih.site` | TLS, default: `admin` / `admin123` |
| Prometheus | Port-forward only | `kubectl port-forward svc/kube-prometheus-stack-prometheus 9090 -n monitoring` |
| Alertmanager | Port-forward only | `kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093 -n monitoring` |
| Loki | Internal only | `http://loki-gateway.monitoring.svc.cluster.local` |

> **Security note:** Prometheus and Alertmanager UIs are not exposed via Ingress. Use `kubectl port-forward` for local access or add authenticated Ingress rules if needed.

---

## Directory Structure

```
observability/
├── prometheus-values.yaml      # kube-prometheus-stack Helm values
├── loki-values.yaml            # Loki Helm values (SingleBinary mode)
├── promtail-values.yaml        # Promtail Helm values (DaemonSet)
├── alertmanager-config.yaml    # Alertmanager routing & receivers
├── alert-rules.yaml            # PrometheusRule — application alert rules
├── service-monitor.yaml        # ServiceMonitors for backend & mysql-exporter
└── mysql-exporter.yaml         # MySQL Exporter Deployment, Service, ServiceMonitor
```
