# ☸️ Kubernetes Manifests

Production Kubernetes manifests for the `devops1` application stack deployed to the `prod` namespace on EKS `devops1-cluster`.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Namespace & Prerequisites](#namespace--prerequisites)
- [Manifests Reference](#manifests-reference)
  - [storageclass.yaml](#storageclassyaml)
  - [mysql.yaml](#mysqlyaml)
  - [backend.yaml](#backendyaml)
  - [frontend.yaml](#frontendyaml)
  - [ingress.yaml](#ingressyaml)
  - [cert-issuer.yaml](#cert-issueryaml)
- [Deployment Order](#deployment-order)
- [Rolling Update Strategy](#rolling-update-strategy)
- [Resource Requests & Limits](#resource-requests--limits)
- [Networking & Traffic Flow](#networking--traffic-flow)
- [Persistent Storage](#persistent-storage)
- [Secrets & ConfigMaps](#secrets--configmaps)
- [Health & Observability](#health--observability)
- [Applying Manifests](#applying-manifests)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
Internet
    │
    │ HTTPS (443)
    ▼
┌──────────────────────────────────────────────┐
│         AWS Load Balancer (EKS)              │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│      NGINX Ingress Controller                │
│      (fasih.site / www.fasih.site)           │
│                                              │
│  /api, /metrics  ──────► backend-svc :5000   │
│  /              ───────► frontend-svc :80    │
└──────────┬──────────────────┬────────────────┘
           │                  │
           ▼                  ▼
┌─────────────────┐  ┌─────────────────────┐
│    backend      │  │      frontend       │
│  (3 replicas)   │  │    (3 replicas)     │
│  Node.js :5000  │  │    Nginx :80        │
└────────┬────────┘  └─────────────────────┘
         │
         │ mysql:3306
         ▼
┌─────────────────────────────────────────────┐
│         MySQL StatefulSet (1 replica)       │
│         PVC: 5Gi EBS (ebs-sc)               │
└─────────────────────────────────────────────┘
```

---

## Namespace & Prerequisites

All resources are deployed to the `prod` namespace.

The following must be installed in the cluster **before** applying these manifests:

| Dependency | Purpose |
|------------|---------|
| NGINX Ingress Controller | Routes external traffic to services |
| cert-manager | Provisions and renews TLS certificates |
| AWS EBS CSI Driver | Provisions EBS volumes for PVCs |
| `prod` namespace | Target namespace for all resources |

```bash
kubectl create namespace prod
```

---

## Manifests Reference

### `storageclass.yaml`

Defines the `ebs-sc` StorageClass backed by the AWS EBS CSI driver. This is a **cluster-scoped** resource used by MySQL and the observability stack for persistent storage.

```
StorageClass: ebs-sc
  provisioner:         ebs.csi.aws.com
  volumeBindingMode:   WaitForFirstConsumer   ← PVC bound only when a pod is scheduled
  reclaimPolicy:       Retain                 ← Volume kept after PVC deletion
```

> `WaitForFirstConsumer` prevents cross-AZ volume/pod mismatches on multi-AZ EKS clusters.

---

### `mysql.yaml`

Contains all MySQL-related resources in a single file:

```
mysql.yaml
├── Secret          mysql-secret        ← MYSQL_ROOT_PASSWORD
├── ConfigMap       mysql-config        ← MYSQL_DATABASE = crud_app
├── ConfigMap       mysql-initdb-config ← init.sql (schema + users table)
├── Service         mysql               ← ClusterIP :3306
└── StatefulSet     mysql               ← 1 replica, PVC 5Gi ebs-sc
```

**StatefulSet** is used instead of a Deployment because MySQL requires:
- A stable, predictable pod name (`mysql-0`)
- A stable persistent volume that follows the pod across reschedules

**Init SQL** (applied on first startup via `/docker-entrypoint-initdb.d`):

```sql
CREATE DATABASE IF NOT EXISTS crud_app;

CREATE TABLE IF NOT EXISTS users (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  name       VARCHAR(255) NOT NULL,
  email      VARCHAR(255) NOT NULL UNIQUE,
  password   VARCHAR(255) NOT NULL,
  role       ENUM('admin', 'viewer') NOT NULL DEFAULT 'viewer',
  is_active  TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

> ⚠️ **Security:** The `Secret` block in this file contains a placeholder password. Remove it before committing to Git and create the real secret manually:
> ```bash
> kubectl create secret generic mysql-secret \
>   --from-literal=MYSQL_ROOT_PASSWORD='<strong-password>' \
>   -n prod
> ```

---

### `backend.yaml`

```
backend.yaml
├── Deployment   backend     ← 3 replicas, RollingUpdate
└── Service      backend-svc ← ClusterIP :5000
```

**Deployment highlights:**

| Setting | Value |
|---------|-------|
| Image | `fasih6/backend:<IMAGE_TAG>` (updated by CI/CD) |
| Port | `5000` (named `http`) |
| Replicas | 3 |
| Strategy | RollingUpdate — `maxSurge: 1`, `maxUnavailable: 0` |
| DB host | `mysql` (resolves to MySQL Service via cluster DNS) |
| DB credentials | Sourced from `mysql-secret` and `mysql-config` |

**Environment variables:**

| Variable | Source |
|----------|--------|
| `DB_HOST` | Hardcoded `mysql` |
| `DB_USER` | Hardcoded `root` |
| `DB_PASSWORD` | `mysql-secret` → `MYSQL_ROOT_PASSWORD` |
| `DB_NAME` | `mysql-config` → `MYSQL_DATABASE` |

**Prometheus annotations** on the pod template enable automatic metrics discovery:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "5000"
  prometheus.io/path: "/metrics"
```

**Resource limits:**

| | CPU | Memory |
|-|-----|--------|
| Request | 250m | 256Mi |
| Limit | 500m | 512Mi |

---

### `frontend.yaml`

```
frontend.yaml
├── Deployment   frontend     ← 3 replicas, RollingUpdate
└── Service      frontend-svc ← ClusterIP :80
```

**Deployment highlights:**

| Setting | Value |
|---------|-------|
| Image | `fasih6/frontend:<IMAGE_TAG>` (updated by CI/CD) |
| Port | `80` (named `http`) |
| Replicas | 3 |
| Strategy | RollingUpdate — `maxSurge: 1`, `maxUnavailable: 0` |

**Resource limits:**

| | CPU | Memory |
|-|-----|--------|
| Request | 100m | 128Mi |
| Limit | 200m | 256Mi |

---

### `ingress.yaml`

Single `Ingress` resource routing traffic for both `fasih.site` and `www.fasih.site` with automatic TLS via cert-manager.

**Routing rules (identical for both hosts):**

| Path | Type | Backend Service | Port |
|------|------|-----------------|------|
| `/api` | Prefix | `backend-svc` | 5000 |
| `/metrics` | Prefix | `backend-svc` | 5000 |
| `/` | Prefix | `frontend-svc` | 80 |

**TLS:**

| Setting | Value |
|---------|-------|
| Annotation | `cert-manager.io/cluster-issuer: letsencrypt-prod` |
| SSL redirect | Enforced (`ssl-redirect: "true"`, `force-ssl-redirect: "true"`) |
| TLS Secret | `fasih6-co-tls` (auto-created by cert-manager) |
| Hosts | `fasih.site`, `www.fasih.site` |

> After applying, cert-manager will automatically request a Let's Encrypt certificate. Initial provisioning takes 1–2 minutes. Check status with:
> ```bash
> kubectl get certificate -n prod
> kubectl describe certificate fasih6-co-tls -n prod
> ```

---

### `cert-issuer.yaml`

Defines the `letsencrypt-prod` ClusterIssuer used by cert-manager to obtain TLS certificates via the ACME HTTP-01 challenge.

```
ClusterIssuer: letsencrypt-prod
  ACME server:  https://acme-v02.api.letsencrypt.org/directory
  Solver:       HTTP-01 via NGINX Ingress
  Key secret:   letsencrypt-prod
```

> This is a **cluster-scoped** resource — apply it once regardless of how many namespaces need TLS.

---

## Deployment Order

Resources must be applied in dependency order:

```
1. storageclass.yaml      ← cluster-scoped, no dependencies
2. cert-issuer.yaml       ← cluster-scoped, no dependencies
3. mysql.yaml             ← depends on ebs-sc StorageClass
4. backend.yaml           ← depends on mysql Service + mysql-secret
5. frontend.yaml          ← no backend dependency at deploy time
6. ingress.yaml           ← depends on backend-svc + frontend-svc + cert-issuer
```

Apply all at once (kubectl handles ordering for independent resources):

```bash
kubectl apply -f k8s-prod/ -n prod
```

Or in explicit order:

```bash
kubectl apply -f k8s-prod/storageclass.yaml
kubectl apply -f k8s-prod/cert-issuer.yaml
kubectl apply -f k8s-prod/mysql.yaml -n prod
kubectl apply -f k8s-prod/backend.yaml -n prod
kubectl apply -f k8s-prod/frontend.yaml -n prod
kubectl apply -f k8s-prod/ingress.yaml -n prod
```

---

## Rolling Update Strategy

Both `backend` and `frontend` Deployments use the same strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # temporarily run 4 pods during update (replicas + 1)
    maxUnavailable: 0  # never reduce below 3 running pods
```

This means:
- **Zero downtime** — the old replica is only terminated after the new one passes health checks
- **Maximum 4 pods** at peak during a rollout
- Kubernetes will wait for each new pod to become `Ready` before proceeding

To monitor a rollout:

```bash
kubectl rollout status deployment/backend -n prod
kubectl rollout status deployment/frontend -n prod
```

To roll back if needed:

```bash
kubectl rollout undo deployment/backend -n prod
kubectl rollout undo deployment/frontend -n prod
```

---

## Resource Requests & Limits

| Workload | CPU Request | CPU Limit | Memory Request | Memory Limit |
|----------|-------------|-----------|----------------|--------------|
| `backend` | 250m | 500m | 256Mi | 512Mi |
| `frontend` | 100m | 200m | 128Mi | 256Mi |
| `mysql` | _(not set)_ | _(not set)_ | _(not set)_ | _(not set)_ |

> Consider adding resource limits to the MySQL StatefulSet to prevent it from starving other workloads under load.

**Total per-node resources consumed** (worst case, all pods on one node):

| Resource | Backend × 3 | Frontend × 3 | Total Request |
|----------|-------------|--------------|---------------|
| CPU | 750m | 300m | **1050m** |
| Memory | 768Mi | 384Mi | **1152Mi** |

EKS nodes (`c7i-flex.large`) provide 2 vCPU and ~4Gi RAM, so this leaves healthy headroom for the monitoring stack and system pods.

---

## Networking & Traffic Flow

All inter-service communication uses **Kubernetes cluster DNS**:

```
backend → mysql:3306
           resolves to: mysql.prod.svc.cluster.local

ingress → backend-svc:5000
           resolves to: backend-svc.prod.svc.cluster.local

ingress → frontend-svc:80
           resolves to: frontend-svc.prod.svc.cluster.local
```

All Services use `ClusterIP` — they are not directly accessible from outside the cluster. External access is exclusively via the Ingress.

**Port naming** is applied consistently across all containers and Services (e.g., `name: http`, `name: mysql`, `name: metrics`). Named ports are required by `ServiceMonitor` resources for Prometheus scraping.

---

## Persistent Storage

| Workload | PVC Name | Size | StorageClass | Access Mode | Reclaim Policy |
|----------|----------|------|--------------|-------------|----------------|
| `mysql` | `mysql-persistent-storage-mysql-0` | 5Gi | `ebs-sc` | ReadWriteOnce | Retain |

The PVC is created automatically by the StatefulSet's `volumeClaimTemplates`. The `Retain` reclaim policy means the EBS volume is **not deleted** when the StatefulSet or PVC is deleted — data is preserved for manual recovery.

To inspect storage:

```bash
kubectl get pvc -n prod
kubectl get pv
```

---

## Secrets & ConfigMaps

| Resource | Kind | Namespace | Contents |
|----------|------|-----------|----------|
| `mysql-secret` | Secret | `prod` | `MYSQL_ROOT_PASSWORD` |
| `mysql-config` | ConfigMap | `prod` | `MYSQL_DATABASE=crud_app` |
| `mysql-initdb-config` | ConfigMap | `prod` | `init.sql` schema file |

**The `mysql-secret` must be created manually** before applying manifests:

```bash
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD='<strong-password>' \
  -n prod
```

Verify all config resources exist:

```bash
kubectl get secret mysql-secret -n prod
kubectl get configmap mysql-config -n prod
kubectl get configmap mysql-initdb-config -n prod
```

---

## Health & Observability

### Metrics endpoint

The backend exposes Prometheus metrics at `GET /metrics` on port `5000`. This path is publicly routable via the Ingress at `https://fasih.site/metrics`.

> Consider restricting `/metrics` to internal access only (e.g., via Ingress annotations or a NetworkPolicy) to avoid exposing runtime internals publicly.

### Health check

The backend exposes `GET /health` which returns:

```json
{ "status": "healthy" }
```

Consider adding this as a Kubernetes **liveness** and **readiness** probe:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Prometheus scraping

The backend pod template includes annotations that enable scraping via the `additionalScrapeConfigs` defined in `prometheus-values.yaml`:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "5000"
prometheus.io/path: "/metrics"
```

A `ServiceMonitor` in `observability/service-monitor.yaml` also scrapes `backend-svc` directly — both methods are in place for compatibility.

---

## Applying Manifests

### First-time deployment

```bash
# 1. Ensure the prod namespace exists
kubectl create namespace prod

# 2. Create the MySQL secret manually (do NOT rely on the one in mysql.yaml)
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD='<strong-password>' \
  -n prod

# 3. Apply all manifests
kubectl apply -f k8s-prod/ -n prod

# 4. Wait for MySQL to be ready before the backend connects
kubectl rollout status statefulset/mysql -n prod

# 5. Monitor backend startup (it retries DB connection automatically)
kubectl logs -f deployment/backend -n prod
```

### CI/CD deployments

The GitLab pipeline updates image tags in `backend.yaml` and `frontend.yaml` automatically via `sed` in the `update-k8s-manifests-job`, then applies the updated manifests in `deploy-to-eks-job`:

```bash
kubectl apply -f k8s-prod/ -n prod
```

---

## Verification

```bash
# All pods running
kubectl get pods -n prod

# All services exist
kubectl get svc -n prod

# Ingress has an external address
kubectl get ingress -n prod

# TLS certificate issued
kubectl get certificate -n prod

# PVC bound
kubectl get pvc -n prod

# Check backend logs
kubectl logs deployment/backend -n prod

# Check frontend logs
kubectl logs deployment/frontend -n prod

# Check MySQL logs
kubectl logs statefulset/mysql -n prod

# Quick connectivity test (from inside the cluster)
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://backend-svc.prod.svc.cluster.local:5000/health
```

Expected pod states:

```
NAME                        READY   STATUS    RESTARTS
backend-xxxx-xxxx           1/1     Running   0
backend-xxxx-yyyy           1/1     Running   0
backend-xxxx-zzzz           1/1     Running   0
frontend-xxxx-xxxx          1/1     Running   0
frontend-xxxx-yyyy          1/1     Running   0
frontend-xxxx-zzzz          1/1     Running   0
mysql-0                     1/1     Running   0
mysql-exporter-xxxx-xxxx    1/1     Running   0
```

---

## Troubleshooting

**Backend pods in `CrashLoopBackOff`**

The most common cause is the backend failing to connect to MySQL on startup.

```bash
kubectl logs deployment/backend -n prod

# The backend retries 30 times with 2s delay — check for:
# "🔴 MySQL query failed" or "❌ MySQL `users` table not available"

# Ensure MySQL pod is Running first
kubectl get pods -n prod -l app=mysql

# Verify the secret exists with the correct key
kubectl get secret mysql-secret -n prod -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
```

---

**MySQL pod stuck in `Pending`**

Usually a PVC binding issue.

```bash
kubectl describe pvc -n prod
kubectl get pv

# Check EBS CSI driver is running
kubectl get pods -n kube-system -l app=ebs-csi-controller

# Ensure ebs-sc StorageClass exists
kubectl get storageclass ebs-sc
```

---

**Ingress has no external IP / ADDRESS**

The NGINX Ingress Controller may not be installed or its LoadBalancer Service hasn't received an IP yet.

```bash
kubectl get svc -n ingress-nginx
# Wait for EXTERNAL-IP to be populated (can take 2–3 minutes on EKS)
```

---

**TLS certificate stuck in `False` / not issuing**
do this:
```bash
# Delete TLS secret (forces regeneration)
kubectl delete secret fasih6-co-tls -n prod --ignore-not-found
# Delete old CertificateRequest
kubectl delete certificaterequest -n prod --all
```
then wait 10-15 minutes and try again
```
kubectl get certificate -n prod
```
Common causes: DNS not pointing to the Ingress IP, or port 80 blocked by a security group (HTTP-01 challenge requires inbound port 80).

---

**`ImagePullBackOff` after CI/CD push**

```bash
kubectl describe pod <pod-name> -n prod
# Check Events section for "Failed to pull image"

# Verify the image tag exists in Docker Hub
# Check that DOCKER_USERNAME is correct in the manifest
grep "image:" k8s-prod/backend.yaml
grep "image:" k8s-prod/frontend.yaml
```

---

## Directory Structure

```
k8s-prod/
├── storageclass.yaml   # StorageClass: ebs-sc (cluster-scoped)
├── cert-issuer.yaml    # ClusterIssuer: letsencrypt-prod (cluster-scoped)
├── mysql.yaml          # Secret + ConfigMaps + Service + StatefulSet
├── backend.yaml        # Deployment + Service
├── frontend.yaml       # Deployment + Service
└── ingress.yaml        # Ingress (TLS, path routing)
```


