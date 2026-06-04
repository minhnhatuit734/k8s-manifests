# 🚀 KLTN K8s Manifests — GitOps with ArgoCD

Kho lưu trữ này chứa toàn bộ **Kubernetes manifests** cho dự án KLTN (Khóa Luận Tốt Nghiệp) — một nền tảng du lịch trực tuyến (`uittravel.shop`). Các manifest được tổ chức theo mô hình **Kustomize Base/Overlay** và được triển khai tự động thông qua **ArgoCD** theo phương pháp **GitOps**.

---

## 📁 Cấu trúc thư mục

```
k8s-manifests/
├── argocd/                        # ArgoCD Application definitions
│   ├── kltn-dev-app.yaml          # App định nghĩa cho môi trường Dev
│   └── kltn-prod-app.yaml         # App định nghĩa cho môi trường Prod
│
├── base/                          # Manifests gốc (dùng chung cho mọi môi trường)
│   ├── kustomization.yaml         # Danh sách tất cả resources
│   ├── configmap.yaml             # ConfigMap chứa URL nội bộ các services
│   ├── api-gateway/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── auth-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── users-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── tours-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── bookings-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── reviews-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── blog-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── chat-service/
│       ├── deployment.yaml
│       └── service.yaml
│
└── overlays/                      # Cấu hình ghi đè theo môi trường
    ├── dev/
    │   ├── kustomization.yaml     # Image tags, labels, namespace: dev
    │   ├── ingress.yaml           # dev.uittravel.shop / api-dev.uittravel.shop
    │   └── patches/
    │       ├── replicas.yaml      # Số lượng replicas cho dev (= 1)
    │       └── mongo-url-per-service.yaml  # MongoDB URL riêng từng service
    └── prod/
        ├── kustomization.yaml     # Image tags, labels, namespace: prod
        ├── ingress.yaml           # prod.uittravel.shop / api-prod.uittravel.shop
        └── patches/
            ├── replicas.yaml      # Số lượng replicas cho prod
            └── mongo-url-per-service.yaml  # MongoDB URL riêng từng service
```

---

## 🏗️ Kiến trúc triển khai với ArgoCD

Khi được triển khai trên ArgoCD, hệ thống hoạt động theo mô hình **GitOps Pull-based**:

```
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
            └─────────────┼──────────────────┘
                          │  kubectl apply (automated sync)
                          │
┌─────────────────────────▼──────────────────────────────────────┐
│                    KUBERNETES CLUSTER                          │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  namespace: dev                         │  │
│  │                                                         │  │
│  │  [Ingress]  dev.uittravel.shop  →  frontend:80          │  │
│  │  [Ingress]  api-dev.uittravel.shop  →  api-gateway:80   │  │
│  │             (nginx + cert-manager + Let's Encrypt TLS)  │  │
│  │                                                         │  │
│  │  [Deployments × 1 replica each]                        │  │
│  │    frontend  │  api-gateway  │  auth-service            │  │
│  │    users-service  │  tours-service  │  bookings-service  │  │
│  │    reviews-service  │  blog-service  │  chat-service     │  │
│  │                                                         │  │
│  │  [ConfigMap: kltn-app-config]                          │  │
│  │  [Secret: kltn-app-secrets]                            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                  namespace: prod                        │  │
│  │                                                         │  │
│  │  [Ingress]  prod.uittravel.shop  →  frontend:80         │  │
│  │  [Ingress]  api-prod.uittravel.shop  →  api-gateway:80  │  │
│  │             (nginx + cert-manager + Let's Encrypt TLS)  │  │
│  │                                                         │  │
│  │  [Deployments — production replicas]                   │  │
│  │    frontend  │  api-gateway  │  auth-service            │  │
│  │    users-service  │  tours-service  │  bookings-service  │  │
│  │    reviews-service  │  blog-service  │  chat-service     │  │
│  │                                                         │  │
│  │  [ConfigMap: kltn-app-config]                          │  │
│  │  [Secret: kltn-app-secrets]                            │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Luồng CI/CD hoàn chỉnh

```
Developer pushes code
        │
        ▼
┌──────────────┐     Build & Test      ┌───────────────────┐
│  Source Repo │ ──────────────────►   │  CI Pipeline      │
│  (app code)  │                       │  (GitHub Actions) │
└──────────────┘                       └────────┬──────────┘
                                                │
                                    Docker build & push image
                                    mnhat1/<service>:<env>-<run>-<sha>
                                                │
                                                ▼
                                   ┌────────────────────────┐
                                   │  k8s-manifests repo    │
                                   │  (this repo)           │
                                   │                        │
                                   │  Update image tag in   │
                                   │  overlays/dev or       │
                                   │  overlays/prod         │
                                   └────────────┬───────────┘
                                                │  git commit & push
                                                ▼
                                   ┌────────────────────────┐
                                   │       ArgoCD           │
                                   │  Detects git diff      │
                                   │  Auto-syncs cluster    │
                                   └────────────────────────┘
```

---

## 🧩 Các thành phần (Microservices)

| Service | Image | Port | Mô tả |
|---|---|---|---|
| `frontend` | `mnhat1/frontend` | 80 | Giao diện người dùng |
| `api-gateway` | `mnhat1/api-gateway` | 4000 | API Gateway – điểm vào duy nhất |
| `auth-service` | `mnhat1/auth-service` | — | Xác thực & phân quyền |
| `users-service` | `mnhat1/users-service` | — | Quản lý người dùng |
| `tours-service` | `mnhat1/tours-service` | — | Quản lý tour du lịch |
| `bookings-service` | `mnhat1/bookings-service` | — | Đặt tour |
| `reviews-service` | `mnhat1/reviews-service` | — | Đánh giá & nhận xét |
| `blog-service` | `mnhat1/blog-service` | — | Bài viết & blog |
| `chat-service` | `mnhat1/chat-service` | — | Chat thời gian thực |

---

## 🌍 Môi trường

| Môi trường | Namespace | Frontend URL | API URL | Image tag pattern |
|---|---|---|---|---|
| **Dev** | `dev` | `dev.uittravel.shop` | `api-dev.uittravel.shop` | `dev-<run>-<sha>` |
| **Prod** | `prod` | `prod.uittravel.shop` | `api-prod.uittravel.shop` | `prod-<run>-<sha>` |

---

## ⚙️ Cấu hình Kustomize

### Base
- Định nghĩa toàn bộ Deployments và Services dùng chung.
- `configmap.yaml`: URL nội bộ các services (được inject qua `envFrom`).
- `kltn-app-secrets` (Secret): Phải được tạo thủ công trong cluster (chứa DB credentials, JWT secret, v.v.).

### Overlays
Mỗi overlay ghi đè các thông tin môi trường cụ thể:

| Phần | Dev | Prod |
|---|---|---|
| **Namespace** | `dev` | `prod` |
| **Labels** | `environment: dev` | `environment: prod` |
| **Image tags** | `dev-<run>-<sha>` | `prod-<run>-<sha>` |
| **Replicas** | 1 mỗi service | Theo cấu hình prod |
| **Ingress** | `dev.uittravel.shop` | `prod.uittravel.shop` |
| **MongoDB URL** | Per-service (patch) | Per-service (patch) |

---

## 🔄 ArgoCD Applications

### `kltn-dev` ([argocd/kltn-dev-app.yaml](./argocd/kltn-dev-app.yaml))

```yaml
source:
  repoURL: https://github.com/minhnhatuit734/k8s-manifests.git
  targetRevision: main
  path: overlays/dev
destination:
  namespace: dev
syncPolicy:
  automated:
    prune: true      # Xóa resources không còn trong Git
    selfHeal: true   # Tự phục hồi nếu cluster bị thay đổi ngoài Git
```

### `kltn-prod` ([argocd/kltn-prod-app.yaml](./argocd/kltn-prod-app.yaml))

```yaml
source:
  repoURL: https://github.com/minhnhatuit734/k8s-manifests.git
  targetRevision: main
  path: overlays/prod
destination:
  namespace: prod
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

---

## 🚀 Cách triển khai

### 1. Cài đặt ArgoCD vào cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Tạo Secret cho ứng dụng (bắt buộc trước khi sync)

```bash
# Tạo trong namespace dev
kubectl create secret generic kltn-app-secrets \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  --from-literal=MONGO_URL=<your-mongo-url> \
  -n dev

# Tạo trong namespace prod
kubectl create secret generic kltn-app-secrets \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  --from-literal=MONGO_URL=<your-mongo-url> \
  -n prod
```

### 3. Đăng ký ArgoCD Applications

```bash
# Apply ArgoCD Application manifests
kubectl apply -f argocd/kltn-dev-app.yaml
kubectl apply -f argocd/kltn-prod-app.yaml
```

### 4. Kiểm tra trạng thái

```bash
# Xem trạng thái sync
argocd app list

# Chi tiết một app
argocd app get kltn-dev
argocd app get kltn-prod

# Sync thủ công nếu cần
argocd app sync kltn-dev
```

---

## 🔒 TLS / HTTPS

Ingress sử dụng **cert-manager** với **Let's Encrypt** (`letsencrypt-prod` ClusterIssuer) để tự động cấp và gia hạn chứng chỉ SSL.

> **Yêu cầu**: cert-manager phải được cài đặt vào cluster trước khi deploy.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## 📋 Yêu cầu hệ thống

| Thành phần | Phiên bản khuyến nghị |
|---|---|
| Kubernetes | ≥ 1.25 |
| ArgoCD | ≥ 2.8 |
| Kustomize | ≥ 5.0 (tích hợp trong kubectl) |
| NGINX Ingress Controller | ≥ 1.8 |
| cert-manager | ≥ 1.12 |

---

## 🔗 Liên kết liên quan

- **App Source Repo**: Chứa source code các microservices và CI pipeline (GitHub Actions)
- **Terraform Repo**: Chứa cấu hình hạ tầng (Kubernetes cluster, node pools, networking)
- **Docker Hub**: `docker.io/mnhat1/` — nơi lưu trữ container images