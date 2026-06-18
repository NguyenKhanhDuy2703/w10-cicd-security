# Alert Setup

## Manual Steps

### 1. Tạo Gmail App Password
```bash
open https://myaccount.google.com/apppasswords
# Tạo password mới, copy 16 ký tự
```

### 2. Apply Email Secret
```bash
# Edit và paste password vào
nano app-alert/email-secret.yaml

# Apply secret (file này bị .argocdignore)
kubectl apply -f app-alert/email-secret.yaml
```

### 3. Verify
```bash
# Check secret exists
kubectl get secret alertmanager-email -n monitoring

# Check Alertmanager running
kubectl get pod -n monitoring -l app.kubernetes.io/name=alertmanager

# Check secret mounted
kubectl exec -n monitoring -c alertmanager \
  $(kubectl get pod -n monitoring -l app.kubernetes.io/name=alertmanager -o name) \
  -- ls /etc/alertmanager/secrets/alertmanager-email/
```

## Files
- `email-secret.yaml` - Gmail credentials ⛔ KHÔNG commit (ignored by ArgoCD)
- `prometheus-rules.yaml` - SLO alert rules ✅ Auto-deployed by ArgoCD
- Alertmanager config → `argocd/apps/k8s-prometheus.yaml` (Helm values)
