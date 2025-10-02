# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

HwameiStor is a Kubernetes-native high-availability local storage system for stateful workloads. It creates local storage resource pools, manages disks (HDD, SSD, NVMe), and provides distributed services with local volumes using the CSI architecture. This is a CNCF sandbox project written in Go.

## Build Commands

### Compilation
```bash
# Compile all modules
make compile

# Compile specific modules
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

### Docker Images
```bash
# Build all images (amd64)
make image

# Build all ARM64 images
make arm-image

# Build specific module images
make build_ldm_image
make build_ls_image
make build_scheduler_image
# ... etc for other modules
```

### Testing
```bash
# Run unit tests
make unit-test

# Run E2E tests
make e2e-test

# Run PR tests
make pr-test

# Run adaptation tests
make adaptation_test

# Run relok8s tests
make relok8s-test
```

### Linting
```bash
# Run linter
make lint

# Run linter with auto-fix
make lint-fix
```

### API Generation
```bash
# Regenerate CRDs and clients
make apis
```

### Documentation
```bash
cd docs
npm run start                    # English docs
npm run start -- --locale cn     # Chinese docs
```

### CLI Tool
```bash
# Build hwameictl for specific OS/ARCH
make build_hwameictl OS=linux ARCH=amd64
make build_hwameictl OS=darwin ARCH=arm64
make build_hwameictl OS=windows ARCH=amd64
```

### Vendor Management
```bash
make vendor
```

## Architecture

### Module Overview

HwameiStor consists of multiple independent modules that work together:

1. **local-disk-manager (LDM)** - `pkg/local-disk-manager/`, `cmd/local-disk-manager/`
   - Manages physical disks on nodes
   - Detects, monitors, and provisions disks
   - Provides disk health checks and SMART monitoring
   - CSI driver for raw disk volumes

2. **local-storage (LS)** - `pkg/local-storage/`, `cmd/local-storage/`
   - Core storage provisioning using LVM
   - Volume lifecycle management (create, expand, convert, migrate)
   - HA volumes using DRBD
   - Member controller runs on each node

3. **scheduler** - `pkg/scheduler/`, `cmd/scheduler/`
   - Kubernetes scheduler extender
   - Places pods on nodes with available HwameiStor volumes
   - Considers volume locality and capacity

4. **admission-controller** - `pkg/webhook/`, `cmd/admission/`
   - Mutating webhook
   - Automatically sets `schedulerName: hwameistor-scheduler` for pods using HwameiStor volumes
   - Validates volume configurations

5. **evictor** - `pkg/evictor/`, `cmd/evictor/`
   - Handles node/pod eviction scenarios
   - Migrates volume replicas when nodes are cordoned or drained

6. **exporter** - `pkg/exporter/`, `cmd/exporter/`
   - Prometheus metrics exporter
   - Metrics for nodes, storage pools, volumes, disks

7. **apiserver** - `pkg/apiserver/`, `cmd/apiserver/`
   - REST API and Swagger documentation
   - HTTP interface for cluster management
   - Built with Gin framework

8. **failover-assistant** - `pkg/failover-assistant/`, `cmd/failover-assistant/`
   - Automated application failover to healthy nodes
   - Works with volume replicas

9. **auditor** - `pkg/auditor/`, `cmd/auditor/`
   - Tracks resource history (cluster, node, storage pool, volume)

10. **pvc-autoresizer** - `pkg/pvc-autoresizer/`, `cmd/pvc-autoresizer/`
    - Automatically expands volumes based on policies

11. **local-disk-action-controller** - `pkg/local-disk-action-controller/`, `cmd/local-disk-action-controller/`
    - Handles disk actions and operations

### Key Concepts

- **Volume Scheduling**: Two-phase scheduling where HwameiStor scheduler works with Kubernetes scheduler to place pods near their data
- **HA Volumes**: Use DRBD for replication across nodes
- **Member/Controller Pattern**: Each storage module has a "member" component running on each node (DaemonSet) and a "controller" component (usually part of the member)
- **Custom Resources**: Extensive use of CRDs in `pkg/apis/hwameistor/v1alpha1/`:
  - `LocalDisk`, `LocalDiskNode`, `LocalDiskClaim`, `LocalDiskVolume`
  - `LocalVolume`, `LocalVolumeReplica`, `LocalVolumeGroup`
  - `LocalVolumeMigrate`, `LocalVolumeConvert`, `LocalVolumeExpand`
  - `LocalVolumeSnapshot`, `LocalVolumeReplicaSnapshot`
  - `LocalStorageNode`
  - `ResizePolicy`

### Code Organization

```
cmd/                    # Entry points for each module
pkg/
  apis/                 # CRD definitions (v1alpha1)
    client/             # Generated clientsets
    hwameistor/         # API types
  local-disk-manager/   # LDM module
    controller/         # Reconcilers for disk CRDs
    member/             # Node-level disk operations
    csi/                # CSI driver implementation
    disk/               # Disk detection and management
    smart/              # SMART monitoring
    udev/               # udev event handling
  local-storage/        # LS module
    controller/         # Volume reconcilers
    member/             # Node-level volume operations
      controller/       # Member controller logic
  scheduler/            # Scheduler plugin
    scheduler/          # Main scheduling logic
  webhook/              # Admission controller
  apiserver/            # REST API
    api/, controller/, router/
  hwameictl/            # CLI tool
  utils/                # Shared utilities
build/                  # Dockerfiles for each module
deploy/
  crds/                 # CRD YAML files
helm/
  hwameistor/           # Helm chart
test/
  e2e/                  # E2E test suites
```

### CSI Architecture

- LDM provides CSI driver for raw disk volumes
- LS provides CSI driver for LVM volumes (with HA support via DRBD)
- Both implement CSI spec for provisioning, attaching, mounting

### Storage Flow

1. User creates PVC with HwameiStor StorageClass
2. LS CSI controller creates `LocalVolume` CR
3. LS member controller on selected node creates LVM volume
4. For HA volumes, DRBD replication is configured
5. Scheduler ensures pod lands on node with volume access
6. LS CSI node plugin mounts volume into pod

## Development Notes

### Updating Swagger Documentation
```bash
make apiserver_swag    # Regenerates API docs from code annotations
```

### Running API Server Locally
```bash
make apiserver_run     # Starts on http://127.0.0.1/swagger/index.html
```

### Builder Image
The project uses a builder container for consistent builds:
- Image: `ghcr.io/hwameistor/hwameistor-builder:latest`
- Contains Go toolchain, operator-sdk, code generators
- Used via `DOCKER_MAKE_CMD` in Makefile

### Multi-Architecture Support
- Builds support amd64 and arm64
- Use `DOCKER_BUILDX_CMD_*` variables for cross-platform builds
- Release process builds both architectures and creates multi-arch manifest

### Image Tagging
- Development: `IMAGE_TAG=99.9-dev` (default)
- Release: `RELEASE_TAG` derived from git tags (e.g., `v1.0.0`)
- Override with `IMAGE_TAG=<tag>` or `RELEASE_TAG=<tag>`

### Test Exclusions
Unit tests exclude these packages (due to integration test nature):
- `./pkg/local-storage/member`
- `./pkg/scheduler`
- `./pkg/evictor`
- `./pkg/apiserver`

### Code Generation
When modifying CRD types in `pkg/apis/hwameistor/v1alpha1/`:
1. Update the type definitions
2. Run `make apis` to regenerate:
   - CRD YAML files
   - Clientsets, listers, informers
   - Deep copy methods

### Versioning
- Go version: 1.21
- Kubernetes compatibility: 1.24-1.30
- Uses vendored dependencies (`-mod vendor`)

## Reply Guidelines
- Always reference **file path + function name** when explaining code.
- Use **Mermaid diagrams** for flows, call chains, and module dependencies.
- If context is missing, ask explicitly which files to `/add`.
- Never hallucinate non-existing functions or files.
- Always reply in **Chinese**

## Excluded Paths
- vendor/
- build/
- dist/
- .git/
- third_party/

## Glossary


## Run Instructions