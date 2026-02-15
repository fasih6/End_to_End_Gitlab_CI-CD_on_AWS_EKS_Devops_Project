# 🔐 GitLab CI/CD RBAC Setup for EKS

Secure GitLab CI/CD access to the `devops1-cluster` EKS cluster using a dedicated Kubernetes **ServiceAccount** with least-privilege RBAC — no cluster-admin, no static AWS credentials.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Part 1: Create RBAC Resources](#part-1-create-rbac-resources)
- [Part 2: Generate ServiceAccount Token](#part-2-generate-serviceaccount-token)
- [Part 3: Generate Kubeconfig](#part-3-generate-kubeconfig)
- [Part 4: Test Kubeconfig Locally](#part-4-test-kubeconfig-locally)
- [Part 5: Upload to GitLab](#part-5-upload-to-gitlab)
- [Part 6: Create MySQL Secret](#part-6-create-mysql-secret)
- [Permissions Reference](#permissions-reference)
- [GitLab CI/CD Variables Reference](#gitlab-cicd-variables-reference)
- [Token Rotation](#token-rotation)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

---

## Architecture Overview

```
GitLab CI/CD Pipeline
        │
        │  KUBECONFIG_DATA (base64 JWT token)
        ▼
┌───────────────────────┐
│   kubectl / helm      │
│   (in pipeline job)   │
└──────────┬────────────┘
           │ authenticates as
           ▼
┌───────────────────────────────────────────────────────────┐
│                   EKS Cluster                             │
│                                                           │
│   ServiceAccount: gitlab-cicd-sa (namespace: prod)        │
│           │                                               │
│           ├── RoleBinding ──────► Role                    │
│           │   (prod ns only)      gitlab-cicd-prod-role   │
│           │                       (pods, deployments,     │
│           │                        services, secrets...)  │
│           │                                               │
│           └── ClusterRoleBinding ► ClusterRole            │
│               (cluster-wide)      gitlab-cicd-cluster-role│
│                                   (nodes, storageclasses, │
│                                    PVs, namespaces READ)  │
└───────────────────────────────────────────────────────────┘
```

**Scope summary:**

| Resource Scope | Permissions |
|----------------|-------------|
| `prod` namespace | Full CRUD on pods, deployments, services, secrets, configmaps, ingresses, PVCs |
| Cluster-wide | Read-only on nodes, storageclasses, PersistentVolumes, namespaces |

---

## Prerequisites

- EKS cluster (`devops1-cluster`) running in `us-east-1`
- `kubectl` configured with admin access to the cluster
- `aws` CLI configured with appropriate credentials
- Access to GitLab project **Settings → CI/CD → Variables**

---

## Part 1: Create RBAC Resources

### Step 1 — Create the `prod` namespace

```bash
kubectl create namespace prod
kubectl get namespace prod
```

### Step 2 — Create the ServiceAccount

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-cicd-sa
  namespace: prod
EOF

kubectl get serviceaccount gitlab-cicd-sa -n prod
```

### Step 3 — Create the namespace-scoped Role

Grants full access to workloads and networking resources **within `prod` only**.

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-cicd-prod-role
  namespace: prod
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
      - services
      - secrets
      - configmaps
      - persistentvolumeclaims
      - endpoints
      - events
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources:
      - deployments
      - statefulsets
      - replicasets
      - daemonsets
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["cert-manager.io"]
    resources:
      - certificates
      - certificaterequests
      - issuers
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

kubectl describe role gitlab-cicd-prod-role -n prod
```

### Step 4 — Bind the Role to the ServiceAccount

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-cicd-prod-rolebinding
  namespace: prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gitlab-cicd-prod-role
subjects:
  - kind: ServiceAccount
    name: gitlab-cicd-sa
    namespace: prod
EOF

kubectl describe rolebinding gitlab-cicd-prod-rolebinding -n prod
```

### Step 5 — Create the cluster-scoped ClusterRole

Grants **read-only** access to cluster-wide resources needed by the pipeline (StorageClasses, PersistentVolumes, nodes, namespaces, cert-manager ClusterIssuers).

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gitlab-cicd-cluster-role
rules:
  - apiGroups: [""]
    resources:
      - persistentvolumes
      - nodes
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources:
      - storageclasses
    verbs: ["get", "list", "watch"]
  - apiGroups: ["cert-manager.io"]
    resources:
      - clusterissuers
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - namespaces
    verbs: ["get", "list", "watch", "create"]
EOF

kubectl describe clusterrole gitlab-cicd-cluster-role
```

### Step 6 — Bind the ClusterRole to the ServiceAccount

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitlab-cicd-cluster-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gitlab-cicd-cluster-role
subjects:
  - kind: ServiceAccount
    name: gitlab-cicd-sa
    namespace: prod
EOF

kubectl describe clusterrolebinding gitlab-cicd-cluster-rolebinding
```

---

## Part 2: Generate ServiceAccount Token

Kubernetes 1.24+ no longer auto-generates long-lived tokens. You must create the Secret explicitly.

### Step 7 — Create the token Secret

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-cicd-sa-token
  namespace: prod
  annotations:
    kubernetes.io/service-account.name: gitlab-cicd-sa
type: kubernetes.io/service-account-token
EOF

# Wait for Kubernetes to populate the token
sleep 5
kubectl get secret gitlab-cicd-sa-token -n prod
```

### Step 8 — Verify the token is populated

```bash
kubectl get secret gitlab-cicd-sa-token -n prod \
  -o jsonpath='{.data.token}' | base64 -d | wc -c
```

Expected output: a number **greater than 500**. If it shows `0`, wait 10 seconds and retry.

---

## Part 3: Generate Kubeconfig

### Step 9 — Run the generation script

Save as `generate-kubeconfig.sh` and run once:

```bash
#!/bin/bash
CLUSTER_NAME="devops1-cluster"
REGION="us-east-1"
NAMESPACE="prod"

echo "🔧 Generating kubeconfig for GitLab CI/CD..."

CLUSTER_ENDPOINT=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$REGION" \
  --query "cluster.endpoint" \
  --output text)

CLUSTER_CA=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$REGION" \
  --query "cluster.certificateAuthority.data" \
  --output text)

SA_TOKEN=$(kubectl -n prod get secret gitlab-cicd-sa-token \
  -o jsonpath='{.data.token}' | base64 -d)

cat > gitlab-kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters:
  - name: ${CLUSTER_NAME}
    cluster:
      server: ${CLUSTER_ENDPOINT}
      certificate-authority-data: ${CLUSTER_CA}
users:
  - name: gitlab-cicd-sa
    user:
      token: ${SA_TOKEN}
contexts:
  - name: gitlab-prod
    context:
      cluster: ${CLUSTER_NAME}
      user: gitlab-cicd-sa
      namespace: ${NAMESPACE}
current-context: gitlab-prod
EOF

echo "✅ Kubeconfig generated → gitlab-kubeconfig"
echo "📊 File size: $(du -h gitlab-kubeconfig | cut -f1)"
```

```bash
chmod +x generate-kubeconfig.sh
./generate-kubeconfig.sh
```

> ⚠️ **Never commit `gitlab-kubeconfig` to Git.** Add it to `.gitignore` immediately.

```bash
echo "gitlab-kubeconfig" >> .gitignore
echo "*.kubeconfig" >> .gitignore
```

---

## Part 4: Test Kubeconfig Locally

**Always test before uploading to GitLab.**

```bash
export KUBECONFIG=$PWD/gitlab-kubeconfig

# Context
kubectl config current-context

# Cluster access
kubectl get namespaces
kubectl get nodes

# Prod namespace access
kubectl get pods -n prod
kubectl get deployments -n prod
kubectl get services -n prod

# Cluster-wide resources
kubectl get storageclasses

# Dry-run write permission
kubectl create deployment test --image=nginx --dry-run=client -n prod
```

All commands should succeed. If any return `Forbidden`, revisit the Role or ClusterRole bindings in Part 1.

---

## Part 5: Upload to GitLab

### Step 10 — Encode the kubeconfig

```bash
cat gitlab-kubeconfig | base64 -w0
```

Copy the entire single-line output.

### Step 11 — Add CI/CD variables in GitLab

Navigate to your project: **Settings → CI/CD → Variables → Add Variable**

| Key | Value | Masked | Protected |
|-----|-------|--------|-----------|
| `KUBECONFIG_DATA` | Base64 output from Step 10 | ✅ | ✅ |
| `DOCKER_USERNAME` | Your Docker Hub username | ❌ | ❌ |
| `DOCKER_PASSWORD` | Docker Hub access token | ✅ | ✅ |
| `GITLAB_PUSH_TOKEN` | GitLab Personal Access Token | ✅ | ✅ |
| `SONAR_HOST_URL` | SonarQube server URL | ❌ | ❌ |
| `SONAR_TOKEN` | SonarQube token | ✅ | ✅ |

### Step 12 — Create the GitLab Personal Access Token

**Avatar → Edit Profile → Access Tokens → Add new token**

| Field | Value |
|-------|-------|
| Token name | `gitlab-ci-push-token` |
| Expiration | 1 year from today |
| Scopes | `read_repository`, `write_repository` |

> ⚠️ Copy the token immediately — it won't be shown again.

### Step 13 — Create the Docker Hub Access Token

**Docker Hub → Account Settings → Security → New Access Token**

| Field | Value |
|-------|-------|
| Description | `gitlab-ci-token` |
| Permissions | Read & Write |

> ⚠️ Copy the token immediately — it won't be shown again.

---

## Part 6: Create MySQL Secret

> ⚠️ **Never store database passwords in Git.** Create the secret directly in the cluster.

```bash
# Generate a strong password
openssl rand -base64 32

# Create the secret
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD='<your-strong-password>' \
  -n prod

# Verify
kubectl get secret mysql-secret -n prod
kubectl get secret mysql-secret -n prod \
  -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
```

> If `mysql.yaml` in your manifests contains a hardcoded `MYSQL_ROOT_PASSWORD`, remove the `Secret` block from that file before committing to Git. The pipeline will use the secret you created manually above.

---

## Permissions Reference

### Role: `gitlab-cicd-prod-role` (namespace: `prod`)

| API Group | Resources | Verbs |
|-----------|-----------|-------|
| core (`""`) | pods, pods/log, services, secrets, configmaps, PVCs, endpoints, events | get, list, watch, create, update, patch, delete |
| `apps` | deployments, statefulsets, replicasets, daemonsets | get, list, watch, create, update, patch, delete |
| `networking.k8s.io` | ingresses | get, list, watch, create, update, patch, delete |
| `cert-manager.io` | certificates, certificaterequests, issuers | get, list, watch, create, update, patch, delete |

### ClusterRole: `gitlab-cicd-cluster-role` (cluster-wide, read-only)

| API Group | Resources | Verbs |
|-----------|-----------|-------|
| core (`""`) | persistentvolumes, nodes | get, list, watch |
| `storage.k8s.io` | storageclasses | get, list, watch |
| `cert-manager.io` | clusterissuers | get, list, watch |
| core (`""`) | namespaces | get, list, watch, create |

---

## GitLab CI/CD Variables Reference

How `KUBECONFIG_DATA` is consumed in the deploy job (`.gitlab-ci.yml`):

```yaml
deploy-to-eks-job:
  stage: deploy_to_eks
  image:
    name: bitnami/kubectl:latest
    entrypoint: [""]
  script:
    - echo "$KUBECONFIG_DATA" | base64 -d > kubeconfig
    - export KUBECONFIG=$PWD/kubeconfig
    - kubectl apply -f k8s-prod/ -n prod
```

The `entrypoint: [""]` override is required because the `bitnami/kubectl` image sets `kubectl` as its default entrypoint, which would prevent shell commands from running.

---

## Token Rotation

ServiceAccount tokens should be rotated every **90 days**.

```bash
# 1. Delete the old token secret
kubectl delete secret gitlab-cicd-sa-token -n prod

# 2. Recreate it
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-cicd-sa-token
  namespace: prod
  annotations:
    kubernetes.io/service-account.name: gitlab-cicd-sa
type: kubernetes.io/service-account-token
EOF

# 3. Wait for token population
sleep 10

# 4. Re-generate kubeconfig
./generate-kubeconfig.sh

# 5. Test the new kubeconfig
export KUBECONFIG=$PWD/gitlab-kubeconfig
kubectl get pods -n prod

# 6. Re-encode and update the KUBECONFIG_DATA variable in GitLab
cat gitlab-kubeconfig | base64 -w0

# 7. Clean up local files
rm gitlab-kubeconfig
history -c
```

---

## Troubleshooting

**`Unauthorized` when testing kubeconfig**

The token hasn't been fully populated yet, or the secret annotation is wrong.

```bash
kubectl describe secret gitlab-cicd-sa-token -n prod
# Check that `kubernetes.io/service-account.name: gitlab-cicd-sa` is present
# and that the `token` field is non-empty
```

---

**`Forbidden` in pipeline for a specific resource**

The pipeline is attempting an action not covered by the current Role.

```bash
# Check what's currently granted
kubectl describe role gitlab-cicd-prod-role -n prod
kubectl describe clusterrole gitlab-cicd-cluster-role

# Add the missing resource/verb and reapply
kubectl edit role gitlab-cicd-prod-role -n prod
```

---

**`cluster not found` in generate script**

Wrong cluster name or region.

```bash
aws eks list-clusters --region us-east-1
# Update CLUSTER_NAME in generate-kubeconfig.sh accordingly
```

---

**`kubeconfig: invalid` in pipeline**

Base64 encoding corruption — likely a line-wrap issue.

```bash
# Re-encode with line wrapping disabled
cat gitlab-kubeconfig | base64 -w0 > kubeconfig-b64.txt

# Verify it decodes correctly
cat kubeconfig-b64.txt | base64 -d | head -3
# Should output:
# apiVersion: v1
# kind: Config
# clusters:
```

---

**Token is empty or too short**

Kubernetes hasn't populated the token yet.

```bash
TOKEN_LEN=$(kubectl get secret gitlab-cicd-sa-token -n prod \
  -o jsonpath='{.data.token}' | base64 -d | wc -c)
echo "Token length: $TOKEN_LEN"
# Expected: > 500
```

---

## Security Best Practices

**✅ Do**

- Use **protected + masked** variables in GitLab for all tokens and passwords
- Rotate ServiceAccount tokens every 90 days
- Apply **least-privilege RBAC** — only the verbs and resources the pipeline actually needs
- Store database passwords in Kubernetes Secrets created manually, never in Git
- Add `gitlab-kubeconfig` and `*.kubeconfig` to `.gitignore`
- Use separate ServiceAccounts per environment (e.g., `gitlab-cicd-sa-staging`, `gitlab-cicd-sa-prod`)
- Review RBAC permissions quarterly

**❌ Don't**

- Never commit kubeconfig files, tokens, or passwords to Git
- Never use `cluster-admin` for CI/CD pipelines
- Never share tokens via Slack, email, or SMS
- Never reuse credentials across environments
- Never disable variable masking in GitLab
- Never use default/weak passwords like `admin123` or `password123`

---

## Resources Created — Summary

| Resource | Kind | Scope | Purpose |
|----------|------|-------|---------|
| `gitlab-cicd-sa` | ServiceAccount | `prod` | Identity for pipeline jobs |
| `gitlab-cicd-prod-role` | Role | `prod` | Permissions within prod namespace |
| `gitlab-cicd-prod-rolebinding` | RoleBinding | `prod` | Binds Role to ServiceAccount |
| `gitlab-cicd-cluster-role` | ClusterRole | Cluster | Read-only cluster-wide permissions |
| `gitlab-cicd-cluster-rolebinding` | ClusterRoleBinding | Cluster | Binds ClusterRole to ServiceAccount |
| `gitlab-cicd-sa-token` | Secret | `prod` | Long-lived JWT for kubeconfig |
| `mysql-secret` | Secret | `prod` | MySQL root password (created manually) |
