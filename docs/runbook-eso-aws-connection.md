# Runbook: External Secrets Operator (ESO) mất kết nối AWS

**Phiên bản:** 1.0  
**Cập nhật lần cuối:** 2026-06-19  
**Áp dụng cho:** Môi trường Minikube local và EKS production  
**Thời gian xử lý ước tính:** 5–15 phút

---

## 1. Mục đích

Runbook này hướng dẫn kỹ sư vận hành xử lý sự cố khi External Secrets Operator (ESO) mất kết nối tới AWS Secrets Manager, dẫn đến Alertmanager không thể xác thực SMTP để gửi email cảnh báo.

---

## 2. Kiến trúc liên quan

Hiểu đúng luồng dữ liệu giúp xác định nhanh điểm lỗi:

```
AWS IAM User (eso-local)
  └── Access Key + Secret Key
        │ lưu trong
        ▼
k8s Secret "aws-credentials" (namespace: external-secrets)
        │ đọc bởi
        ▼
ClusterSecretStore "aws-secrets-store"
        │ dùng để authenticate
        ▼
AWS Secrets Manager (ap-southeast-1)
  └── secret: alertmanager/email
        │ ESO fetch về
        ▼
k8s Secret "alertmanager-email" (namespace: monitoring)
        │ mount vào
        ▼
Alertmanager Pod
  └── /etc/alertmanager/secrets/alertmanager-email/password
```

Mỗi mắt xích trong chuỗi này đều có thể là điểm lỗi.

---

## 3. Dấu hiệu nhận biết sự cố

### 3.1 Triệu chứng ở tầng ứng dụng

- Alertmanager không gửi được email khi có alert.
- Log Alertmanager hiển thị lỗi authentication:
  ```
  msg="Notify for alerts failed" err="dial tcp smtp.gmail.com:587: 
  535 5.7.8 Username and Password not accepted"
  ```
- File password bị rỗng hoặc không tồn tại trong container:
  ```powershell
  kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 `
    -c alertmanager `
    -- cat /etc/alertmanager/secrets/alertmanager-email/password
  # Output rỗng hoặc: cat: No such file or directory
  ```

### 3.2 Triệu chứng ở tầng ESO

```powershell
# Kiểm tra nhanh toàn bộ trạng thái ESO
kubectl get externalsecret alertmanager-email -n monitoring
```

Output bình thường:
```
NAME                 STORE               REFRESH INTERVAL   STATUS         READY
alertmanager-email   aws-secrets-store   5m                 SecretSynced   True
```

Output khi có sự cố:
```
NAME                 STORE               REFRESH INTERVAL   STATUS              READY
alertmanager-email   aws-secrets-store   5m                 SecretSyncedError   False
```

```powershell
# Kiểm tra ClusterSecretStore
kubectl get clustersecretstore aws-secrets-store
```

Output bình thường:
```
NAME                READY   STATUS   AGE
aws-secrets-store   True    Valid    2d
```

Output khi có sự cố:
```
NAME                READY   STATUS    AGE
aws-secrets-store   False   Invalid   2d
```

---

## 4. Sơ đồ quyết định xử lý

```
Phát hiện sự cố (Alertmanager không gửi email)
                    │
                    ▼
     ExternalSecret STATUS = SecretSyncedError?
           │                        │
          YES                       NO
           │                        │
           ▼                        ▼
  ClusterSecretStore             Kiểm tra Alertmanager
  READY = False?                 config và SMTP trực tiếp
    │           │                (nằm ngoài scope runbook này)
   YES          NO
    │           │
    ▼           ▼
aws-credentials   Xem Events chi tiết
Secret tồn tại?   của ExternalSecret
  │      │        → kubectl describe es
 NO     YES            alertmanager-email
  │      │                    -n monitoring
  ▼      ▼
Tạo lại  IAM policy     Secret trên AWS
Secret   bị thiếu?      tồn tại không?
         │      │          │       │
        YES     NO         NO      YES
         │      │          │       │
         ▼      ▼          ▼       ▼
      Attach  Kiểm tra  Tạo lại  Force Sync
      lại     region    secret   → xem bước 4.4
      policy  trong CSS trên AWS
```

---

## 5. Các bước khắc phục chi tiết

### Bước 5.1: Xác định điểm lỗi chính xác

```powershell
# Xem toàn bộ status và events của ExternalSecret
kubectl describe externalsecret alertmanager-email -n monitoring
```

Tìm phần `Events` ở cuối output — đây là nơi ESO ghi lại lỗi cụ thể nhất:

```
Events:
  Type     Reason          Age    Message
  ----     ------          ----   -------
  Warning  UpdateFailed    2m     error retrieving secret: AccessDeniedException
```

```powershell
# Xem status ClusterSecretStore chi tiết
kubectl describe clustersecretstore aws-secrets-store
```

Tìm phần `Status.Conditions`:
```yaml
Conditions:
  Last Transition Time: 2026-06-19T10:00:00Z
  Message: InvalidClientTokenId: The security token included in the request is invalid.
  Reason: ValidationFailed
  Status: "False"
  Type: Ready
```

---

### Bước 5.2: Xử lý lỗi theo từng loại

#### Lỗi `InvalidClientTokenId` — Access Key ID không hợp lệ

**Nguyên nhân:** Access Key ID trong `aws-credentials` Secret bị sai, đã bị xóa trên AWS, hoặc Secret bị mất sau khi restart Minikube.

```powershell
# Kiểm tra Secret còn tồn tại không
kubectl get secret aws-credentials -n external-secrets
# Nếu không có output → Secret bị mất → tạo lại (xem bước 5.3)

# Nếu còn → decode để kiểm tra giá trị
$encoded = kubectl get secret aws-credentials -n external-secrets `
  -o jsonpath='{.data.access-key}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
# Access Key ID hợp lệ phải bắt đầu bằng "AKIA" (static) hoặc "ASIA" (temporary)

# Kiểm tra Access Key còn active trên AWS không
aws iam list-access-keys --user-name eso-local
# Cột Status phải là: Active
```

---

#### Lỗi `SignatureDoesNotMatch` — Secret Access Key sai

**Nguyên nhân:** Secret Access Key trong `aws-credentials` bị sai.  
**Lưu ý:** AWS không cho phép xem lại Secret Access Key sau khi tạo — phải tạo key mới.

```powershell
# Xóa Access Key cũ trên AWS
aws iam delete-access-key `
  --user-name eso-local `
  --access-key-id <AccessKeyId-cũ>

# Tạo Access Key mới — LƯU LẠI OUTPUT NGAY
aws iam create-access-key --user-name eso-local
# Output chứa AccessKeyId và SecretAccessKey (chỉ hiện 1 lần)

# Xóa Secret cũ và tạo lại với key mới
kubectl delete secret aws-credentials -n external-secrets
kubectl create secret generic aws-credentials `
  --namespace external-secrets `
  --from-literal=access-key=<AccessKeyId-mới> `
  --from-literal=secret-access-key=<SecretAccessKey-mới>
```

---

#### Lỗi `AccessDeniedException` — IAM thiếu quyền

**Nguyên nhân:** IAM User `eso-local` không có quyền đọc secret `alertmanager/email`.

```powershell
# Xem policy hiện tại
aws iam get-user-policy `
  --user-name eso-local `
  --policy-name eso-secrets-policy

# Gắn lại policy từ file (đảm bảo đang ở thư mục gốc project)
aws iam put-user-policy `
  --user-name eso-local `
  --policy-name eso-secrets-policy `
  --policy-document file://policy.json

# Verify policy đã được gắn
aws iam get-user-policy `
  --user-name eso-local `
  --policy-name eso-secrets-policy
```

Nội dung `policy.json` đúng phải là:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret"
    ],
    "Resource": "arn:aws:secretsmanager:ap-southeast-1:*:secret:alertmanager/*"
  }]
}
```

---

#### Lỗi `ResourceNotFoundException` — Secret không tồn tại trên AWS

**Nguyên nhân:** Secret `alertmanager/email` chưa được tạo hoặc bị xóa trên AWS Secrets Manager, hoặc sai region.

```powershell
# Kiểm tra secret có tồn tại không
aws secretsmanager describe-secret `
  --secret-id alertmanager/email `
  --region ap-southeast-1

# Nếu không tồn tại → tạo lại
aws secretsmanager create-secret `
  --name alertmanager/email `
  --region ap-southeast-1 `
  --secret-string '{\"password\":\"ehyp jiqs bshf gkau\"}'

# Kiểm tra region trong ClusterSecretStore có đúng không
kubectl get clustersecretstore aws-secrets-store -o jsonpath='{.spec.provider.aws.region}'
# Phải trả về: ap-southeast-1
```

---

### Bước 5.3: Tạo lại `aws-credentials` Secret (khi bị mất)

Thường xảy ra sau `minikube delete` hoặc cluster reset.

```powershell
# Bước 1: Lấy Access Key từ AWS (nếu còn key cũ)
aws iam list-access-keys --user-name eso-local
# Ghi lại AccessKeyId

# Bước 2: Tạo namespace nếu chưa có
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -

# Bước 3: Tạo lại Secret
# Thay <AccessKeyId> và <SecretAccessKey> bằng giá trị thật
kubectl create secret generic aws-credentials `
  --namespace external-secrets `
  --from-literal=access-key=<AccessKeyId> `
  --from-literal=secret-access-key=<SecretAccessKey>

# Bước 4: Verify
kubectl get secret aws-credentials -n external-secrets
# Cột DATA phải là: 2
```

---

### Bước 5.4: Force Sync — Ép đồng bộ ngay lập tức

Sau khi đã sửa bất kỳ vấn đề nào ở trên, không cần đợi 5 phút (`refreshInterval`):

```powershell
# Gắn annotation để trigger reconcile ngay
$timestamp = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
kubectl annotate externalsecret alertmanager-email `
  "force-sync=$timestamp" `
  --overwrite `
  -n monitoring

# Đợi ESO xử lý
Start-Sleep -Seconds 10

# Kiểm tra kết quả
kubectl get externalsecret alertmanager-email -n monitoring
# Mong đợi: STATUS=SecretSynced, READY=True
```

---

### Bước 5.5: Xác minh end-to-end

```powershell
# 1. ClusterSecretStore phải Valid
kubectl get clustersecretstore aws-secrets-store
# READY=True, STATUS=Valid

# 2. ExternalSecret phải Synced
kubectl get externalsecret alertmanager-email -n monitoring
# STATUS=SecretSynced, READY=True

# 3. k8s Secret phải tồn tại
kubectl get secret alertmanager-email -n monitoring
# TYPE=Opaque, DATA=1

# 4. Giá trị password phải đúng
$encoded = kubectl get secret alertmanager-email -n monitoring `
  -o jsonpath='{.data.password}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
# Mong đợi: ehyp jiqs bshf gkau

# 5. Alertmanager Pod đọc được file password
kubectl exec -n monitoring `
  alertmanager-kube-prometheus-stack-alertmanager-0 `
  -c alertmanager `
  -- cat /etc/alertmanager/secrets/alertmanager-email/password
# Mong đợi: ehyp jiqs bshf gkau
```

---

### Bước 5.6: Xem log ESO để debug sâu hơn

```powershell
# Log của ESO controller (nguồn thông tin chi tiết nhất)
kubectl logs -n external-secrets `
  -l app.kubernetes.io/name=external-secrets `
  --tail=100 `
  --timestamps

# Lọc chỉ lỗi liên quan đến alertmanager
kubectl logs -n external-secrets `
  -l app.kubernetes.io/name=external-secrets `
  --tail=200 | Select-String "alertmanager|error|Error"
```

---

## 6. Bảng tổng hợp lỗi và xử lý

| Mã lỗi | Tầng xảy ra | Nguyên nhân phổ biến | Bước xử lý |
|---|---|---|---|
| `InvalidClientTokenId` | ClusterSecretStore | Access Key ID sai/bị xóa | 5.2 → tạo key mới |
| `SignatureDoesNotMatch` | ClusterSecretStore | Secret Access Key sai | 5.2 → tạo key mới |
| `AccessDeniedException` | ClusterSecretStore | IAM thiếu quyền | 5.2 → gắn lại policy |
| `ResourceNotFoundException` | ExternalSecret | Secret AWS không tồn tại hoặc sai region | 5.2 → tạo secret AWS |
| `SecretStoreNotReady` | ExternalSecret | ClusterSecretStore chưa Valid | Xử lý CSS trước |
| `secret not found` | ClusterSecretStore | `aws-credentials` k8s Secret bị mất | 5.3 → tạo lại Secret |
| `property not found` | ExternalSecret | Sai field `property` trong ExternalSecret | Sửa `email-externalsecret.yaml` |

---

## 7. Phòng ngừa

- **Backup Access Key:** Lưu Access Key vào password manager ngay sau khi tạo.
- **Minikube restart:** Sau mỗi `minikube delete`, phải chạy lại bước tạo `aws-credentials` Secret trước khi apply ArgoCD root.
- **Monitor:** Thiết lập alert Prometheus theo dõi trạng thái ExternalSecret nếu có thể.
- **Rotate key định kỳ:** Thay Access Key mỗi 90 ngày theo best practice AWS.

---

## 8. Thông tin tham chiếu

| Tài nguyên | Chi tiết |
|---|---|
| ClusterSecretStore | `app-eso/cluster-secret-store.yaml` |
| ExternalSecret | `app-alert/email-externalsecret.yaml` |
| IAM Policy | `policy.json` |
| AWS Secret | `alertmanager/email` tại region `ap-southeast-1` |
| IAM User | `eso-local` |
| k8s Secret credentials | `aws-credentials` tại namespace `external-secrets` |
| k8s Secret output | `alertmanager-email` tại namespace `monitoring` |
| ESO namespace | `external-secrets` |
