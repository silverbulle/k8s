# 05b - GPU 与高性能互联

GPU 密集型工作负载（深度学习训练、HPC）不仅关注单卡算力，更依赖多卡/多节点间的高速通信。本章介绍 K8s 中 GPU 调度机制，以及 RDMA、数据并行（DP）、NCCL 等关键互联组件的原理与架构。

---

## 1. K8s GPU 调度

### Device Plugin 机制

K8s 原生不支持 GPU，通过 **Device Plugin** 扩展机制让第三方驱动向 kubelet 注册硬件资源。

```
┌─────────────────────────────────────────────────┐
│  kubelet                                        │
│                                                 │
│  ┌───────────────┐   gRPC    ┌───────────────┐  │
│  │  Device       │◄────────►│  NVIDIA       │  │
│  │  Plugin       │           │  Device       │  │
│  │  Manager      │           │  Plugin       │  │
│  └───────────────┘           │               │  │
│         │                    │ - List()      │  │
│         │ 上报资源           │ - Allocate()  │  │
│         ▼                    │ - PreStart()  │  │
│  Node.Status.Capacity        └───────────────┘  │
│    nvidia.com/gpu: 8                            │
└─────────────────────────────────────────────────┘
```

**Device Plugin 生命周期：**

| 阶段 | 操作 |
|------|------|
| **Register** | 插件启动，通过 Unix Socket 向 kubelet 注册（提供 endpoint + resource name） |
| **List** | kubelet 调用 `List()` 获取设备列表和健康状态 |
| **上报** | kubelet 将 `nvidia.com/gpu: 8` 写入 Node.Status，scheduler 可见 |
| **Allocate** | Pod 调度到该 Node 后，kubelet 调用 `Allocate()` 设置环境变量、挂载设备 |

### Pod 申请 GPU

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: training-job
spec:
  containers:
  - name: trainer
    image: pytorch:2.0
    resources:
      limits:
        nvidia.com/gpu: 2        # 申请 2 张 GPU
    env:
    - name: NVIDIA_VISIBLE_DEVICES
      value: "all"               # 由 Device Plugin Allocate() 自动设置
```

scheduler 看到 `nvidia.com/gpu: 2`，将其作为扩展资源（Extended Resource）匹配到有足够 GPU 的 Node。

### NVIDIA GPU Operator

生产环境通常部署 **GPU Operator**，一个 Operator 自动完成所有 GPU 相关组件的安装和管理。裸机安装 NVIDIA 全栈需要手动在每个节点配置驱动、运行时、插件等，GPU Operator 将整个流程 CRD 化、一键化。

#### 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  GPU Operator Controller (Deployment)                               │
│  监听 CRD: ClusterPolicy                                            │
│  根据 spec 自动创建和管理以下所有 DaemonSet                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │ Node Feature    │  │ NVIDIA Driver   │  │ Container          │  │
│  │ Discovery (NFD) │  │ (DaemonSet)     │  │ Toolkit (DaemonSet)│  │
│  │                 │  │                 │  │                    │  │
│  │ 检测硬件特性    │  │ 安装/升级 GPU   │  │ nvidia-container-  │  │
│  │ 标记 Node       │  │ 内核驱动        │  │ runtime            │  │
│  └────────┬────────┘  └────────┬────────┘  └─────────┬──────────┘  │
│           │                    │                      │             │
│           ▼                    ▼                      ▼             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │ Device Plugin   │  │ MIG Manager     │  │ DCGM Exporter      │  │
│  │ (DaemonSet)     │  │ (DaemonSet)     │  │ (DaemonSet)        │  │
│  │                 │  │                 │  │                    │  │
│  │ 向 kubelet      │  │ A100/H100       │  │ GPU 监控指标       │  │
│  │ 注册 GPU 资源   │  │ 多实例切分      │  │ 导出 Prometheus    │  │
│  └─────────────────┘  └─────────────────┘  └────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Network Operator（可选，独立 Operator）                       │   │
│  │                                                              │   │
│  │  ├── OFED Driver (DaemonSet)     — Mellanox 网卡驱动        │   │
│  │  ├── SR-IOV Device Plugin        — RDMA 网卡资源注册        │   │
│  │  ├── RDMA Shared Device Plugin   — 共享 RDMA 设备           │   │
│  │  └── Secondary Network (Multus)  — 多网络平面               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 各组件详解

**1. Node Feature Discovery (NFD)**

NFD 检测每个节点的硬件特性，以 Node Label 形式暴露，供后续组件和调度器使用。

```yaml
# NFD 自动添加的 Node Labels
metadata:
  labels:
    feature.node.kubernetes.io/pci-10de.present: "true"    # 10de = NVIDIA 厂商 ID
    feature.node.kubernetes.io/pci-15b3.present: "true"    # 15b3 = Mellanox 厂商 ID
    nvidia.com/gpu.present: "true"
    nvidia.com/gpu.count: "8"
    nvidia.com/gpu.product: "NVIDIA-H100-80GB-HBM3"
    nvidia.com/gpu.memory: "81920"
    nvidia.com/cuda.driver.major: "535"
```

**作用**：让 DaemonSet 的 nodeSelector 精确匹配有 GPU 的节点，避免在无 GPU 节点上浪费资源。调度器也可基于这些 label 做 GPU 型号亲和调度（比如只调度到 H100 节点）。

---

**2. NVIDIA Driver**

在宿主机上安装和管理 GPU 内核驱动（`nvidia.ko`）。这是整个 GPU 栈的基础——没有驱动，所有上层组件都无法工作。

```
安装方式:

方式 A — Pre-installed Driver
  宿主机已装好驱动，GPU Operator 只安装上层组件
  → 适合已有运维流程的环境

方式 B — Driver Container（推荐）
  GPU Operator 部署一个特权 DaemonSet，以容器方式安装驱动
  → 驱动版本由 Operator 统一管理，升级回滚方便

Driver Container DaemonSet:
  镜像: nvcr.io/nvidia/driver:535.104.05-ubuntu22.04
  权限: privileged（需要加载内核模块）
  挂载: /lib/modules（宿主机内核模块目录）
  
  启动时执行:
    1. 检查当前驱动版本
    2. 如需更新 → nvidia-installer 安装新驱动
    3. 加载 nvidia.ko, nvidia_uvm.ko, nvidia_drm.ko
    4. 创建 /dev/nvidia* 设备文件
```

**关键内核模块：**

| 模块 | 功能 |
|------|------|
| `nvidia.ko` | GPU 核心驱动，管理 GPU 设备、内存、调度 |
| `nvidia_uvm.ko` | Unified Virtual Memory，GPU/CPU 统一虚拟内存管理 |
| `nvidia_drm.ko` | Direct Rendering Manager，显示输出（训练场景通常不需要） |
| `nvidia_peermem.ko` | GPU Direct RDMA，允许网卡直接读写 GPU 显存 |
| `nvidia_fs.ko` | GPU Direct Storage，允许 NVMe 直接写入 GPU 显存 |

---

**3. NVIDIA Container Toolkit**

让容器运行时（containerd/Docker）感知 GPU 设备，在容器创建时自动配置 GPU 访问。

```
容器创建流程:

  kubectl run pod (nvidia.com/gpu: 2)
       │
       ▼
  kubelet 调用 containerd 创建容器
       │
       ▼
  containerd 识别 RuntimeClass: nvidia
       │ 调用 nvidia-container-runtime（替代默认的 runc）
       ▼
  nvidia-container-runtime:
    1. 读取 NVIDIA_VISIBLE_DEVICES 环境变量
       （由 Device Plugin Allocate() 设置，如 "0,1"）
    2. 调用 nvidia-container-cli 配置容器:
       - 将 /dev/nvidia0, /dev/nvidia1 设备挂载到容器
       - 将 CUDA 库文件 (libcuda.so, libcudart.so) 注入容器
       - 设置 NVIDIA_DRIVER_CAPABILITIES (compute, utility)
    3. 然后调用 runc 正常启动容器

结果: 容器内可以直接使用 nvidia-smi、CUDA、cuDNN
      且只能看到分配给它的 GPU（设备隔离）
```

```yaml
# RuntimeClass 定义（GPU Operator 自动创建）
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia          # 对应 containerd 中的 runtime handler
scheduling:
  nodeSelector:
    nvidia.com/gpu.present: "true"

# containerd 配置（GPU Operator 自动配置）
# /etc/containerd/config.toml:
#   [plugins.cri.containerd.runtimes.nvidia]
#     runtime_type = "io.containerd.runc.v2"
#     [plugins.cri.containerd.runtimes.nvidia.options]
#       BinaryName = "/usr/bin/nvidia-container-runtime"
```

**核心作用**：解决"容器内如何用 GPU"的问题。没有 Toolkit，即使宿主机有驱动、Device Plugin 分配了设备，容器也无法使用 GPU——因为容器默认是隔离的，看不到 `/dev/nvidia*` 和 CUDA 库。

---

**4. NVIDIA Device Plugin**

向 kubelet 注册 GPU 为可扩展资源（Extended Resource），处理 Pod 的 GPU 分配请求。

```
Device Plugin 完整生命周期:

  ┌──────────────────────────────────────────────────────────────┐
  │  nvidia-device-plugin DaemonSet                              │
  │  每个有 GPU 的 Node 运行一个 Pod                             │
  └──────────────┬───────────────────────────────────────────────┘
                 │
  ① Register    │  插件启动，通过 /var/lib/kubelet/device-plugins/ 下的
                 │  Unix Socket 向 kubelet 注册:
                 │    ResourceName: "nvidia.com/gpu"
                 │    Endpoint:     /var/lib/kubelet/device-plugins/nvidia.sock
                 │
  ② List        │  kubelet 调用 List():
                 │    → 返回 8 个设备 (GPU 0~7)，全部 Healthy
                 │
  ③ 上报        │  kubelet 更新 Node.Status:
                 │    Capacity:   nvidia.com/gpu: 8
                 │    Allocatable: nvidia.com/gpu: 8
                 │    → scheduler 可见，可以调度 GPU Pod 到此节点
                 │
  ④ Allocate    │  Pod 调度到此 Node，kubelet 调用 Allocate():
                 │    输入: 需要 2 个 GPU
                 │    操作:
                 │      - 选择 GPU 0, GPU 1（或根据拓扑选择最优组合）
                 │      - 设置环境变量:
                 │          NVIDIA_VISIBLE_DEVICES=0,1
                 │          NVIDIA_DRIVER_CAPABILITIES=compute,utility
                 │      - 挂载设备: /dev/nvidia0, /dev/nvidia1
                 │      - 挂载设备: /dev/nvidiactl, /dev/nvidia-uvm
                 │    输出: ContainerConfig（传给 Container Toolkit）
                 │
  ⑤ Health      │  持续监控设备健康状态:
                 │    - nvidia-smi 查询 ECC 错误、XID 错误
                 │    - GPU 掉卡 → 标记 Unhealthy → kubelet 上报
                 │    → scheduler 不再调度新 GPU Pod 到该节点
```

**GPU 拓扑感知调度**（Device Plugin 高级特性）：

```
8 GPU 节点示例:

  NVLink 域 0:  GPU 0, 1, 2, 3  (NVSwitch 全互联, 900 GB/s)
  NVLink 域 1:  GPU 4, 5, 6, 7  (NVSwitch 全互联, 900 GB/s)
  跨域通信:     PCIe (64 GB/s)

Device Plugin 拓扑感知:
  Pod 申请 4 GPU → 优先分配 GPU 0,1,2,3 (同 NVLink 域)
                    而不是 GPU 0,1,4,5 (跨域，通信慢 14 倍)

通过 GPU 拓扑注解暴露给 scheduler:
  nvidia.com/gpu.topology: "nvlink,nvlink,nvlink,pxb,pxb,pxb,pxb"
```

---

**5. DCGM Exporter**

DCGM（Data Center GPU Manager）Exporter 采集 GPU 运行时指标，以 Prometheus 格式暴露，供 Grafana 展示和告警。

```
DCGM Exporter DaemonSet
  │
  │ 采集 GPU 指标（每秒）
  │
  ├── 温度 & 功耗
  │   DCGM_FI_DEV_GPU_TEMP          GPU 核心温度 (°C)
  │   DCGM_FI_DEV_MEMORY_TEMP       显存温度 (°C)
  │   DCGM_FI_DEV_POWER_USAGE       功耗 (W)
  │   DCGM_FI_DEV_TOTAL_ENERGY      累计能耗 (mJ)
  │
  ├── 利用率
  │   DCGM_FI_DEV_GPU_UTIL          GPU 计算利用率 (%)
  │   DCGM_FI_DEV_MEM_COPY_UTIL     显存拷贝利用率 (%)
  │   DCGM_FI_DEV_ENC_UTIL          编码器利用率 (%)
  │   DCGM_FI_DEV_DEC_UTIL          解码器利用率 (%)
  │
  ├── 显存
  │   DCGM_FI_DEV_FB_USED           已用显存 (MB)
  │   DCGM_FI_DEV_FB_FREE           可用显存 (MB)
  │
  ├── ECC 错误（硬件故障预兆）
  │   DCGM_FI_DEV_ECC_SBE_VOL       单比特可纠正错误数
  │   DCGM_FI_DEV_ECC_DBE_VOL       双比特不可纠正错误数（严重!）
  │
  ├── XID 错误（驱动级异常）
  │   DCGM_FI_DEV_XID_ERRORS        XID 错误码
  │
  ├── PCIe & NVLink
  │   DCGM_FI_DEV_PCIE_TX_THROUGHPUT  PCIe 发送吞吐 (KB/s)
  │   DCGM_FI_DEV_PCIE_RX_THROUGHPUT  PCIe 接收吞吐 (KB/s)
  │   DCGM_FI_DEV_NVLINK_BANDWIDTH    NVLink 带宽 (bytes/s)
  │
  └── 进程级指标
      DCGM_FI_PROF_SM_OCCUPANCY      SM 占用率
      DCGM_FI_PROF_PIPE_TENSOR_ACTIVE Tensor Core 活跃率
      DCGM_FI_PROF_DRAM_ACTIVE        显存带宽利用率
```

**监控架构：**

```
Node: DCGM Exporter ──/metrics──► Prometheus ──► Grafana Dashboard
       (DaemonSet)                 (每 15s      (GPU 监控面板)
                                    scrape)
                                  │
                                  └──► AlertManager
                                        GPU 温度 > 85°C → 告警
                                        ECC DBE > 0 → 紧急告警（硬件故障）
                                        GPU 利用率 < 5% 持续 30min → 浪费告警
```

**典型运维场景：**

| 场景 | 指标 | 阈值 | 动作 |
|------|------|------|------|
| GPU 过热 | `GPU_TEMP` | > 85°C | 降频/停机检查散热 |
| 硬件故障 | `ECC_DBE_VOL` | > 0 | 立即替换 GPU |
| 显存泄漏 | `FB_USED` | 持续增长 | 检查训练脚本 |
| 训练卡死 | `GPU_UTIL` | = 0 持续 | 检查死锁/hang |
| NVLink 降级 | `NVLINK_BANDWIDTH` | 低于预期 | 检查物理连接 |

---

**6. MIG Manager**

MIG（Multi-Instance GPU）将一张物理 GPU 切分为多个独立的 GPU 实例，每个实例有独立的显存、计算单元和故障隔离。**仅支持 A100/H100/B200 架构。**

```
A100 80GB — MIG 切分方式:

  物理 GPU:
  ┌──────────────────────────────────────────┐
  │          A100 80GB (完整)                 │
  │  108 SM  │  80GB HBM  │  2039 GB/s      │
  └──────────────────────────────────────────┘

  切分方案 1 — 1g.10gb × 7:
  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐
  │ 1g   │ 1g   │ 1g   │ 1g   │ 1g   │ 1g   │ 1g   │
  │ 14SM │ 14SM │ 14SM │ 14SM │ 14SM │ 14SM │ 14SM │
  │ 10GB │ 10GB │ 10GB │ 10GB │ 10GB │ 10GB │ 10GB │
  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘
  → 7 个小实例，适合推理/小模型训练

  切分方案 2 — 3g.40gb × 2:
  ┌────────────────────┬────────────────────┐
  │       3g           │       3g           │
  │     42 SM          │     42 SM          │
  │     40GB           │     40GB           │
  └────────────────────┴────────────────────┘
  → 2 个中等实例，适合中等模型

  切分方案 3 — 7g.80gb × 1:
  ┌──────────────────────────────────────────┐
  │                 7g                       │
  │              108 SM                      │
  │              80GB                        │
  └──────────────────────────────────────────┘
  → 完整 GPU，不切分
```

```yaml
# MIG Manager 通过 ConfigMap 配置切分策略
apiVersion: v1
kind: ConfigMap
metadata:
  name: mig-parted-config
data:
  config.yaml: |
    version: v1
    mig-configs:
      all-disabled:        # 不切分
        - devices: all
          mig-enabled: false
      all-1g.10gb:         # 全部切为 1g.10gb
        - devices: all
          migenabled: true
          migdevices:
            1g.10gb: 7
      all-3g.40gb:         # 全部切为 3g.40gb
        - devices: all
          migenabled: true
          migdevices:
            3g.40gb: 2

# 通过 Node Label 触发切分
kubectl label node gpu-node-1 nvidia.com/mig.config=all-1g.10gb

# MIG Manager 检测到 label 变化:
#   1. 读取 ConfigMap 中的 all-1g.10gb 配置
#   2. 调用 nvidia-mig-parted 执行切分
#   3. 重置 Device Plugin（重新 List 设备）
#   4. kubelet 上报新资源:
#        nvidia.com/mig-1g.10gb: 56  (8 GPU × 7 实例)
```

**MIG 隔离保证：**

| 隔离维度 | 说明 |
|---------|------|
| **计算隔离** | 每个实例有独立的 SM（流多处理器），互不争抢算力 |
| **显存隔离** | 硬件级显存分区，一个实例 OOM 不影响其他实例 |
| **故障隔离** | 一个实例的 GPU 硬件错误不会传播到其他实例 |
| **性能隔离** | 实例间不共享 L2 Cache，无缓存争用 |

---

**7. Network Operator（独立 Operator，与 GPU Operator 配合）**

管理 RDMA/InfiniBand 网卡驱动和设备注册，让 GPU Pod 能使用高性能网络。

```
Network Operator 组件:

┌─────────────────────────────────────────────────────────┐
│  Network Operator Controller                             │
│  监听 CRD: NicClusterPolicy                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ OFED Driver Container (DaemonSet)                │   │
│  │                                                  │   │
│  │ 安装 Mellanox OFED 驱动栈:                       │   │
│  │   - mlx5_core (网卡驱动)                         │   │
│  │   - ib_core (InfiniBand 核心)                    │   │
│  │   - rdma_cm (RDMA 连接管理)                      │   │
│  │   - ib_uverbs (用户态 verbs API)                 │   │
│  │                                                  │   │
│  │ 安装后节点出现:                                   │   │
│  │   /dev/infiniband/uverbs0  (verbs 设备)          │   │
│  │   /dev/infiniband/rdma_cm  (连接管理)            │   │
│  │   ib0, ib1 网络接口                              │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ SR-IOV Network Device Plugin (DaemonSet)         │   │
│  │                                                  │   │
│  │ 利用网卡 SR-IOV 虚拟化，将物理网卡切分为多个 VF:  │   │
│  │                                                  │   │
│  │ 物理网卡 (PF): ConnectX-7 400G                   │   │
│  │   ├── VF 0 → 注册为 rdma/hca 资源               │   │
│  │   ├── VF 1 → 注册为 rdma/hca 资源               │   │
│  │   ├── VF 2 → 注册为 rdma/hca 资源               │   │
│  │   └── ...                                       │   │
│  │                                                  │   │
│  │ kubelet 上报:                                    │   │
│  │   rdma/hca: 8        (8 个 RDMA VF 可用)        │   │
│  │   rdma/hca_handles: 2000  (MR handle 总量)       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Multus CNI (可选)                                │   │
│  │                                                  │   │
│  │ 为 Pod 附加多个网络接口:                          │   │
│  │   eth0: 默认 K8s 网络 (Calico/Cilium) → 业务流量 │   │
│  │   net1: RDMA 网络 (InfiniBand/RoCE)   → GPU 通信 │   │
│  │                                                  │   │
│  │ Pod 内:                                         │   │
│  │   eth0: 10.244.x.x (K8s Service 通信)           │   │
│  │   net1: 192.168.100.x (RDMA GPU 通信)           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**SR-IOV 原理：**

```
SR-IOV (Single Root I/O Virtualization):

  物理网卡 (PF - Physical Function):
  ┌──────────────────────────────────────┐
  │  ConnectX-7 400GbE                   │
  │  ┌─────────────────────────────────┐ │
  │  │ 硬件队列 + DMA 引擎 + 协议处理 │ │
  │  └────────┬────────────────────────┘ │
  │           │ 硬件虚拟化               │
  │  ┌────┬──┴───┬────┬────┬────┬────┐  │
  │  │ VF0│ VF1 │ VF2│ VF3│ VF4│ VF5│  │  ← 轻量虚拟网卡
  │  │    │     │    │    │    │    │  │    各自有独立队列
  │  └────┴─────┴────┴────┴────┴────┘  │    直接操作硬件
  └──────────────────────────────────────┘

  优势:
    - VF 直接访问硬件，无 hypervisor 开销
    - 每个 VF 直通给容器，延迟极低
    - 硬件级隔离，VF 间互不影响
```

---

**8. GDS — GPU Direct Storage（可选组件）**

让 NVMe 存储设备绕过 CPU 内存，直接将数据 DMA 写入 GPU 显存。主要用于 AI 训练的数据加载加速。

```
传统存储读取路径:
  NVMe → PCIe → CPU 内存 (bounce buffer) → PCIe → GPU 显存
  (2 次 PCIe 传输，CPU 参与数据搬运)

GDS (GPU Direct Storage):
  NVMe → PCIe → GPU 显存
  (1 次 PCIe 传输，CPU 完全不参与)

  需要: nvidia_fs.ko 内核模块 + 支持 GDS 的 NVMe 驱动
  收益: 数据加载吞吐提升 2-3 倍，CPU 利用率降低
```

#### ClusterPolicy 完整示例

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  # 驱动管理
  driver:
    enabled: true
    version: "535.104.05"
    useOpenKernelModules: true      # 开源内核模块（H100+ 推荐）

  # 容器运行时
  toolkit:
    enabled: true

  # GPU 设备注册
  devicePlugin:
    enabled: true
    config:
      name: device-plugin-config     # 拓扑感知配置

  # MIG 切分
  migManager:
    enabled: true                    # A100/H100 启用

  # 监控
  dcgmExporter:
    enabled: true
    serviceMonitor:
      enabled: true                  # 自动创建 Prometheus ServiceMonitor

  # GPU Direct Storage
  gds:
    enabled: false                   # 按需启用

  # GPU Direct RDMA
  gdrcopy:
    enabled: false                   # 按需启用（跨机 GPU 通信优化）

  # VFIO 直通（VM 场景）
  vfioManager:
    enabled: false

  # 节点标签
  nodeStatusExporter:
    enabled: true                    # 将 GPU 状态写入 Node Label
```

#### 组件协作流程

```
一个 GPU Pod 从创建到运行的完整链路:

1. NFD 检测节点 → 标记 nvidia.com/gpu.present=true, gpu.count=8

2. Driver Container 安装驱动 → /dev/nvidia* 设备就绪

3. Container Toolkit 配置 runtime → containerd 识别 nvidia RuntimeClass

4. Device Plugin 注册资源 → Node.Allocatable: nvidia.com/gpu=8

5. 用户提交 Pod (nvidia.com/gpu: 2)
       │
       ▼
6. Scheduler 匹配 → 调度到有 2+ GPU 的节点

7. kubelet 调用 Device Plugin Allocate()
       │ → 选择 GPU 0,1
       │ → 设置 NVIDIA_VISIBLE_DEVICES=0,1
       ▼
8. kubelet 调用 containerd + nvidia-container-runtime
       │ → 挂载 /dev/nvidia0, /dev/nvidia1
       │ → 注入 CUDA 库
       ▼
9. 容器启动 → 容器内 nvidia-smi 看到 2 张 GPU

10. DCGM Exporter 持续采集该 Pod 的 GPU 指标 → Prometheus
```

---

## 2. RDMA（Remote Direct Memory Access）

### 核心思想

传统网络通信中，数据要经过 **内核协议栈** 多次拷贝：

```
传统 TCP/IP 通信（发送端）:

  应用缓冲区 (用户态)
       │ ① copy_from_user
       ▼
  Socket 缓冲区 (内核态)
       │ ② TCP/IP 封装
       ▼
  协议栈缓冲区 (内核态)
       │ ③ DMA copy
       ▼
  网卡缓冲区 (硬件)
       │ 网络传输
       ▼
  ... 接收端再做一遍反向拷贝 ...
```

每次通信至少 **4 次内存拷贝 + 2 次上下文切换**（用户态↔内核态），延迟高、CPU 开销大。

**RDMA 的核心突破**：网卡直接读写应用内存，绕过内核协议栈。

```
RDMA 通信（发送端）:

  应用缓冲区 (用户态)          ← 注册为 Memory Region (MR)
       │                          网卡持有虚拟地址映射
       │ ① 网卡直接 DMA 读取（零拷贝）
       ▼
  RDMA 网卡 (硬件)             ← 硬件实现可靠传输协议
       │ 网络传输
       ▼

RDMA 通信（接收端）:

  RDMA 网卡 (硬件)
       │ ② 网卡直接 DMA 写入（零拷贝）
       ▼
  应用缓冲区 (用户态)          ← 数据直接到达，无需内核介入
```

**关键特性：**

| 特性 | 说明 |
|------|------|
| **零拷贝** | 数据在应用缓冲区和网卡之间直接 DMA 传输，不经过内核 |
| **内核旁路** | 应用通过 verbs API 直接向网卡下发指令，无需系统调用 |
| **硬件卸载** | 可靠传输、拥塞控制、分段重组由网卡硬件完成 |
| **CPU 卸载** | CPU 不参与数据搬运，可去做计算 |

### Memory Region (MR) 注册

```
应用程序:
  1. malloc() 分配缓冲区
  2. ibv_reg_mr() 注册为 Memory Region
     → 网卡获得该缓冲区的物理地址映射
     → 返回 lkey（本地访问）和 rkey（远程访问）

后续通信:
  发送方: ibv_post_send(wr) → 网卡用 lkey 读取本地缓冲区
  接收方: 网卡用 rkey 直接写入远程缓冲区
```

### RDMA 传输模式

| 模式 | 说明 | 类比 |
|------|------|------|
| **RC (Reliable Connection)** | 可靠连接，一对一，有序交付 | 类似 TCP |
| **UC (Unreliable Connection)** | 不可靠连接，一对一，无序 | 类似 UDP |
| **UD (Unreliable Datagram)** | 不可靠数据报，一对多 | 类似 UDP 广播 |

深度学习训练通常使用 **RC 模式**（需要可靠传输）。

### 三种 RDMA 网络实现

```
┌────────────────────────────────────────────────────────────┐
│                  RDMA 三种实现方式                          │
├──────────────────┬──────────────────┬──────────────────────┤
│   InfiniBand     │   RoCE v2        │   iWARP              │
│   (原生 RDMA)    │   (RDMA over     │   (RDMA over         │
│                  │    Converged     │    TCP/IP)           │
│                  │    Ethernet)     │                      │
├──────────────────┼──────────────────┼──────────────────────┤
│ 专用网络硬件     │ 以太网 + RDMA    │ 以太网 + TCP         │
│ 专用交换机       │ 需 PFC/ECN      │ 标准交换机           │
│ 专用网卡(HCA)   │ 普通交换机可跑   │ 普通网卡(卸载到硬件) │
│                  │ (性能略降)       │                      │
├──────────────────┼──────────────────┼──────────────────────┤
│ 延迟: ~1μs      │ 延迟: ~2-3μs    │ 延迟: ~5-10μs       │
│ 带宽: 400Gbps   │ 带宽: 400Gbps   │ 带宽: 100Gbps       │
│ (HDR/NDR)       │ (400GbE)        │                      │
├──────────────────┼──────────────────┼──────────────────────┤
│ 最贵，性能最好   │ 性价比主流方案   │ 最便宜，性能最差     │
│ HPC / 超大规模   │ 大多数 GPU 集群  │ 对延迟不敏感场景     │
│ 训练             │                  │                      │
└──────────────────┴──────────────────┴──────────────────────┘
```

**RoCE v2 是当前 GPU 集群主流**：用以太网承载 RDMA 流量，复用现有网络基础设施，通过 PFC（Priority Flow Control）保证无损传输。

### RoCE v2 协议栈

```
┌─────────────────────────────────┐
│  应用层 (NCCL / MPI)           │  ← verbs API
├─────────────────────────────────┤
│  RDMA Transport (IB Verbs)      │  ← 硬件实现
├─────────────────────────────────┤
│  UDP                            │  ← 用于路由（ECMP）
├─────────────────────────────────┤
│  IP                             │  ← 标准 IP 路由
├─────────────────────────────────┤
│  Ethernet                       │  ← 标准以太网
├─────────────────────────────────┤
│  PFC + ECN                      │  ← 无损网络保障
│  (Priority Flow Control)        │     不丢包 + 拥塞通知
│  (Explicit Congestion Notif.)   │
└─────────────────────────────────┘
```

### RDMA 在 K8s 中的使用

```
┌─────────────────────────────────────────────────────┐
│  Node (Worker)                                      │
│                                                     │
│  ┌───────────────┐         ┌─────────────────────┐  │
│  │  kubelet      │         │  Mellanox Network   │  │
│  │               │         │  Operator           │  │
│  │  Device Plugin│◄────────│  (SR-IOV DP)        │  │
│  │  Manager      │         │                     │  │
│  └───────────────┘         │  注册资源:           │  │
│                            │  rdma/hca: 2        │  │
│                            │  rdma/hca_handles:  │  │
│                            │  2000               │  │
│                            └─────────────────────┘  │
└─────────────────────────────────────────────────────┘

Pod YAML:
  resources:
    limits:
      nvidia.com/gpu: 8
      rdma/hca: 2              # 申请 2 个 RDMA HCA（网卡）
      rdma/hca_handles: 2000   # 申请 2000 个 MR handle
```

---

## 3. 集合通信（Collective Communication）

多 GPU 训练的核心问题是**梯度同步**——每张卡算出局部梯度后，需要聚合为全局梯度。这依赖集合通信原语。

### 核心原语

```
AllReduce（最常用 — 梯度聚合）:
  GPU0: [1, 2, 3]  ──┐
  GPU1: [4, 5, 6]  ──┤  Sum    [15, 21, 27]
  GPU2: [10,14,18] ──┘         → 分发到所有 GPU

Broadcast:
  GPU0: [a, b, c]  ──────► GPU1: [a, b, c]
                           GPU2: [a, b, c]

ReduceScatter:
  GPU0: [1,2,3,4] ──┐         GPU0: [sum_chunk0]
  GPU1: [5,6,7,8] ──┤ Reduce  GPU1: [sum_chunk1]
  GPU2: [9,10,11,12]┘ Scatter GPU2: [sum_chunk2]

AllGather:
  GPU0: [a] ──┐              GPU0: [a,b,c,d]
  GPU1: [b] ──┤ Gather       GPU1: [a,b,c,d]
  GPU2: [c] ──┤ ──────►     GPU2: [a,b,c,d]
  GPU3: [d] ──┘
```

### AllReduce 实现算法

**朴素 AllReduce（Naive）** — 所有 GPU 发给一个节点求和再广播：

```
GPU0 ──┐                    ┌── GPU0: [sum]
GPU1 ──┤──► GPU0: Reduce ──┤── GPU1: [sum]   2N 次通信
GPU2 ──┘    (求和)          └── GPU2: [sum]   瓶颈在 GPU0

通信量: 2 × (N-1) × DataSize
```

**Ring AllReduce** — 数据切成 N 份，沿环形流水线传输：

```
4 个 GPU 组成环: GPU0 → GPU1 → GPU2 → GPU3 → GPU0

阶段 1: ReduceScatter（每步传 1/N 数据）
  Step 1: GPU0→GPU1(chunk3), GPU1→GPU2(chunk0), GPU2→GPU3(chunk1), GPU3→GPU0(chunk2)
  Step 2: GPU0→GPU1(reduced_2,3), ...
  Step 3: GPU0→GPU1(reduced_1,2,3), ...
  → 每个 GPU 持有 1/N 的完整 reduce 结果

阶段 2: AllGather（沿环传递完整 chunk）
  Step 1~3: 每个 GPU 将完整 chunk 传给下一个
  → 每个 GPU 获得完整的全局梯度

通信量: 2 × (N-1)/N × DataSize ≈ 2 × DataSize (与 N 无关!)
```

```
Ring AllReduce vs Naive AllReduce:

        Naive              Ring
GPU 数  通信量              通信量
  2     2 × D              1 × D
  4     6 × D              1.5 × D
  8     14 × D             1.75 × D
  16    30 × D             1.875 × D
  64    126 × D            1.97 × D
                            ↑ 接近理论最优 2×D

D = 总数据量
```

**Tree AllReduce** — 二叉树结构，两阶段完成：

```
Reduce 阶段（自底向上）:         Broadcast 阶段（自顶向下）:

    GPU0                            GPU0 [sum]
    /  \                            /  \
  GPU1  GPU2                      GPU1  GPU2
  / \    / \                      / \    / \
GPU3 GPU4 GPU5 GPU6            GPU3 GPU4 GPU5 GPU6
 [s]  [s]  [s]  [s]            [sum][sum][sum][sum]

通信量: 2 × (N-1) × (D/N) ≈ 2 × D (与 Ring 相当)
优势: 延迟更低（log(N) 步 vs N 步），适合小数据量
```

### NCCL（NVIDIA Collective Communication Library）

NCCL 是 NVIDIA 提供的高性能集合通信库，**自动选择最优通信拓扑和路径**。

```
┌──────────────────────────────────────────┐
│  深度学习框架 (PyTorch / TensorFlow)      │
│  torch.distributed.all_reduce(grad)       │
├──────────────────────────────────────────┤
│  NCCL                                    │
│  ┌────────────────────────────────────┐  │
│  │  Topology Discovery               │  │
│  │  自动探测: PCIe / NVLink / NVSwitch│  │
│  │          / RDMA / IB / RoCE       │  │
│  ├────────────────────────────────────┤  │
│  │  Algorithm Selection              │  │
│  │  Ring / Tree / Direct / CollNet   │  │
│  │  根据数据量和拓扑自动选择          │  │
│  ├────────────────────────────────────┤  │
│  │  Channel / Stream Pipeline        │  │
│  │  多通道并行传输，隐藏延迟          │  │
│  ├────────────────────────────────────┤  │
│  │  Transport Layer                  │  │
│  │  P2P (NVLink) / SHM / IB / Socket │  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│  硬件: GPU + NVLink + NIC               │
└──────────────────────────────────────────┘
```

**NCCL 自动选择策略：**

| 通信范围 | 拓扑 | NCCL 选择 |
|---------|------|----------|
| 同机同 NVLink 域 | GPU 间直连 | NVLink P2P（最快，~900GB/s H100） |
| 同机不同 NVLink 域 | 需跨 PCIe Switch | NVLink + PCIe P2P |
| 跨机（RDMA 可用） | 有 InfiniBand/RoCE | IB Verbs / RoCE（GPU Direct RDMA） |
| 跨机（无 RDMA） | 仅 TCP | TCP Socket（最慢） |

**GPU Direct RDMA**：GPU 显存直接通过 RDMA 网卡传输，不经过 CPU 内存。

```
传统跨机 GPU 通信:
  GPU0 显存 → PCIe → CPU 内存 → PCIe → 网卡 → 网络 → 网卡 → PCIe → CPU 内存 → PCIe → GPU1 显存
  (4 次拷贝)

GPU Direct RDMA:
  GPU0 显存 → PCIe → 网卡 → 网络 → 网卡 → PCIe → GPU1 显存
  (0 次 CPU 内存拷贝，延迟降低 30-50%)
```

---

## 4. 数据并行（Data Parallelism, DP）

### 核心思想

将**训练数据**切分到多张 GPU 上，每张卡持有**完整的模型副本**，各自计算局部梯度后通过 AllReduce 聚合。

```
                 训练数据集 (100 万样本)
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
        GPU 0        GPU 1        GPU 2
     ┌─────────┐  ┌─────────┐  ┌─────────┐
     │ 模型副本 │  │ 模型副本 │  │ 模型副本 │
     │ (完整)   │  │ (完整)   │  │ (完整)   │
     ├─────────┤  ├─────────┤  ├─────────┤
     │ 数据     │  │ 数据     │  │ 数据     │
     │ shard 0  │  │ shard 1  │  │ shard 2  │
     │ 33万样本 │  │ 33万样本 │  │ 34万样本 │
     ├─────────┤  ├─────────┤  ├─────────┤
     │ 前向传播 │  │ 前向传播 │  │ 前向传播 │
     │ 反向传播 │  │ 反向传播 │  │ 反向传播 │
     │ 局部梯度 │  │ 局部梯度 │  │ 局部梯度 │
     └────┬────┘  └────┬────┘  └────┬────┘
          │            │            │
          └────────────┼────────────┘
                       │
                  AllReduce
                  (梯度求和/平均)
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       全局梯度     全局梯度      全局梯度
          │            │            │
          ▼            ▼            ▼
       更新模型     更新模型      更新模型
       (相同权重)  (相同权重)   (相同权重)
```

### 训练步骤

```
每个训练 iteration:

1. 前向传播 (Forward)
   每张 GPU 独立用自己的数据 shard 做前向计算
   → 得到 loss

2. 反向传播 (Backward)
   每张 GPU 独立计算局部梯度 (∂loss/∂weights)
   → 每张卡有一份梯度

3. 梯度同步 (AllReduce)
   所有 GPU 的局部梯度做 AllReduce（求和或平均）
   → 每张卡获得相同的全局梯度
   ← 这是通信瓶颈所在

4. 参数更新 (Optimizer Step)
   每张 GPU 用相同的全局梯度更新模型
   → 所有卡的模型权重保持一致
```

### PyTorch DDP（Distributed Data Parallel）

```python
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

# 1. 初始化分布式环境（NCCL 后端 = GPU 通信）
dist.init_process_group(backend='nccl')
local_rank = dist.get_rank()
torch.cuda.set_device(local_rank)

# 2. 创建模型并包装为 DDP
model = MyModel().to(local_rank)
model = DDP(model, device_ids=[local_rank])

# 3. 数据切分（DistributedSampler 自动分片）
sampler = torch.utils.data.DistributedSampler(dataset)
loader = DataLoader(dataset, sampler=sampler, batch_size=32)

# 4. 训练循环
for epoch in range(epochs):
    sampler.set_epoch(epoch)    # 确保每个 epoch 分片不同
    for batch in loader:
        loss = model(batch).loss
        loss.backward()         # 反向传播 → 局部梯度
        # DDP 自动 hook: 反向传播完成后触发 AllReduce
        optimizer.step()        # 用全局梯度更新参数
        optimizer.zero_grad()
```

**DDP 通信优化：**

```
朴素实现: 等所有梯度算完 → 一次性 AllReduce 整个梯度张量
  → 通信和计算串行

DDP 实现: 梯度分桶 (bucket) + 流水线
  → 第一层梯度算完就立即 AllReduce 该桶
  → 后续层继续反向传播的同时，前面的桶在并行通信
  → 计算和通信 overlap，隐藏延迟

  Layer 1 grad ──► AllReduce bucket 1 ──┐
  Layer 2 grad ──► AllReduce bucket 2 ──┤ 并行
  Layer 3 grad ──► AllReduce bucket 3 ──┘
  (计算和通信 overlap)
```

### DP 的扩展性瓶颈

```
GPU 数    理想加速比    实际加速比    原因
  1         1x           1x          -
  2         2x           1.9x        AllReduce 开销小
  4         4x           3.6x        通信占比增大
  8         8x           6.8x        单节点 NVLink 高效
 16        16x          13x          跨机通信瓶颈
 32        32x          22x          AllReduce 时间占比过大
 64        64x          35x          通信成为主要瓶颈

当模型参数量大 + GPU 数量多时:
  AllReduce 通信量 = 2 × 参数量 × 4 bytes (fp32)
  GPT-3 (175B 参数): 每次 AllReduce 传输 700GB 数据
```

**解决方案**：
- **小规模（≤8 卡）**：DP 足够，NVLink 互联通信开销可控
- **中规模（8-64 卡）**：DP + 高带宽 RDMA（InfiniBand/RoCE）
- **大规模（64+ 卡）**：需要**模型并行**（Tensor Parallel / Pipeline Parallel）配合 DP，减少单卡参数量

---

## 5. 其他并行策略简介

当单张 GPU 放不下整个模型时，DP 无法使用，需要模型级并行。

### Tensor Parallelism（张量并行）

将**单个算子**的权重矩阵切分到多张 GPU：

```
矩阵乘法 Y = X × W (W 是 8192×8192)

切分为 2 张 GPU:
  GPU0: W[:, :4096]    GPU1: W[:, 4096:]
  X × W_0 → Y_0        X × W_1 → Y_1
  AllGather(Y_0, Y_1) → 完整 Y
```

**特点**：通信量小（AllGather），但需要高频同步 → 仅限同机 NVLink 互联。

### Pipeline Parallelism（流水线并行）

将模型按**层**切分，形成流水线：

```
Model layers [1-20] on GPU0, [21-40] on GPU1, [41-60] on GPU2

  GPU0:  [micro-batch 1] [micro-batch 2] [micro-batch 3] ...
           ↓               ↓               ↓
  GPU1:    [micro-batch 1] [micro-batch 2] [micro-batch 3] ...
             ↓               ↓               ↓
  GPU2:      [micro-batch 1] [micro-batch 2] [micro-batch 3] ...
```

**特点**：通信量小（只传激活值），但有 pipeline bubble（空闲等待时间）。

### 混合并行（3D Parallelism）

大规模训练通常三种并行组合使用：

```
64 GPU 集群训练 GPT-3:

  数据并行 (DP = 8):     8 组，每组独立数据 shard
  张量并行 (TP = 4):     每组内 4 卡切分算子
  流水线并行 (PP = 2):   每组内 2 级流水线

  8 × 4 × 2 = 64 GPUs

  通信模式:
    TP: AllReduce / AllGather (同机 NVLink，高频)
    PP: P2P Send/Recv (跨机 RDMA，低频)
    DP: AllReduce (跨机 RDMA，每个 iteration 一次)
```

---

## 6. K8s 上的分布式训练架构

### 典型训练任务部署

```yaml
# PyTorchJob (Kubeflow Training Operator)
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: gpt-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: training-image:latest
            resources:
              limits:
                nvidia.com/gpu: 8
                rdma/hca: 2
    Worker:
      replicas: 7                # 总共 8 节点 × 8 GPU = 64 GPU
      template:
        spec:
          containers:
          - name: pytorch
            image: training-image:latest
            resources:
              limits:
                nvidia.com/gpu: 8
                rdma/hca: 2
```

### 完整硬件拓扑

```
Node 0                              Node 1
┌─────────────────────────────┐     ┌─────────────────────────────┐
│                             │     │                             │
│  ┌─────┐ ┌─────┐ ┌─────┐   │     │   ┌─────┐ ┌─────┐ ┌─────┐  │
│  │GPU 0│ │GPU 1│ │ ... │   │     │   │GPU 0│ │GPU 1│ │ ... │  │
│  └──┬──┘ └──┬──┘ └──┬──┘   │     │   └──┬──┘ └──┬──┘ └──┬──┘  │
│     └───────┼───────┘       │     │      └───────┼───────┘      │
│          NVLink/NVSwitch    │     │           NVLink/NVSwitch   │
│             │               │     │              │              │
│     ┌───────┴───────┐       │     │      ┌───────┴───────┐     │
│     │  NVLink 域     │       │     │      │  NVLink 域     │     │
│     │  (8 GPU 全互联)│       │     │      │  (8 GPU 全互联)│     │
│     └───────┬───────┘       │     │      └───────┬───────┘     │
│             │ PCIe          │     │              │ PCIe        │
│     ┌───────┴───────┐       │     │      ┌───────┴───────┐     │
│     │  RDMA NIC ×2  │       │     │      │  RDMA NIC ×2  │     │
│     │  (200Gbps each)│      │     │      │  (200Gbps each)│     │
│     └───────┬───────┘       │     │      └───────┬───────┘     │
└─────────────┼───────────────┘     └──────────────┼──────────────┘
              │ RoCE v2 / InfiniBand               │
              └────────────┐    ┌──────────────────┘
                           ▼    ▼
                    ┌──────────────┐
                    │  交换机       │
                    │  (PFC + ECN) │
                    │  无损网络     │
                    └──────────────┘

通信层级:
  同机 GPU 间:  NVLink (~900 GB/s H100)  → Tensor Parallel
  跨机 GPU 间:  RDMA (~400 Gbps)          → Data Parallel + Pipeline Parallel
```

---

## 本章小结

| 组件 | 一句话总结 | 关键点 |
|------|-----------|--------|
| Device Plugin | K8s GPU 调度入口 | List/Allocate, Extended Resource |
| GPU Operator | 一键部署 GPU 全栈 | Driver + Toolkit + DP + DCGM |
| RDMA | 网卡直接 DMA 读写应用内存 | 零拷贝, 内核旁路, 硬件卸载 |
| InfiniBand | 原生 RDMA 网络 | 延迟最低(~1μs), 最贵 |
| RoCE v2 | 以太网上跑 RDMA | 主流方案, 需 PFC/ECN |
| NCCL | NVIDIA 集合通信库 | 自动拓扑探测, 算法选择 |
| GPU Direct RDMA | GPU 显存直接走网卡 | 跳过 CPU 内存, 降延迟 30-50% |
| AllReduce | 梯度聚合核心原语 | Ring / Tree 算法 |
| Data Parallel | 数据切分 + 模型副本 | DDP, 梯度分桶流水线 |
| Tensor Parallel | 算子级权重切分 | 同机 NVLink, AllGather |
| Pipeline Parallel | 层级模型切分 | 跨机, 流水线减少 bubble |

**上一章：[05-配置与存储](05-配置与存储.md)** | **下一章：[06-核心业务流程](06-核心业务流程.md)** →
