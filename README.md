# Argo Rollouts Gateway API Plugin

**Author:** Phạm Lê Ngọc Sơn  
**License:** Apache 2.0  
**Language:** Go 1.23+

---

## Giới thiệu

**Argo Rollouts Gateway API Plugin** là một plugin mở rộng cho [Argo Rollouts](https://argoproj.github.io/rollouts/) — bộ điều khiển triển khai tiến tiến (progressive delivery) trên Kubernetes. Plugin này cho phép tích hợp [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) vào quy trình triển khai Canary và Blue/Green, giúp quản lý lưu lượng truy cập (traffic routing) một cách tự động, linh hoạt và chính xác.

Dự án được phát triển và bảo trì bởi **Phạm Lê Ngọc Sơn**.

---

## Tính năng chính

### 1. Hỗ trợ đa loại Route
- **HTTPRoute** — Quản lý lưu lượng HTTP/HTTPS, hỗ trợ chia tải theo trọng số giữa canary và stable service.
- **GRPCRoute** — Quản lý lưu lượng gRPC cho các ứng dụng microservice hiệu năng cao.
- **TCPRoute** — Quản lý lưu lượng TCP cho các dịch vụ không phải HTTP.

### 2. Header-Based Routing
Hỗ trợ định tuyến dựa trên HTTP header, cho phép triển khai A/B testing bằng cách chuyển hướng request có header cụ thể sang phiên bản canary.

### 3. Quản lý nhiều Route đồng thời
Khả năng quản lý nhiều route cùng lúc trong một lần triển khai rollout, phù hợp với các hệ thống phức tạp có nhiều endpoint.

### 4. Tích hợp Argo Rollouts Experiments
Hỗ trợ đầy đủ tính năng [Experiments](https://argo-rollouts.readthedocs.io/en/stable/features/experiment/) của Argo Rollouts, cho phép chạy các thí nghiệm triển khai song song.

### 5. Tương thích đa nhà cung cấp Gateway API
Plugin hoạt động với bất kỳ nhà cung cấp nào triển khai chuẩn Kubernetes Gateway API, bao gồm:

| Nhà cung cấp | Trạng thái |
|---|---|
| Traefik | Đã kiểm thử đầy đủ |
| Envoy Gateway | Đã kiểm thử đầy đủ |
| Kong | Đã kiểm thử đầy đủ |
| Cilium | Đã kiểm thử đầy đủ |
| Linkerd | Đã kiểm thử đầy đủ |
| NGINX Gateway Fabric | Đã kiểm thử đầy đủ |
| Google Cloud (GKE) | Đã kiểm thử đầy đủ |
| Gloo Gateway | Đã kiểm thử đầy đủ |

---

## Kiến trúc hệ thống

```
┌─────────────────────────────────────────────────────┐
│                 Argo Rollouts Controller             │
│                                                     │
│   ┌─────────────┐    RPC    ┌────────────────────┐  │
│   │  Rollout     │◄────────►│  Gateway API       │  │
│   │  Controller  │          │  Plugin (RPC)      │  │
│   └─────────────┘          └────────┬───────────┘  │
│                                      │              │
└──────────────────────────────────────┼──────────────┘
                                       │
                                       ▼
                          ┌────────────────────────┐
                          │  Kubernetes Gateway API │
                          │  (HTTPRoute / GRPCRoute │
                          │   / TCPRoute)           │
                          └────────────┬───────────┘
                                       │
                          ┌────────────▼───────────┐
                          │  Gateway API Provider   │
                          │  (Traefik, Envoy, Kong, │
                          │   Cilium, NGINX, ...)   │
                          └────────────────────────┘
```

Plugin hoạt động như một RPC server riêng biệt, được Argo Rollouts Controller tải và giao tiếp qua giao thức RPC (sử dụng HashiCorp go-plugin). Khi phát hiện sự kiện rollout (ví dụ: `SetWeight`, `SetHeaderRoute`), controller sẽ gọi plugin để cập nhật cấu hình route trên Kubernetes Gateway API.

---

## Yêu cầu hệ thống

- **Kubernetes** >= 1.26
- **Argo Rollouts** >= 1.6.0
- **Gateway API CRDs** >= 1.1.0
- Một Gateway API provider đã được cài đặt trên cluster (Traefik, Envoy Gateway, Kong, v.v.)
- **Go** >= 1.23.0 (nếu build từ source)

---

## Cài đặt

### Cách 1: Sử dụng binary có sẵn

Tải binary phù hợp với hệ điều hành từ [trang Releases](https://github.com/phamlengocson/rollouts-plugin-trafficrouter-gatewayapi/releases):

- `gatewayapi-plugin-linux-amd64`
- `gatewayapi-plugin-linux-arm64`
- `gatewayapi-plugin-darwin-amd64`
- `gatewayapi-plugin-darwin-arm64`
- `gatewayapi-plugin-windows-amd64.exe`

### Cách 2: Build từ source code

```bash
git clone https://github.com/phamlengocson/rollouts-plugin-trafficrouter-gatewayapi.git
cd rollouts-plugin-trafficrouter-gatewayapi
make local-build
```

### Cách 3: Sử dụng Docker

```bash
docker build -t gatewayapi-plugin:latest .
```

### Cấu hình Argo Rollouts

Sau khi có binary plugin, cấu hình ConfigMap `argo-rollouts-config` để Argo Rollouts tải plugin:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config
  namespace: argo-rollouts
data:
  trafficRouterPlugins: |
    - name: "phamlengocson/gatewayAPI"
      location: "https://github.com/phamlengocson/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.6.0/gatewayapi-plugin-linux-amd64"
```

---

## Hướng dẫn sử dụng nhanh

### Bước 1: Cài đặt Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/experimental-install.yaml
```

### Bước 2: Tạo Gateway và HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argo-rollouts-http-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - backendRefs:
        - name: stable-service
          port: 80
          weight: 100
        - name: canary-service
          port: 80
          weight: 0
```

### Bước 3: Tạo Rollout resource

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-rollout
spec:
  strategy:
    canary:
      canaryService: canary-service
      stableService: stable-service
      trafficRouting:
        plugins:
          phamlengocson/gatewayAPI:
            httpRoutes:
              - name: argo-rollouts-http-route
            namespace: default
      steps:
        - setWeight: 20
        - pause: { duration: 30s }
        - setWeight: 50
        - pause: { duration: 30s }
        - setWeight: 80
        - pause: { duration: 30s }
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          ports:
            - containerPort: 80
```

---

## Cấu trúc dự án

```
.
├── .github/                    # GitHub Actions CI/CD workflows
│   ├── ISSUE_TEMPLATE/         # Template cho bug report
│   └── workflows/
│       ├── ci.yaml             # CI pipeline (lint, unit test, e2e test)
│       └── release.yaml        # Release automation pipeline
├── docs/                       # Tài liệu dự án (MkDocs)
│   ├── assets/                 # Logo, CSS, JS
│   ├── features/               # Tài liệu tính năng nâng cao
│   ├── images/                 # Hình ảnh minh họa
│   ├── CONTRIBUTING.md         # Hướng dẫn đóng góp
│   ├── index.md                # Trang chủ tài liệu
│   ├── installation.md         # Hướng dẫn cài đặt
│   ├── provider-status.md      # Trạng thái hỗ trợ provider
│   └── quick-start.md          # Hướng dẫn nhanh
├── examples/                   # Ví dụ triển khai cho từng provider
│   ├── cilium/
│   ├── envoygateway/
│   ├── gloo-gateway/
│   ├── google-cloud/
│   ├── grpcroute/
│   ├── kong/
│   ├── linkerd/
│   ├── nginx/
│   └── traefik/
├── internal/                   # Package nội bộ
│   ├── defaults/               # Giá trị mặc định
│   └── utils/                  # Tiện ích chung (logging, types)
├── pkg/                        # Package chính
│   ├── mocks/                  # Mock objects cho testing
│   └── plugin/                 # Logic plugin chính
│       ├── errors.go           # Xử lý lỗi
│       ├── experiment.go       # Hỗ trợ Experiments
│       ├── grpcroute.go        # Quản lý GRPCRoute
│       ├── httproute.go        # Quản lý HTTPRoute
│       ├── plugin.go           # Interface và entry point plugin
│       ├── tcproute.go         # Quản lý TCPRoute
│       └── types.go            # Khai báo kiểu dữ liệu
├── test/                       # Bộ kiểm thử
│   ├── cluster-setup/          # Cấu hình cluster cho e2e test
│   └── e2e/                    # End-to-end tests
├── Dockerfile                  # Build container image
├── go.mod                      # Go module dependencies
├── go.sum                      # Checksum dependencies
├── LICENCE                     # Apache 2.0 License
├── main.go                     # Entry point ứng dụng
├── Makefile                    # Build automation
├── mkdocs.yml                  # Cấu hình trang tài liệu
├── README.md                   # File này
└── RELEASE_NOTES.md            # Ghi chú phát hành
```

---

## Build và Phát triển

### Build cục bộ (development)

```bash
make local-build
```

### Build release cho đa nền tảng

```bash
make release
```

Tạo binary cho:
- Linux (amd64, arm64)
- macOS (amd64, arm64)
- Windows (amd64)

### Chạy lint

```bash
make lint
```

### Chạy Unit Tests

```bash
make unit-tests
```

### Chạy End-to-End Tests

```bash
make e2e-tests
```

Lệnh này sẽ tự động:
1. Tạo cluster Kubernetes cục bộ bằng Kind
2. Cài đặt Gateway API CRDs, Argo Rollouts, và Traefik
3. Chạy bộ test e2e
4. Dọn dẹp cluster sau khi hoàn thành

Để giữ lại cluster sau test:

```bash
make CLUSTER_DELETE=false e2e-tests
```

---

## Công nghệ sử dụng

| Công nghệ | Mục đích |
|---|---|
| **Go 1.23** | Ngôn ngữ lập trình chính |
| **Kubernetes Gateway API v1.1.0** | Chuẩn giao tiếp routing |
| **Argo Rollouts v1.6.6** | Bộ điều khiển progressive delivery |
| **HashiCorp go-plugin** | Giao tiếp RPC giữa controller và plugin |
| **logrus** | Structured logging |
| **testify** | Unit testing framework |
| **e2e-framework** | End-to-end testing framework |
| **Kind** | Local Kubernetes cluster cho testing |
| **MkDocs Material** | Trang tài liệu |
| **GitHub Actions** | CI/CD pipeline |

---

## Quy trình phát hành

1. Cập nhật nội dung `RELEASE_NOTES.md`
2. Tạo tag release:
   ```bash
   git tag release-v1.0.0
   git push origin release-v1.0.0
   ```
3. GitHub Actions tự động build và publish release với binary cho tất cả nền tảng.

---

## Tài liệu chi tiết

Truy cập trang tài liệu đầy đủ tại: [https://phamlengocson.github.io/rollouts-plugin-trafficrouter-gatewayapi/](https://phamlengocson.github.io/rollouts-plugin-trafficrouter-gatewayapi/)

Các tài liệu có sẵn:

- [Cài đặt](docs/installation.md)
- [Hướng dẫn nhanh](docs/quick-start.md)
- [Trạng thái Provider](docs/provider-status.md)
- [Advanced Deployments](docs/features/advanced-deployments.md)
- [Multiple Routes](docs/features/multiple-routes.md)
- [Header-Based Routing](docs/features/header-based-routing.md)
- [TCP Routing](docs/features/tcp.md)
- [gRPC Routing](docs/features/grpc.md)
- [Đóng góp](docs/CONTRIBUTING.md)

---

## Ví dụ triển khai

Thư mục `examples/` chứa cấu hình mẫu cho từng Gateway API provider:

- **Traefik** — `examples/traefik/`
- **Traefik (Header-Based)** — `examples/traefik-header-based/`
- **Envoy Gateway** — `examples/envoygateway/`
- **Kong** — `examples/kong/`
- **Cilium** — `examples/cilium/`
- **Linkerd** — `examples/linkerd/`
- **Linkerd (Header-Based)** — `examples/linkerd-header-based/`
- **NGINX** — `examples/nginx/`
- **Google Cloud** — `examples/google-cloud/`
- **Gloo Gateway** — `examples/gloo-gateway/`
- **gRPC Route** — `examples/grpcroute/`

---

## Giấy phép

Dự án được phân phối dưới giấy phép **Apache License 2.0**. Xem file [LICENCE](LICENCE) để biết chi tiết.

```
Copyright 2024 Phạm Lê Ngọc Sơn

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
```

---

## Tác giả

**Phạm Lê Ngọc Sơn**

- GitHub: [https://github.com/phamlengocson](https://github.com/phamlengocson)
- Email: phamlengocson@gmail.com

---

> Dự án này được xây dựng dựa trên nền tảng Argo Rollouts plugin system và Kubernetes Gateway API, nhằm mục đích cung cấp giải pháp progressive delivery mạnh mẽ, linh hoạt và tương thích đa nền tảng.
