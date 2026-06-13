# 🚀 KLTN K8s Manifests — GitOps với ArgoCD

Kho lưu trữ này chứa toàn bộ **Kubernetes manifests** cho dự án KLTN (Khóa Luận Tốt Nghiệp) — một nền tảng du lịch trực tuyến (`uittravel.shop`). Các manifest được tổ chức theo mô hình **Kustomize Base/Overlay** và được triển khai tự động thông qua **ArgoCD** theo phương pháp **GitOps**. Hệ thống cũng tích hợp cấu hình **Prometheus ServiceMonitor** để giám sát các ứng dụng và ArgoCD.

---

## 📁 Cấu trúc thư mục

```text
k8s-manifests/
├── argocd/                        # Các định nghĩa Application của ArgoCD
│   ├── kltn-dev-app.yaml          # App định nghĩa cho môi trường Dev
│   ├── kltn-prod-app.yaml         # App định nghĩa cho môi trường Prod
│   └── kltn-monitoring-app.yaml   # App định nghĩa cho cấu hình Monitoring
│
├── base/                          # Manifests gốc (dùng chung cho mọi môi trường)
│   ├── kustomization.yaml         # Danh sách tất cả resources gốc
│   ├── configmap.yaml             # ConfigMap chứa URL nội bộ các services
│   ├── api-gateway/               # API Gateway service
│   ├── auth-service/              # Service xác thực
│   ├── blog-service/              # Service bài viết
│   ├── bookings-service/          # Service đặt tour
│   ├── chat-service/              # Service chat
│   ├── frontend/                  # Frontend Web UI
│   ├── reviews-service/           # Service đánh giá
│   ├── tours-service/             # Service quản lý tour
│   └── users-service/             # Service quản lý user
│
├── overlays/                      # Cấu hình ghi đè (patches) theo môi trường
│   ├── dev/
│   │   ├── kustomization.yaml     # Cấu hình môi trường dev (namespace, image tags)
│   │   ├── ingress.yaml           # Ingress dev.uittravel.shop / api-dev...
│   │   └── patches/               # Các patch (replicas, mongo URL...)
│   └── prod/
│       ├── kustomization.yaml     # Cấu hình môi trường prod
│       ├── ingress.yaml           # Ingress prod.uittravel.shop / api-prod...
│       └── patches/               # Các patch cho prod
│
├── monitoring/                    # Cấu hình giám sát (Prometheus ServiceMonitors)
│   ├── app-servicemonitor-dev.yaml
│   ├── app-servicemonitor-prod.yaml
│   ├── argocd-servicemonitor.yaml
│   ├── kustomization.yaml
│   └── namespace.yaml
│
└── argocd-metrics/                # Thư mục cho metrics (hiện tại trống)
```

*(Các file test mạng như `nettest2.json`, `patch-ctrl.yaml` ở thư mục gốc được dùng cho mục đích debug mạng và cấu hình vá lỗi cho ArgoCD nội bộ)*

---

## 🏗️ Kiến trúc triển khai với ArgoCD

Khi được triển khai trên ArgoCD, hệ thống hoạt động theo mô hình **GitOps Pull-based**:

```text
┌────────────────────────────────────────────────────────────────┐
│                         GITHUB REPOSITORY                      │
│              (minhnhatuit734/k8s-manifests @ main)             │
└───────────────────────────┬────────────────────────────────────┘
                            │  ArgoCD polls / watches for changes
                            │
            ┌───────────────▼────────────────┐
            │           ARGOCD               │
            │    (namespace: argocd)         │
            │                                │
            │  ┌──────────────────────────┐  │
            │  │  Application: kltn-dev   │  │
            │  │  source: overlays/dev    │  │
            │  │  dest namespace: dev     │  │
            │  └──────────┬───────────────┘  │
            │             │                  │
            │  ┌──────────▼───────────────┐  │
            │  │  Application: kltn-prod  │  │
            │  │  source: overlays/prod   │  │
            │  │  dest namespace: prod    │  │
            │  └──────────┬───────────────┘  │
            │             │                  │
            │  ┌──────────▼───────────────┐  │
            │  │  App: kltn-monitoring    │  │
            │  │  source: monitoring      │  │
            │  │  dest namespace: monitor │  │
            │  └──────────┬───────────────┘  │
            └─────────────┼──────────────────┘
                          │  kubectl apply (automated sync)
                          ▼
                 KUBERNETES CLUSTER
```

### Luồng CI/CD hoàn chỉnh

1. **Code changes:** Developer đẩy code lên repository chứa mã nguồn (Source Repo).
2. **CI Pipeline (GitHub Actions):** Tự động chạy Build, Test và Push Docker Image (với tag `dev-<run>-<sha>` hoặc `prod-<run>-<sha>`) lên Docker Hub.
3. **Cập nhật Manifests:** CI Pipeline sử dụng Kustomize để cập nhật image tag mới vào thư mục `overlays/dev/kustomization.yaml` hoặc `overlays/prod/kustomization.yaml` ở repository này (`k8s-manifests`), sau đó commit và push.
4. **ArgoCD Sync:** ArgoCD tự động phát hiện thay đổi trong Git và đồng bộ (sync) áp dụng trạng thái cấu hình mới nhất vào Kubernetes Cluster.

---

## 🧩 Các thành phần (Microservices)

| Service | Image trên Docker Hub | Mô tả |
|---|---|---|
| `frontend` | `mnhat1/frontend` | Giao diện Web Client |
| `api-gateway` | `mnhat1/api-gateway` | Cổng giao tiếp API duy nhất (Gateway) |
| `auth-service` | `mnhat1/auth-service` | Xác thực & Cấp phát JWT Token |
| `users-service` | `mnhat1/users-service` | Quản lý thông tin người dùng |
| `tours-service` | `mnhat1/tours-service` | Quản lý dữ liệu tour du lịch |
| `bookings-service`| `mnhat1/bookings-service`| Xử lý quy trình đặt tour và thanh toán |
| `reviews-service` | `mnhat1/reviews-service` | Xử lý tính năng đánh giá & nhận xét |
| `blog-service` | `mnhat1/blog-service` | Quản lý và cung cấp bài viết blog |
| `chat-service` | `mnhat1/chat-service` | Xử lý tin nhắn và chat thời gian thực |

*(Các service giao tiếp nội bộ thông qua các Kubernetes Service DNS. Bên ngoài chỉ truy cập qua Ingress)*

---

## 🌍 Môi trường triển khai

| Môi trường | Namespace | Web URL | API Gateway URL | Image tag pattern | Replicas |
|---|---|---|---|---|---|
| **Dev** | `dev` | `dev.uittravel.shop` | `api-dev.uittravel.shop` | `dev-<run>-<sha>` | 1 (Tiết kiệm) |
| **Prod** | `prod` | `prod.uittravel.shop` | `api-prod.uittravel.shop` | `prod-<run>-<sha>` | Scaled (Nhiều instances) |

---

## 🚀 Hướng dẫn cài đặt & Triển khai

### 1. Yêu cầu hệ thống
- **Kubernetes Cluster** ≥ 1.25
- **NGINX Ingress Controller** ≥ 1.8
- **cert-manager** ≥ 1.12 (để cấu hình tự động cấp phát chứng chỉ Let's Encrypt với `letsencrypt-prod`)
- **Prometheus Operator** (để hỗ trợ các `ServiceMonitor` resources trong thư mục `monitoring`)

### 2. Cài đặt ArgoCD vào Cluster
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Khởi tạo Application Secrets
Trước khi sync ứng dụng qua ArgoCD, bạn phải tạo thủ công các secret (vì lý do bảo mật, không lưu thông tin nhạy cảm vào Git):

```bash
# Tạo secret cho môi trường Dev
kubectl create secret generic kltn-app-secrets \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  --from-literal=MONGO_URL=<your-mongo-url> \
  -n dev

# Tạo secret cho môi trường Prod
kubectl create secret generic kltn-app-secrets \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  --from-literal=MONGO_URL=<your-mongo-url> \
  -n prod
```

### 4. Đăng ký các ứng dụng ArgoCD
Apply các tệp cấu hình Application để ArgoCD bắt đầu quản lý:
```bash
kubectl apply -f argocd/kltn-dev-app.yaml
kubectl apply -f argocd/kltn-prod-app.yaml
kubectl apply -f argocd/kltn-monitoring-app.yaml
```

Kiểm tra trạng thái triển khai:
```bash
argocd app list
argocd app get kltn-dev
```

---

## 🔗 Liên kết liên quan

- **App Source Repo:** Chứa source code của các microservices và luồng CI pipeline (GitHub Actions).
- **Terraform Repo:** Chứa cấu hình hạ tầng Infrastructure as Code (Kubernetes cluster, node pools, networking).
- **[Docker Hub (`mnhat1`)](https://hub.docker.com/u/mnhat1):** Nơi lưu trữ public các container images của ứng dụng.