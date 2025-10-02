# HwameiStor 项目概览

## 项目简介

**HwameiStor** 是一个 Kubernetes 原生的高可用本地存储系统，专为有状态工作负载设计。它基于 CSI 架构，创建本地存储资源池，管理各种类型的磁盘（HDD、SSD、NVMe），并通过本地卷提供分布式存储服务。

- **项目类型**: CNCF 沙箱项目
- **开发语言**: Go 1.21
- **Kubernetes 兼容性**: 1.24-1.30
- **代码仓库**: github.com/hwameistor/hwameistor

---

## 模块架构总览

### 核心模块与职责

| 目录 | 主要职责 | 关键文件 |
|------|---------|---------|
| [cmd/local-disk-manager](../../cmd/local-disk-manager/) | LDM 模块入口 | `diskmanager.go` |
| [pkg/local-disk-manager](../../pkg/local-disk-manager/) | 物理磁盘管理、SMART 监控、磁盘 CSI 驱动 | `controller/`, `member/`, `csi/`, `disk/`, `smart/`, `udev/` |
| [cmd/local-storage](../../cmd/local-storage/) | LS 模块入口 | `storage.go` |
| [pkg/local-storage](../../pkg/local-storage/) | LVM 卷管理、HA 卷（DRBD）、卷生命周期管理 | `controller/`, `member/controller/` |
| [cmd/scheduler](../../cmd/scheduler/) | 调度器入口 | `scheduler.go` |
| [pkg/scheduler](../../pkg/scheduler/) | Kubernetes 调度扩展，基于卷位置和容量调度 Pod | `scheduler/` |
| [cmd/admission](../../cmd/admission/) | 准入控制器入口 | `admission.go`, `app/mutate.go` |
| [pkg/webhook](../../pkg/webhook/) | Mutating Webhook，自动设置调度器名称 | - |
| [cmd/evictor](../../cmd/evictor/) | 驱逐控制器入口 | `main.go` |
| [pkg/evictor](../../pkg/evictor/) | 节点/Pod 驱逐场景下的卷副本迁移 | - |
| [cmd/exporter](../../cmd/exporter/) | 指标导出器入口 | `main.go` |
| [pkg/exporter](../../pkg/exporter/) | Prometheus 指标导出（节点、存储池、卷、磁盘） | - |
| [cmd/apiserver](../../cmd/apiserver/) | API 服务器入口 | `main.go` |
| [pkg/apiserver](../../pkg/apiserver/) | REST API 和 Swagger 文档（Gin 框架） | `api/`, `controller/`, `router/` |
| [cmd/failover-assistant](../../cmd/failover-assistant/) | 故障转移助手入口 | `main.go` |
| [pkg/failover-assistant](../../pkg/failover-assistant/) | 应用自动故障转移到健康节点 | - |
| [cmd/auditor](../../cmd/auditor/) | 审计器入口 | `main.go` |
| [pkg/auditor](../../pkg/auditor/) | 资源历史记录跟踪 | - |
| [cmd/pvc-autoresizer](../../cmd/pvc-autoresizer/) | PVC 自动扩容器入口 | `main.go` |
| [pkg/pvc-autoresizer](../../pkg/pvc-autoresizer/) | 基于策略自动扩展卷 | - |
| [cmd/local-disk-action-controller](../../cmd/local-disk-action-controller/) | 磁盘动作控制器入口 | `main.go` |
| [pkg/local-disk-action-controller](../../pkg/local-disk-action-controller/) | 处理磁盘操作和动作 | - |
| [cmd/hwameictl](../../cmd/hwameictl/) | CLI 工具入口 | `hwameictl.go` |
| [pkg/hwameictl](../../pkg/hwameictl/) | 命令行管理工具 | - |

### API 与公共模块

| 目录 | 主要职责 | 关键文件 |
|------|---------|---------|
| [pkg/apis/hwameistor/v1alpha1](../../pkg/apis/hwameistor/v1alpha1/) | CRD 类型定义 | `localdisk*.go`, `localvolume*.go`, `localstorage*.go` |
| [pkg/apis/client](../../pkg/apis/client/) | 生成的 clientsets、listers、informers | `clientset/`, `listers/`, `informers/` |
| [pkg/utils](../../pkg/utils/) | 共享工具库 | - |
| [pkg/exechelper](../../pkg/exechelper/) | 命令执行辅助工具 | - |
| [pkg/common](../../pkg/common/) | 公共定义和常量 | - |

### 部署与构建

| 目录 | 主要职责 | 关键文件 |
|------|---------|---------|
| [build/](../../build/) | 各模块的 Dockerfile | `local-disk-manager/Dockerfile`, `local-storage/Dockerfile` 等 |
| [deploy/crds](../../deploy/crds/) | CRD YAML 定义文件 | `*.yaml` |
| [helm/hwameistor](../../helm/hwameistor/) | Helm Chart | `values.yaml`, `templates/` |
| [test/e2e](../../test/e2e/) | E2E 测试套件 | - |

---

## 构建与运行方式

### 编译

#### 编译所有模块
```bash
make compile
```

#### 编译特定模块
```bash
make compile_ldm          # Local Disk Manager
make compile_ls           # Local Storage
make compile_scheduler    # Scheduler
make compile_admission    # Admission Controller
make compile_evictor      # Evictor
make compile_exporter     # Exporter
make compile_apiserver    # API Server
make compile_failover     # Failover Assistant
make compile_auditor      # Auditor
make compile_pvc-autoresizer  # PVC Auto Resizer
make compile_lda          # Local Disk Action Controller
```

### Docker 镜像构建

#### 构建所有镜像 (amd64)
```bash
make image
```

#### 构建 ARM64 镜像
```bash
make arm-image
```

#### 构建特定模块镜像
```bash
make build_ldm_image
make build_ls_image
make build_scheduler_image
# ... 其他模块
```

### 测试

```bash
make unit-test          # 单元测试
make e2e-test           # E2E 测试
make pr-test            # PR 测试
make adaptation_test    # 适配测试
make relok8s-test       # Relok8s 测试
```

### 代码质量

```bash
make lint               # 运行 linter
make lint-fix           # 自动修复
```

### API 生成

```bash
make apis               # 重新生成 CRD 和 clients
```

### CLI 工具构建

```bash
make build_hwameictl OS=linux ARCH=amd64
make build_hwameictl OS=darwin ARCH=arm64
make build_hwameictl OS=windows ARCH=amd64
```

### 本地运行 API Server

```bash
make apiserver_run      # 启动 http://127.0.0.1/swagger/index.html
```

### 依赖管理

```bash
make vendor             # 更新 vendor 目录
```

---

## 外部依赖

### Go 依赖

#### Kubernetes 生态
- `k8s.io/api` v0.26.0
- `k8s.io/client-go` v12.0.0+
- `k8s.io/apimachinery` v0.26.0
- `sigs.k8s.io/controller-runtime` v0.9.2
- `k8s.io/kubernetes` v1.24.0

#### CSI 相关
- `github.com/container-storage-interface/spec` v1.5.0
- `github.com/kubernetes-csi/csi-lib-utils` v0.11.0

#### Web 框架
- `github.com/gin-gonic/gin` v1.8.1 (API Server)
- `github.com/gorilla/mux` v1.8.0

#### gRPC
- `google.golang.org/grpc` v1.40.0

#### Operator 框架
- `github.com/operator-framework/operator-sdk` v0.18.2
- `github.com/hwameistor/hwameistor-operator` v0.10.7

#### 系统交互
- `github.com/pilebones/go-udev` v0.9.0 (udev 事件处理)
- `k8s.io/mount-utils` v0.29.8 (挂载工具)
- `github.com/containerd/cgroups/v3` v3.0.2

#### 日志与测试
- `github.com/sirupsen/logrus` v1.9.3
- `github.com/onsi/ginkgo/v2` v2.1.4
- `github.com/stretchr/testify` v1.8.4

### 外部工具依赖

- **Docker/Buildx**: 多架构镜像构建
- **Helm**: Kubernetes 部署
- **golangci-lint**: 代码质量检查
- **Swagger (swag)**: API 文档生成
- **code-generator**: Kubernetes 客户端代码生成

### 运行时依赖

- **LVM**: 逻辑卷管理
- **DRBD**: 高可用卷复制（可选）
- **Prometheus**: 指标采集（可选）
- **udev**: 磁盘事件监听

---

## 新手阅读顺序建议

### 第一阶段：理解整体架构（1-2天）

1. **项目文档**
   - 阅读本文档（概览）
   - [CLAUDE.md](../../CLAUDE.md) - 开发指南
   - [README.md](../../README.md) - 项目介绍

2. **CRD 定义**（理解数据模型）
   - [pkg/apis/hwameistor/v1alpha1/localdisk_types.go](../../pkg/apis/hwameistor/v1alpha1/)
   - [pkg/apis/hwameistor/v1alpha1/localvolume_types.go](../../pkg/apis/hwameistor/v1alpha1/)
   - [pkg/apis/hwameistor/v1alpha1/localstoragenode_types.go](../../pkg/apis/hwameistor/v1alpha1/)

3. **Makefile 与构建流程**
   - [Makefile](../../Makefile) - 理解构建和编译流程
   - [build/](../../build/) - 查看各模块 Dockerfile

### 第二阶段：核心模块深入（3-5天）

4. **Local Disk Manager (LDM)** - 磁盘管理基础
   - 入口: [cmd/local-disk-manager/diskmanager.go](../../cmd/local-disk-manager/diskmanager.go)
   - 磁盘检测: [pkg/local-disk-manager/disk/manager.go](../../pkg/local-disk-manager/disk/)
   - 控制器: [pkg/local-disk-manager/controller/](../../pkg/local-disk-manager/controller/)
   - CSI 驱动: [pkg/local-disk-manager/csi/](../../pkg/local-disk-manager/csi/)

5. **Local Storage (LS)** - 存储核心
   - 入口: [cmd/local-storage/storage.go](../../cmd/local-storage/storage.go)
   - 卷控制器: [pkg/local-storage/controller/](../../pkg/local-storage/controller/)
   - Member 控制器: [pkg/local-storage/member/controller/](../../pkg/local-storage/member/controller/)

6. **Scheduler** - 调度逻辑
   - 入口: [cmd/scheduler/scheduler.go](../../cmd/scheduler/scheduler.go)
   - 调度器: [pkg/scheduler/scheduler/](../../pkg/scheduler/scheduler/)

### 第三阶段：周边模块（2-3天）

7. **Admission Controller**
   - 入口: [cmd/admission/admission.go](../../cmd/admission/admission.go)
   - Webhook: [pkg/webhook/](../../pkg/webhook/)

8. **API Server**
   - 入口: [cmd/apiserver/main.go](../../cmd/apiserver/main.go)
   - API: [pkg/apiserver/api/](../../pkg/apiserver/api/)
   - 路由: [pkg/apiserver/router/](../../pkg/apiserver/router/)

9. **Evictor & Failover**
   - Evictor: [pkg/evictor/](../../pkg/evictor/)
   - Failover: [pkg/failover-assistant/](../../pkg/failover-assistant/)

### 第四阶段：实践与测试（持续）

10. **本地开发环境搭建**
    ```bash
    # 克隆代码
    git clone https://github.com/hwameistor/hwameistor.git
    cd hwameistor

    # 安装依赖
    make vendor

    # 编译模块
    make compile_ldm
    make compile_ls

    # 运行单元测试
    make unit-test
    ```

11. **E2E 测试**
    - [test/e2e/](../../test/e2e/) - 端到端测试用例

12. **Helm 部署**
    - [helm/hwameistor/](../../helm/hwameistor/) - Chart 配置

### 关键学习路径

```
CRD 定义 → LDM (磁盘管理) → LS (卷管理) → Scheduler (调度)
    ↓
Admission Controller → API Server → CLI 工具
    ↓
Evictor/Failover (高级特性)
```

---

## 核心概念速查

| 概念 | 说明 |
|------|------|
| **Volume Scheduling** | 两阶段调度：HwameiStor Scheduler + K8s Scheduler，将 Pod 调度到数据附近 |
| **HA Volumes** | 使用 DRBD 跨节点复制实现高可用 |
| **Member/Controller Pattern** | Member 组件运行在每个节点（DaemonSet），Controller 集中管理 |
| **LocalDisk** | 物理磁盘 CR |
| **LocalVolume** | 逻辑卷 CR |
| **LocalStorageNode** | 存储节点状态 CR |
| **CSI Driver** | LDM 提供裸盘 CSI，LS 提供 LVM CSI |

---

## 开发注意事项

### 修改 CRD 后的操作

1. 修改 `pkg/apis/hwameistor/v1alpha1/` 中的类型定义
2. 运行 `make apis` 重新生成：
   - CRD YAML 文件
   - Clientsets, Listers, Informers
   - DeepCopy 方法

### 镜像标签

- 开发环境: `IMAGE_TAG=99.9-dev` (默认)
- 发布版本: `RELEASE_TAG` 来自 git tags (例如 `v1.0.0`)
- 自定义: `IMAGE_TAG=<tag>` 或 `RELEASE_TAG=<tag>`

### 单元测试排除

以下包因集成测试性质被排除在单元测试外：
- `./pkg/local-storage/member`
- `./pkg/scheduler`
- `./pkg/evictor`
- `./pkg/apiserver`

---

## 参考资源

- **官方文档**: [hwameistor.io](https://hwameistor.io)
- **GitHub**: [github.com/hwameistor/hwameistor](https://github.com/hwameistor/hwameistor)
- **CNCF Sandbox**: [CNCF Projects](https://www.cncf.io/projects/)

---

*文档生成时间: 2025-10-02*
*适用版本: v1.0.0+*
