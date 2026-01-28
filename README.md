## Kiến trúc nội bộ của một Argo CD Application

Trong Argo CD, một `Application` không chạy “một mình”, mà dựa trên nhiều thành phần dịch vụ bên trong namespace `argocd`. Dưới đây là các thành phần chính, vai trò và cách chúng liên quan với nhau.

### 1. Dịch vụ chính (Services)

#### 1.1 `argocd-server`
- **Vai trò**:
  - Cung cấp **Web UI** và **REST/gRPC API**.
  - Xử lý **authentication & authorization** (login, RBAC).
  - Tiếp nhận lệnh từ người dùng/CI/CD:
    - `Sync`, `Diff`, `Rollback`, xem history, logs…
- **Liên quan Application**:
  - Khi bạn xem Application trên UI, bấm Sync, Rollback…, tất cả đi qua `argocd-server`.
  - `argocd-server` gọi tiếp xuống `argocd-application-controller` để thực thi.

---

#### 1.2 `argocd-application-controller`
- **Vai trò**:
  - Là “bộ não GitOps” của Argo CD.
  - Liên tục **watch** các resource `Application` (CRD) trong K8s.
  - Với mỗi Application:
    - Lấy desired state từ Git (qua repo-server).
    - Lấy live state từ K8s API (cluster hiện tại).
    - Tính toán **diff** (OutOfSync/Healthy/Degraded).
    - Nếu cấu hình auto-sync:
      - Áp dụng manifest (sync),
      - Xóa resource thừa (prune),
      - Tự heal khi resource bị chỉnh tay (self-heal).
- **Liên quan Application**:
  - Mọi update trạng thái `Synced/OutOfSync`, `Healthy/Degraded` bạn thấy trên UI đều do controller tính và ghi lại vào `status` của `Application`.

---

#### 1.3 `argocd-repo-server`
- **Vai trò**:
  - Chuyên xử lý phần **render manifests từ Git**:
    - Clone repo từ `spec.source.repoURL`.
    - Checkout `targetRevision` (branch/tag/commit).
    - Vào `path` (helm/kustomize/yaml).
    - Chạy:
      - `helm template` (nếu source là Helm).
      - `kustomize build` (nếu là Kustomize).
      - Đọc YAML thuần (nếu là directory).
  - Trả về manifest YAML đã render cho `application-controller`.
- **Liên quan Application**:
  - Mỗi lần Application sync, `argocd-application-controller` gọi `argocd-repo-server` để lấy manifest mới nhất từ Git cho Application đó.

---

### 2. Cache

#### 2.1 `argocd-redis`
- **Vai trò**:
  - Là **cache layer** cho Argo CD:
    - Cache kết quả clone repo, render Helm/Kustomize.
    - Cache metadata Application/Project, diff, trạng thái Health/Sync.
  - Giúp:
    - UI load nhanh hơn,
    - Giảm số lần clone Git/render lại chart,
    - Giảm tải cho repo-server và controller.
- **Liên quan Application**:
  - Khi bạn mở UI xem Application, Argo có thể lấy nhiều thông tin từ cache Redis thay vì phải gọi lại K8s/Git mỗi lần.

---

### 3. Database / Lưu trữ trạng thái

Argo CD có thể dùng 2 nguồn chính để lưu trữ state:

#### 3.1 etcd (qua Kubernetes API)
- **Vai trò**:
  - Là nơi K8s lưu mọi resource, bao gồm:
    - `Application` CRD
    - `AppProject`
    - ConfigMap/Secret cấu hình Argo CD
  - Đây là **nguồn sự thật chính (source of truth)** cho desired state.
- **Liên quan Application**:
  - Mỗi Application bạn tạo (`kubectl apply -f app.yaml` hoặc từ UI) được lưu như CRD trong etcd.
  - `status` (Synced/OutOfSync/Healthy/Degraded) cũng được controller cập nhật vào CRD, lưu trong etcd.

#### 3.2 PostgreSQL (tuỳ chọn, nếu bạn bật)
- Trong nhiều cài đặt OSS, Argo CD không cần Postgres riêng; trạng thái đủ dùng với CRD + Redis.
- Một số kiến trúc enterprise hoặc chart cấu hình thêm Postgres để:
  - Lưu history, events, RBAC nâng cao, audit logs.
- **Liên quan Application**:
  - Lưu sâu hơn về lịch sử sync/deploy, user action… (tùy cấu hình).

---

### 4. Luồng dữ liệu cho một Application

1. **Application CRD** được tạo trong `namespace: argocd` → lưu vào etcd.
2. **argocd-application-controller**:
   - Watch Application này.
   - Gửi request sang `argocd-repo-server` để lấy manifest từ Git.
3. **argocd-repo-server**:
   - Clone Git, render manifest (Helm/Kustomize/YAML) → kết quả có thể cache trong Redis.
4. Controller so sánh manifest với state hiện tại trong cluster (qua K8s API):
   - Nếu OutOfSync và có auto-sync → gọi K8s apply (sync), prune, self-heal.
5. **argocd-server**:
   - Trả UI/API:
     - Trạng thái Application (Synced/OutOfSync, Healthy/Degraded),
     - Diff, History, Logs.
   - Gửi lệnh sync/rollback từ UI xuống controller.

---

### 5. Tóm tắt

- **Services**:
  - `argocd-server` – giao diện, API, auth.
  - `argocd-application-controller` – reconcile logic, sync/prune/self-heal.
  - `argocd-repo-server` – render Helm/Kustomize/YAML từ Git.
- **Cache**:
  - `argocd-redis` – tăng tốc truy vấn, render, status.
- **Database (state)**:
  - etcd (qua CRD `Application`, `AppProject`…).
  - Có thể thêm Postgres tuỳ deployment.

Một `Application` thực chất là “hợp đồng GitOps” giữa Git ↔ Argo CD ↔ Kubernetes, và 3 service chính (server, controller, repo-server) + Redis + etcd/Postgres là các khối xây dựng bên dưới để duy trì hợp đồng đó.
