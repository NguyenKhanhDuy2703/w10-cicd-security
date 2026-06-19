# ADR 001: Ngoại lệ cấu hình thủ công cho email-secret.yaml

| Trường | Giá trị |
|---|---|
| Số hiệu | ADR-001 |
| Ngày tạo | 2026-06-19 |
| Trạng thái | **Accepted** |
| Người quyết định | Team w10-cicd-security |
| Ngày hết hạn ngoại lệ | Khi ESO ổn định trên EKS production |

---

## 1. Bối cảnh

### 1.1 Nguyên tắc GitOps đang áp dụng

Toàn bộ dự án `w10-cicd-security` vận hành theo mô hình GitOps thông qua ArgoCD với nguyên tắc cốt lõi:

> **Git là nguồn sự thật duy nhất (single source of truth).** Mọi trạng thái của cluster phải được khai báo trong Git và được ArgoCD tự động đồng bộ hóa.

Cụ thể, ArgoCD được cấu hình với:
```yaml
syncPolicy:
  automated:
    prune: true      # Xóa resource không có trong Git
    selfHeal: true   # Tự phục hồi khi bị sửa trực tiếp trên cluster
```

### 1.2 Vấn đề phát sinh

File `app-alert/email-secret.yaml` chứa Gmail App Password dùng cho Alertmanager gửi email cảnh báo SLO violation. File này có cấu trúc:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-email
  namespace: monitoring
type: Opaque
stringData:
  password: ehyp jiqs bshf gkau   # ← Gmail App Password dạng plaintext
```

Kubernetes Secret sử dụng `stringData` hoặc `data` (base64) — đây **không phải mã hóa**. Base64 có thể decode dễ dàng bằng bất kỳ công cụ nào. Việc commit file này lên Git repository đồng nghĩa với việc password được lưu vĩnh viễn trong Git history, có thể bị đọc bởi:
- Bất kỳ ai có quyền đọc repository (nếu repo public).
- Bất kỳ contributor nào (nếu repo private nhưng có nhiều thành viên).
- Công cụ scan bảo mật tự động (GitGuardian, truffleHog...).
- Bất kỳ ai clone repo trong tương lai, kể cả sau khi xóa file.

### 1.3 Giai đoạn chuyển đổi hiện tại

Dự án đang trong lộ trình chuyển từ quản lý secret thủ công sang External Secrets Operator (ESO):

```
Giai đoạn hiện tại:                    Giai đoạn mục tiêu:
─────────────────────                  ──────────────────────
email-secret.yaml (thủ công)           email-externalsecret.yaml (ESO)
+ email-externalsecret.yaml (ESO)      → AWS Secrets Manager
→ song song, ESO là primary            → không còn file thủ công
```

ESO chưa được xác nhận ổn định hoàn toàn trên môi trường production (EKS + IRSA). Trong giai đoạn này, cần có cơ chế fallback.

---

## 2. Quyết định

### 2.1 Nội dung quyết định

**Loại bỏ `email-secret.yaml` khỏi vòng đời quản lý tự động của ArgoCD.**

Cụ thể:
1. File `email-secret.yaml` được thêm vào `app-alert/.argocdignore`.
2. ArgoCD sẽ không bao giờ sync, prune, hoặc selfHeal file này.
3. Kỹ sư vận hành phải chủ động `kubectl apply` file này thủ công khi cần.
4. File `email-secret.yaml` **không được commit** lên Git repository dưới bất kỳ hình thức nào.
5. `email-externalsecret.yaml` là cơ chế primary để Alertmanager lấy password.

### 2.2 Cách áp dụng thủ công

Khi cluster được tạo mới hoặc cần khôi phục fallback:

```powershell
# Tạo file tạm thời (KHÔNG commit file này)
# Thay <PASSWORD> bằng Gmail App Password thật
@"
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-email
  namespace: monitoring
type: Opaque
stringData:
  password: <PASSWORD>
"@ | kubectl apply -f -

# Xác nhận đã tạo
kubectl get secret alertmanager-email -n monitoring
```

Hoặc dùng `--from-literal` để tránh hoàn toàn việc ghi ra file:

```powershell
kubectl create secret generic alertmanager-email `
  --namespace monitoring `
  --from-literal=password=<PASSWORD> `
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 3. Lý do chi tiết

### 3.1 Rủi ro bảo mật nếu commit

| Rủi ro | Mức độ | Giải thích |
|---|---|---|
| Lộ password qua Git history | **Nghiêm trọng** | Dù xóa file sau đó, password vẫn còn trong `git log` |
| GitHub scan phát hiện | **Cao** | GitHub có tính năng Secret Scanning tự động |
| Clone repo bị lộ | **Cao** | Bất kỳ ai clone repo đều có password |
| CI/CD pipeline log | **Trung bình** | Một số tool có thể in Secret value vào log |

Base64 thường bị nhầm là mã hóa, nhưng thực chất chỉ là encoding:
```
echo "ehyp jiqs bshf gkau" | base64
# ZWh5cCBqaXFzIGJzaGYgZ2thdQo=

echo "ZWh5cCBqaXFzIGJzaGYgZ2thdQo=" | base64 -d
# ehyp jiqs bshf gkau   ← ai cũng decode được
```

### 3.2 Tại sao cần fallback trong giai đoạn này

ESO phụ thuộc vào nhiều thành phần có thể gặp sự cố:

```
ESO → AWS Secrets Manager → IAM credentials → k8s Secret
```

Nếu bất kỳ mắt xích nào fail (AWS outage, hết hạn access key, IAM policy bị xóa nhầm), Alertmanager sẽ không có password → không gửi được email cảnh báo → mất visibility vào hệ thống.

Fallback thủ công đảm bảo:
- Kỹ sư có thể restore alerting trong vòng 2 phút khi ESO gặp sự cố.
- Không phụ thuộc vào tính sẵn sàng của AWS trong quá trình debug.

### 3.3 Tại sao chưa dùng giải pháp khác

| Giải pháp thay thế | Lý do chưa áp dụng |
|---|---|
| **Sealed Secrets** | Cần cài thêm Bitnami Sealed Secrets controller; phức tạp hơn mức cần thiết khi ESO đã được chọn là long-term solution |
| **SOPS + age/GPG** | Yêu cầu quản lý keypair, thêm bước trong CI/CD; overhead không cần thiết khi ESO sẽ thay thế hoàn toàn |
| **Chỉ dùng ESO, không fallback** | Rủi ro không kiểm soát được trong giai đoạn đầu; vi phạm nguyên tắc reliability |
| **Hardcode trong Helm values** | Tương đương commit plaintext, vẫn vi phạm bảo mật |

---

## 4. Hệ quả

### 4.1 Hệ quả tích cực

- **Bảo mật:** Gmail App Password không bao giờ tồn tại trong Git repository hay Git history.
- **Độ ổn định:** Alertmanager có cơ chế dự phòng khi ESO hoặc AWS gặp sự cố.
- **Tính linh hoạt:** Kỹ sư có thể cập nhật password ngay lập tức mà không cần qua CI/CD pipeline.
- **Đơn giản:** Không cần cài thêm công cụ mới trong giai đoạn chuyển tiếp.

### 4.2 Hệ quả tiêu cực và cách giảm thiểu

| Hệ quả | Mức ảnh hưởng | Cách giảm thiểu |
|---|---|---|
| Vi phạm GitOps thuần túy | Trung bình | Ghi rõ trong ADR; thời gian giới hạn |
| Phụ thuộc vào trí nhớ kỹ sư | Cao | Ghi vào checklist setup cluster; Runbook ESO |
| Không có audit trail | Trung bình | Log lại trong ticket/wiki khi apply thủ công |
| Nguy cơ drift | Thấp | `deletionPolicy: Retain` trong ESO đảm bảo Secret không bị ArgoCD xóa |

### 4.3 Tác động lên quy trình vận hành

Kỹ sư phải thực hiện thêm bước thủ công trong 2 tình huống:

**Tình huống 1: Tạo cluster mới (từ đầu)**
```
Checklist setup cluster:
  ✅ kubectl apply -f argocd/root.yaml
  ✅ kubectl create secret generic aws-credentials ...   (ESO credentials)
  ✅ kubectl apply alertmanager-email secret thủ công    (← bước bổ sung này)
```

**Tình huống 2: ESO gặp sự cố, cần restore alerting ngay**
```
1. Xác nhận ESO lỗi (xem Runbook ESO)
2. kubectl apply alertmanager-email secret thủ công
3. kubectl rollout restart deployment -n monitoring alertmanager (nếu cần)
4. Kiểm tra Alertmanager gửi được email
5. Song song: debug và phục hồi ESO
```

---

## 5. Điều kiện để đóng ngoại lệ

Ngoại lệ này sẽ được loại bỏ hoàn toàn khi **tất cả** điều kiện sau được đáp ứng:

- [ ] ESO hoạt động ổn định trên EKS production với IRSA (không dùng static Access Key).
- [ ] Đã vận hành ESO liên tục trong ít nhất 30 ngày không có incident liên quan.
- [ ] Team đã có quy trình rotate Access Key / IRSA role rõ ràng.
- [ ] Monitoring cho ExternalSecret status đã được thiết lập.

**Hành động khi đóng ngoại lệ:**
1. Xóa `email-secret.yaml` khỏi repository (kể cả file mẫu nếu có).
2. Xóa entry `email-secret.yaml` khỏi `app-alert/.argocdignore`.
3. Chạy `git filter-repo` để xóa khỏi Git history nếu file đã từng được commit.
4. Cập nhật ADR này sang trạng thái `Superseded`.
5. Cập nhật Runbook ESO để bỏ phần fallback thủ công.

---

## 6. Tham chiếu

- Runbook xử lý sự cố ESO: `docs/runbook-eso-aws-connection.md`
- File ExternalSecret (primary): `app-alert/email-externalsecret.yaml`
- File ClusterSecretStore: `app-eso/cluster-secret-store.yaml`
- IAM Policy: `policy.json`
- ArgoCD ignore config: `app-alert/.argocdignore`
