# 2602.18007v1.pdf - Joint Training on AMD and NVIDIA GPUs

## 中文标题：AMD 与 NVIDIA GPU 的联合训练方案

### 中文摘要
随着大语言模型（LLM）规模的不断扩大，单一厂商的同构集群已逐渐无法满足计算和系统容量的需求。本文提出了一种针对 
AMD-NVIDIA 异构混合训练的技术解决方案。首先，作者采用了一种面向兼容性的“CPU 转发通信”方法，通过在不同并行组间
选择差异化的通信后端，并利用多网卡并行数据传输进行优化。为了追求更高性能，本文进一步提出了一种“设备直接通信”
方案，集成了一种 CPU 卸载（CPU-offloading）的点对点（P2P）机制，实现了跨厂商 GPU 间的直接数据传输，无需经过主
机内存中转。在 LLaMA-8B 和 Qwen2-7B 模型上的实验表明，该设备直接通信方案在 AMD-NVIDIA 异构集群上的吞吐量可达到 
NVIDIA 同构系统的 98%，同时保持了训练的稳定性和正确性。

### 各章节主旨内容
- **Chapter 1: Introduction** - 阐述了异构计算集群产生的必然性（供应链多样化、硬件迭代快），并指出跨厂商资源聚合
  是克服计算瓶颈的关键挑战。
- **Chapter 2: Approach** - 详细介绍了两种混合训练方案：基于 Gloo 的 CPU 转发通信方案及其优化，以及核心的设备直接
  通信方案（涉及 CPU 卸载 P2P 机制和多适配器架构）。
- **Chapter 3: Results and Analysis** - 在 H200 和 MI325X 组成的测试床上验证了方案的性能（吞吐量）、稳定性和收敛正
  确性。
- **Chapter 4: Discussion** - 探讨了为什么异构性目前仅限于流水线并行（PP），并分析了模型切分策略对性能的影响及工
  程实现中的挑战。
- **Chapter 5: Conclusions** - 总结了研究成果，强调了设备直接通信与合理切分策略相结合是利用异构 GPU 资源的有效路径。

### 中文结论
本文研究了 AMD-NVIDIA 环境下的异构混合训练，并提供了端到端的实践方案。通过实现设备直接通信机制，有效地消除了主
机内存拷贝这一瓶颈。实验证明，该方案能让异构集群以极高的效率（接近 NVIDIA 同构集群的 98%）支持大规模预训练，为
高效利用多元计算资源指明了方向。

---

# 章节详细解读

## Chapter 1: Introduction

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：大模型参数量已突破万亿级，算力需求呈指数级增长。单一厂商的同构集群受限于硬件迭代周期和供应链多样
  性，难以规模化扩张。虽然 NVIDIA 和 AMD 各有千秋，但两者在内存容量、互连带宽（NVLink vs PCIe/Infinity Fabric）
  上存在巨大差异，如何高效聚合这些异构资源是核心痛点。
- **核心动机**：打破厂商锁定的生态隔离，通过软件层面的适配和底层通信协议的重构，实现跨平台算力协同。
- **设计目标**：在 AMD 和 NVIDIA 混合集群上实现无缝训练，确保吞吐量损失最小化，同时保证数值精度和训练过程的稳定。

## Chapter 2: Approach

### 1. 设计思想 (Design Philosophy)
- **演进逻辑**：从简单的“能跑通”向“高性能”演进。初期方案（CPU 转发）利用现有的 Gloo 库解决兼容性，后期方案（设备
  直接通信）则直接操作底层硬件以消除 CPU 中转开销。

### 2. 关键技术 (Key Technologies)
- **CPU-Forwarding Communication 的优化**：
    - **DCBS (差异化通信后端选择)**：在 Megatron 框架中，针对 PP（流水线并行）组使用 Gloo 处理异构互连，而 DP 和 
      TP 组依然保持同构并使用原厂库（NCCL/RCCL）以维持峰值性能。
    - **MPDT (多网卡并行数据传输)**：为每个 GPU 分配独立网卡处理 PP 通信，物理上并行化 PP 流量。
- **Device-Direct Communication (设备直接通信)**：
    - **CPU-offloading P2P 机制**：将控制平面（内存注册、连接管理、事件处理）与数据平面解耦。主机仅负责控制流，
      数据流通过 D2D 拷贝进入 Chunk Buffer，再通过 GPUDirect RDMA 直接外发。
    - **多适配器架构**：通过 Device、Net-Plugin 和 CCL 三层适配器屏蔽硬件差异。Device 层封装 CUDA/ROCm API，
      Net-Plugin 处理 ibverbs 和 RDMA 管理，CCL 层标准化集合通信接口（AllReduce 等）。

### 3. 实现细节 (Implementation Details)
- **通信流程**：在 Device-Direct 模式下，源 GPU 首先执行设备内拷贝（D2D）将数据移至 Chunk Buffer，随后利用 
  GPUDirect RDMA 将数据直接推送到对方网卡。接收端执行对称操作。
- **初始化流程**：包括资源发现（加载厂商接口）和拓扑感知（通过 Bootstrap 网络执行 AllGather 同步集群元数据）。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **对比分析**：原生 Gloo 后端虽然兼容性好，但全局数据需经过 CPU，导致频繁的 H2D/D2H 拷贝。Device-Direct 方案虽
  然工程实现极其复杂，但彻底释放了硬件潜力。
- **权衡考量**：作者选择了将异构性限制在 PP 维度，因为 PP 通信量相对较小且主要是点对点，而 DP/TP 的集合通信对拓
  扑和算法极度敏感，强行做异构会引入不可控的开销。

## Chapter 3 & 4: Results, Analysis & Discussion

### 1. 设计思想 (Design Philosophy)
- **验证维度**：性能（吞吐量是否达标）、稳定性（长时训练是否中断）、正确性（Loss 是否收敛）。
- **优化直觉**：由于 AMD 和 NVIDIA 算力不对等，简单的均匀切分会导致流水线气泡。

### 2. 关键技术 (Key Technologies)
- **不均匀模型切分 (Uneven Partitioning)**：针对 LLaMA-8B，在 AMD 端分配 15 层，NVIDIA 端分配 17 层；针对 
  Qwen2-7B，分配比例为 12:16。这种非对称切分是为了对冲 AMD MI325X 与 NVIDIA H200 之间的吞吐量差异，实现负载均衡。

### 3. 实现细节 (Implementation Details)
- **测试床配置**：NVIDIA 节点（8x H200，NVLink 900GB/s），AMD 节点（8x MI325X，Infinity Fabric 128GB/s），
  节点间通过 100GB/s 的 BlueField-3 DPU 互连。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **性能结果**：Device-Direct 方案远优于优化的 Gloo 方案。LLaMA-8B 下异构集群吞吐量达到 NVIDIA 同构的 98.2%，
  甚至略高于纯 AMD 集群（因为混合了更高性能的 H200）。
- **工程挑战**：尽管 RCCL 和 NCCL 接口相似，但在异构混合训练中极易出现 Hang（挂起）问题，需要大量的底层调试和事
  件驱动逻辑的微调。
