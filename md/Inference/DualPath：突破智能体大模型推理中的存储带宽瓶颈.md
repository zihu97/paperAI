# DualPath: Breaking the Storage Bandwidth Bottleneck in Agentic LLM Inference
2602.21548v2.pdf
## 中文标题：DualPath：突破智能体 LLM 推理中的存储带宽瓶颈

### 中文摘要
本文针对智能体（Agentic）大语言模型（LLM）推理中日益凸显的 KV-Cache 存储 I/O 瓶颈问题，提出了 DualPath 推理系统。
在当前主流的预填充-解码（Prefill-Decode, PD）分离架构中，从外部存储加载海量 KV-Cache 导致了严重的不平衡：
预填充引擎的存储网卡（NIC）带宽经常饱和，而解码引擎的存储网卡则处于闲置状态，这极大地限制了系统的整体吞吐量。
DualPath 通过引入“双路径 KV-Cache 加载”机制打破了这一瓶颈。除了传统的“存储到预填充”路径外，DualPath 
新增了“存储到解码”路径，将 KV-Cache 先加载到解码引擎中，再通过计算网络的高速 RDMA 高效传输至预填充引擎。
结合全局调度器动态平衡预填充与解码引擎间的负载，DualPath 有效规避了网络拥塞并减少了对延迟敏感的模型执行通信的干扰。
在多种模型和真实智能体负载下的评估表明，DualPath 将离线推理吞吐量提升了高达 1.87 倍，在线服务吞吐量平均提升了 1.96 倍。

### 各章节主旨内容
- **Chapter 1: Introduction** - 阐述了 LLM 从单轮对话向多轮智能体系统演进的趋势，指出智能体负载具有极高的 KV-Cache 
  命中率（>95%），使得 I/O 带宽而非计算能力成为性能主导因素。
- **Chapter 2: Background** - 介绍了 PD 分离架构、层级预填充（Layerwise Prefill）以及现代 AI 数据中心计算与存储网络分离的架构特点。
- **Chapter 3: Bottleneck & Motivation** - 通过数据分析揭示了预填充侧存储网卡带宽饱和导致的 GPU 欠载问题，并指出硬件算力
  增长远超 I/O 增长的趋势加剧了这一矛盾。
- **Chapter 4: DualPath System Overview** - 提出双路径加载架构，介绍推理引擎、流量管理器和请求调度器三大核心组件。
- **Chapter 5: CNIC-Centric Traffic Manager** - 设计了以计算网卡为中心的流量管理方案，通过虚拟通道（VL）隔离
  KV-Cache 传输与推理通信，并利用 RDMA 实现高效的内存拷贝。
- **Chapter 6: Adaptive Request Scheduler** - 详细描述了两级调度算法：引擎间调度用于平衡各节点的存储与计算负载，
  引擎内调度则通过计算配额和分块预填充最小化 GPU 气泡。
- **Chapter 7: Evaluation** - 在 DeepSeek 和 Qwen 模型上进行实验，验证了 DualPath 在离线和在线场景下的巨大吞吐提升。
- **Chapter 8: Discussion** - 探讨了未来潜在的改进方向（如更灵活的 P/D 配置）以及智能体负载对工作集（Working Set）规模的影响。
- **Chapter 9: Related Work** - 回顾了分布式内存缓存池、KV-Cache I/O 优化和推理加速等相关研究。
- **Chapter 10: Conclusion** - 总结了 DualPath 通过重新分配存储负载和负载感知的调度，显著提升了智能体 LLM 推理的性能。

### 中文结论
DualPath 系统成功解决了 PD 分离推理架构在处理长上下文智能体任务时的存储带宽失衡问题。通过创新的双路径加载机制，
它充分利用了解码引擎空闲的存储带宽，并利用计算网络剩余的带宽进行数据分发。配合精细的流量隔离和自适应调度策略，
DualPath 在不增加额外硬件成本的前提下，显著降低了 JCT（任务完成时间）并提高了在线系统的 APS（每秒处理请求数），
为大规模智能体应用的落地提供了关键的系统级支撑。

---

# 章节详细解读

## 第一章至第三章：背景与痛点分析

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：传统的 LLM 推理主要关注计算密度（FLOPS）。但在智能体场景下，模型频繁与外部环境交互（如调用 Python 
  解释器、浏览器），导致多轮对话上下文极长且 KV-Cache 命中率极高。在现有的 PD 分离架构中，所有的 KV-Cache 加载压力
  都集中在预填充引擎（PE）的存储网卡（SNIC）上，而解码引擎（DE）的 SNIC 基本闲置。这种严重的不对称导致 PE 的 SNIC 
  成为系统瓶颈，GPU 常因等待 I/O 而空转。
- **核心动机**：作者观察到计算网卡（CNIC）的带宽通常远大于存储网卡，且在推理过程中存在间歇性的空闲时间。
  因此，核心思路是“变废为宝”——利用解码引擎闲置的存储带宽和计算网络剩余的传输带宽，来分担预填充引擎的 I/O 压力。

### 2. 关键技术 (Key Technologies)
- **硬件趋势分析**：论文指出从 NVIDIA Ampere 到 Blackwell 架构，I/O 与计算的比率下降了 14.4 倍，这使得 I/O 
  墙（I/O Wall）在高性能算力节点上愈发显著。
- **流量特征建模**：智能体任务具有“短追加、长上下文、多轮次”的特征。这意味着每次只需计算少量新 token，
  但必须从存储中检索大量的历史 KV-Cache。

## 第四章：DualPath 系统架构

### 1. 设计思想 (Design Philosophy)
- **设计目标**：将原本孤立、单一路径的 KV-Cache 加载模式转变为全局池化、多路径可调度的能力。实现对存储网卡带宽的聚合利用。

### 2. 关键技术 (Key Technologies)
- **双路径加载机制**：
    - **路径 A (PE Read Path)**：传统的 PE 直接从存储读取。
    - **路径 B (DE Read Path)**：DE 利用自身的 SNIC 读取 KV-Cache 到本地 DRAM，再通过计算网络经由 RDMA 
      发送给对应的 PE。
- **层级流式处理 (Layerwise Streaming)**：为了应对 HBM 容量限制，系统采用层级预填充模式，即每层计算前才将该层
  所需的 KV-Cache 换入 HBM，计算完后立即释放或移至下一步骤。

### 3. 实现细节 (Implementation Details)
- **DE Buffer 与 PE Buffer**：在 DRAM 中划分专用缓冲区，用于缓存从存储读取的“完整块”（Full Block）
  或分发时的“层块”（Layer Block）。
- **交互流程**：在 DE Read Path 中，DE 读取数据后，PE 在计算每一层之前，会通过 RDMA 信号触发数据从 DE 的 
  DRAM 传输到 PE 的 HBM。

## 第五章：以 CNIC 为中心的流量管理器

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：引入额外的 KV-Cache 传输可能会干扰模型执行时的集体通信（如 AllToAll、ReduceScatter），
  这些通信对延迟极其敏感，任何微小的抖动都会导致 GPU 流水线停顿。
- **核心动机**：利用网卡的硬件服务质量（QoS）机制，在 CNIC 层面实现推理通信与数据传输的严格隔离。

### 2. 关键技术 (Key Technologies)
- **虚拟通道 (VL) 隔离**：在 InfiniBand 或 RoCE 网络中，将推理流量分配到高优先级 VL（保留 99% 带宽），
  将 KV-Cache 传输流量分配到低优先级 VL。
- **CNIC 辅助拷贝 (H2D/D2H)**：放弃传统的 `cudaMemcpy`（因其驱动开销大，处理小数据块效率低），改为利用计算网卡
  的 RDMA Write 语义实现本地 Host 到 Device 的拷贝。这样可以将所有 I/O 路径统一到 CNIC 的调度之下。

### 3. 实现细节 (Implementation Details)
- **门铃批处理 (Doorbell Batching)**：在 RDMA 提交请求时使用批处理技术，将 submission 开销降低到约 1 微秒，
  远优于 `cudaMemcpyAsync` 的 5-7 微秒。

## 第六章：自适应请求调度器

### 1. 设计思想 (Design Philosophy)
- **设计目标**：在动态、异构的负载下，决定每个请求走哪条路径，并平衡各 GPU 的计算进度，避免同步等待导致的“长尾”。

### 2. 关键技术 (Key Technologies)
- **引擎间调度 (Inter-Engine Scheduling)**：
    - **三类引擎划分**：根据 GPU 上的未完成 token 数和存储网卡读取队列长度，将 PE 分为“过载”、“最佳候选”和“普通候选”。
    - **负载平衡逻辑**：优先向读取队列短且计算负载轻的引擎分配新请求。
- **引擎内调度 (Intra-Engine Scheduling)**：
    - **计算配额 (Compute Quota)**：为了防止不同 GPU 上的计算量差异导致同步点处的“气泡”，为每个 forward batch 
      设置时间配额，对于超长请求采用分块预填充（Chunked Prefill）。

### 3. 实现细节 (Implementation Details)
- **算法 1 (Inter-PE Scheduling)**：基于节点反馈的 `read_queue` 长度和 `token_count` 动态更新状态，确保请求
  被分发到当前 I/O 响应最快的路径上。

## 第七章：实验评估与结论

### 1. 实现细节 (Implementation Details)
- **实验配置**：使用 1,152 张 GPU 的超大规模集群进行扩展性测试。对比了 SGLang (MC) 等业界主流方案。
- **模型覆盖**：包括 DeepSeek-V3 系列（MoE 架构）和 Qwen 系列（Dense 架构），涵盖了 MLA 和 GQA 等不同注意机制。

### 2. 权衡与对比 (Trade-offs & Comparison)
- **性能增益分析**：消融实验显示，层级预填充贡献了约 17% 的收益，而双路径加载及调度算法共同贡献了约 28% 的额外提升。
- **吞吐量 vs 延迟**：DualPath 在大幅提升吞吐量的同时，保持了与基准方案相当的 TTFT（首字延迟）和 TPOT（字间延迟），
  证明了流量隔离机制的有效性。
- **局限性**：在小模型或短上下文场景下，I/O 带宽不再是主要矛盾，DualPath 的优势会有所收窄。
