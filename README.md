# W10 - Progressive Delivery with Analysis

GitOps setup for API deployment với Argo Rollouts + AnalysisTemplate.

## Concept

Deploy API với **canary strategy** và **automated analysis**:
- Rollout: 10% → 50% → 100%
- AnalysisTemplate query Prometheus để check success rate ≥ 95%
- Auto rollback nếu analysis fail
- AlertManager gửi email khi có SLO violation
- **ESO (External Secrets Operator)**: Quản lý secret an toàn qua AWS Secrets Manager

## Requirements

- Docker Desktop
- kubectl
- minikube
- git
- AWS CLI (đã configure credentials)
- Helm

## Structure

```
w10/
├── app-api/              # API Rollout manifests
│   ├── rollout.yaml      # Argo Rollout với canary strategy
│   ├── service.yaml      # Service expose API
│   └── servicemonitor.yaml # Prometheus metrics scraper
├── app-analysis/         # Analysis manifests
│   └── analysis-template.yaml # Template phân tích success rate
├── app-alert/            # Alert manifests
│   ├── prometheus-rules.yaml      # PrometheusRule cho SLO alerts
│   └── email-externalsecret.yaml  # ExternalSecret (thay thế email-secret.yaml)
├── app-common/           # Common resources
│   └── demo-namespace.yaml # Namespace demo
├── app-eso/              # ESO config
│   └── cluster-secret-store.yaml  # Kết nối AWS Secrets Manager
├── src/                  # Source code
│   └── api/              # Flask API application
├── argocd/
│   ├── apps/             # ArgoCD Application manifests
│   │   ├── eso-operator.yaml     # Cài ESO Controller (wave -3)
│   │   ├── eso-config.yaml       # Deploy ClusterSecretStore (wave -2)
│   │   ├── app-api.yaml          # Deploy API Rollout
│   │   ├── app-analysis.yaml     # Deploy AnalysisTemplate
│   │   ├── app-alert.yaml        # Deploy PrometheusRule + ExternalSecret
│   │   ├── app-common.yaml       # Deploy common resources
│   │   ├── k8s-prometheus.yaml   # Prometheus + AlertManager
│   │   └── k8s-rollout.yaml      # Argo Rollouts controller
│   └── root.yaml         # App of Apps pattern
└── README.md
```

## Sync Wave Order

ArgoCD applications deploy theo thứ tự:
```
Wave -5  → gatekeeper      (Gatekeeper controller)
Wave -3  → eso-operator    (ESO Controller + CRDs)
Wave -2  → eso-config      (ClusterSecretStore)
Wave -1  → app-common      (Namespace)
Wave  0  → k8s-prometheus, k8s-rollout (Infrastructure)
Wave  1  → app-alert, app-analysis     (ExternalSecret, Config)
Wave  2  → app-api                     (Application)
```

---

## Quick Start (PowerShell - Windows)

### 1. Setup Cluster

```powershell
minikube start -p w10 --driver=docker
kubectl config use-context w10
```

### 2. Install ArgoCD

```powershell
kubectl create ns argocd
kubectl apply --server-side -n argocd `
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

### 3. Access ArgoCD UI

```powershell
# Port forward (chạy trong terminal riêng)
kubectl -n argocd port-forward svc/argocd-server 8080:443

# Lấy password (terminal khác)
$password = kubectl -n argocd get secret argocd-initial-admin-secret `
  -o jsonpath='{.data.password}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))
```

Truy cập: https://localhost:8080 | Username: `admin`

### 4. Setup ESO — Lưu secret lên AWS (chạy 1 lần)

```powershell
# Bước 4a: Tạo secret trên AWS Secrets Manager
aws secretsmanager create-secret `
  --name alertmanager/email `
  --region ap-southeast-1 `
  --secret-string '{\"password\":\"ehyp jiqs bshf gkau\"}'

# Verify
aws secretsmanager get-secret-value `
  --secret-id alertmanager/email `
  --region ap-southeast-1
```

### 5. Setup ESO — Tạo IAM User

```powershell
# Tạo IAM User
aws iam create-user --user-name eso-local

# Gắn policy từ file policy.json (chỉ đọc alertmanager/*)
# policy.json nằm ở root của project
aws iam put-user-policy `
  --user-name eso-local `
  --policy-name eso-secrets-policy `
  --policy-document file://policy.json

# Tạo Access Key — LƯU LẠI output này (chỉ hiện 1 lần)
aws iam create-access-key --user-name eso-local
```

### 6. Setup ESO — Lưu credentials vào Minikube (KHÔNG commit Git)

```powershell
# Tạo namespace trước
kubectl create namespace external-secrets

# Tạo Secret chứa AWS credentials
# Thay <ACCESS_KEY_ID> và <SECRET_ACCESS_KEY> bằng giá trị từ bước 5
kubectl create secret generic aws-credentials `
  --namespace external-secrets `
  --from-literal=access-key=<ACCESS_KEY_ID> `
  --from-literal=secret-access-key=<SECRET_ACCESS_KEY>

# Verify
kubectl get secret aws-credentials -n external-secrets
```

### 7. Deploy App of Apps

```powershell
kubectl apply -f argocd/root.yaml
```

ArgoCD sẽ tự động deploy theo đúng sync wave order.

---

## Verify Deployment (PowerShell)

### Kiểm tra ESO

```powershell
# ESO pods đang chạy?
kubectl get pods -n external-secrets

# ClusterSecretStore kết nối AWS ok?
kubectl get clustersecretstore
# Mong đợi: READY=True, STATUS=Valid

# ExternalSecret đã sync chưa?
kubectl get externalsecret -n monitoring
# Mong đợi: STATUS=SecretSynced, READY=True

# k8s Secret đã được tạo chưa?
kubectl get secret alertmanager-email -n monitoring

# Decode giá trị để verify
$encoded = kubectl get secret alertmanager-email -n monitoring `
  -o jsonpath='{.data.password}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
# Mong đợi: ehyp jiqs bshf gkau
```

### Kiểm tra Rollout

```powershell
# Watch rollout progress
kubectl get rollout api -n demo -w

# Check current state
kubectl get rollout api -n demo

# Check pods
kubectl get pods -n demo -l app=api
```

### Kiểm tra AnalysisRun

```powershell
# List analysis runs
kubectl get analysisrun -n demo

# Describe chi tiết
kubectl describe analysisrun -n demo <name>
```

### Query Prometheus Metrics

```powershell
kubectl run test-query --image=curlimages/curl:latest --rm -i --restart=Never -n monitoring -- `
  curl -s 'http://kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/query?query=api:success_rate:5m'
```

---

## Test Scenarios (PowerShell)

### Test 1: Successful Deployment (Success Rate ≥ 90%)

```powershell
# Sửa file rollout - đặt ERROR_RATE = 0
# Mở file bằng notepad hoặc VS Code
notepad app-api\rollout.yaml
# Set: ERROR_RATE: "0"

git add app-api/rollout.yaml
git commit -m "test: deploy with 0% error rate"
git push origin main

# Watch AnalysisRun succeed
kubectl get analysisrun -n demo -w
```

### Test 2: Failed Deployment (Success Rate < 90%)

```powershell
notepad app-api\rollout.yaml
# Set: ERROR_RATE: "0.15"

git add app-api/rollout.yaml
git commit -m "test: deploy with 15% error rate (should fail)"
git push origin main

# Watch AnalysisRun fail và auto rollback
kubectl get analysisrun -n demo -w
kubectl get rollout api -n demo
```

### Test 3: Trigger SLO Alert Email

```powershell
notepad app-api\rollout.yaml
# Set: ERROR_RATE: "0.10"

git add app-api/rollout.yaml
git commit -m "test: deploy with 10% error rate"
git push origin main

# Canary passes (≥90%) nhưng SLO alert fires (below 95%)
# Đợi 2-3 phút rồi check email
```

### Test 4: Force sync ESO secret (không chờ 5 phút)

```powershell
# Annotate để trigger sync ngay
$timestamp = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
kubectl annotate externalsecret alertmanager-email `
  "force-sync=$timestamp" `
  --overwrite `
  -n monitoring

# Verify sau vài giây
kubectl get externalsecret alertmanager-email -n monitoring
```

---

## Debug (PowerShell)

### ESO logs

```powershell
# Xem log ESO controller
kubectl logs -n external-secrets `
  -l app.kubernetes.io/name=external-secrets `
  --tail=50

# Xem events của ExternalSecret
kubectl describe externalsecret alertmanager-email -n monitoring

# Xem events của ClusterSecretStore
kubectl describe clustersecretstore aws-secrets-store
```

### Lỗi thường gặp

```
ClusterSecretStore STATUS=Invalid:
  AccessDeniedException       → IAM policy chưa đúng
  InvalidClientTokenId        → Access Key ID sai
  SignatureDoesNotMatch       → Secret Access Key sai
  ResourceNotFoundException   → region sai hoặc secret chưa tạo

ExternalSecret STATUS=SecretSyncedError:
  SecretStoreNotReady         → ClusterSecretStore chưa Valid
  ResourceNotFoundException   → sai tên secret trên AWS
  property not found          → sai property trong JSON
```

---

## Cleanup (PowerShell)

```powershell
# Xóa ArgoCD applications
kubectl delete -f argocd/root.yaml

# Đợi resources được dọn dẹp
kubectl get all -n demo
kubectl get all -n monitoring

# Xóa ArgoCD
kubectl delete ns argocd

# Xóa IAM resources trên AWS (nếu cần)
aws iam delete-access-key `
  --user-name eso-local `
  --access-key-id <ACCESS_KEY_ID>
aws iam delete-user-policy --user-name eso-local --policy-name eso-secrets-policy
aws iam delete-user --user-name eso-local

# Xóa secret trên AWS (nếu cần)
aws secretsmanager delete-secret `
  --secret-id alertmanager/email `
  --force-delete-without-recovery

# Stop minikube
minikube stop -p w10
minikube delete -p w10
```
