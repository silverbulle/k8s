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

生产环境通常部署 **GPU Operator**，一个 Operator 自动完成所有 GPU 相关组件的安装和管理：

```
GPU Operator (CRD: ClusterPolicy)
  │
  ├── NVIDIA Driver (DaemonSet)        — 宿主机 GPU 驱动
  ├── Container Toolkit (DaemonSet)    — nvidia-container-runtime
  ├── Device Plugin (DaemonSet)        — 向 kubelet 注册 GPU 资源
  ├── DCGM Exporter (DaemonSet)        — GPU 监控指标（温度、利用率、显存）
  ├── MIG Manager (DaemonSet)          — A100/H100 多实例 GPU 切分
  └── Network Operator (可选)          — RDMA/InfiniBand 驱动和 Device Plugin
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
