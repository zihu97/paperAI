# 2603.12831v1.pdf - Serving Hybrid LLM Loads with SLO Guarantees Using CPU-GPU Attention Piggybacking

## 中文标题：基于 CPU-GPU 注意力搭便车机制的带 SLO 保障的混合 LLM 负载服务系统

### 中文摘要
在共享集群中部署多种大型语言模型（LLM）服务已成为常态。虽然服务共存能提高资源利用率，但由于资源竞争，延迟敏
感型（LS）服务面临严重的 SLO（服务水平目标）违约风险，而尽力而为型（BE）服务也因 GPU 内存限制导致服务容量受
限。现有的系统通常采用粗粒度的资源预留（如预留显存空间）来缓解干扰，但这不仅难以保证 LS 服务的 SLO，也极大压
缩了 BE 服务的吞吐空间。


本文提出了 OmniServe，一个能同时高效利用 CPU 和 GPU 资源来优化混合负载性能的 LLM 服务系统。OmniServe 的核
心是“注意力搭便车（Attention Piggybacking）”机制，它将 BE 服务的注意力计算（Attention）动态卸载到 CPU 执
行，并允许 GPU 在不等待 CPU 结果的情况下继续后续推理，随后再通过层级批处理（layer-wise batching）将 CPU 的计
算结果“搭便车”合并回 GPU 流中。此外，系统引入了基于精确延迟建模的动态批处理控制策略，以确保 LS 服务的 SLO。
实验表明，与现有最优系统相比，OmniServe 将 LS 服务的 SLO 达成率提升了 1.48 倍，BE 服务的吞吐量提升了高达 9.85
倍。

### 各章节主旨内容
- **Chapter 1: Introduction** - 阐述混合 LLM 负载共存的背景，指出 GPU 计算和内存资源竞争是核心挑战，并概述
  OmniServe 的主要技术创新与性能收益。
- **Chapter 2: Background and Motivation** - 分析 LLM 推理的 Dense 和 Attention 计算特性。通过实验揭示现有系
  统（如 Llumnix）在处理 LS 和 BE 混合负载时，由于粗粒度控制导致的 LS 延迟抖动和 BE 饥饿问题。
- **Chapter 3: OmniServe System** - 详细介绍系统的核心设计，包括注意力搭便车机制、异步 CPU-GPU 交互逻辑、精
  细化的延迟建模以及面向混合负载的在线调度算法。
- **Chapter 4: System Implementation** - 描述系统的技术实现细节，如利用 OpenMP 和 AVX-512 加速 CPU 注意力计
  算、在 GPU 显存中实现低开销通信队列、残差连接管理以及对 vLLM 调度器的修改。
- **Chapter 5: Experimental Evaluation** - 在 A100 GPU 和 Intel Xeon CPU 环境下进行端到端测试，验证系统在不同
  模型（Yi-34B, Llama-2-70B）和数据集下的 SLO 保障能力和吞吐提升。
- **Chapter 6: Discussion** - 探讨 OmniServe 在多优先级支持及更灵活的 CPU 卸载设计方面的扩展潜力。
- **Chapter 7: Related Work** - 综述现有的 CPU 辅助推理、SLO 导向的 LLM 服务系统以及异构设备管理的研究进展。
- **Chapter 8: Conclusion** - 总结全篇工作，强调 CPU-GPU 协同在解决混合负载干扰方面的独特性。

### 中文结论
OmniServe 通过创新的“注意力搭便车”机制，成功打破了传统 LLM 服务系统对 GPU 资源的过度依赖。它巧妙地利用了数据中
心内常被忽视的 CPU 算力和内存，通过解耦计算流并实施异步协作，在不损害 LS 服务实时性的前提下，极大地释放了 BE 服
务的吞吐潜力。该研究为异构算力环境下的多级别服务保障提供了一种兼具高性能与高灵活性的系统架构参考。

---

# 章节详细解读

## Chapter 2: Background and Motivation

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：在大规模数据中心，LS 服务（如实时对话）和 BE 服务（如后台离线批处理）常共用 GPU 集群。现有系统
  （如 Llumnix）主要依赖“显存预留”来实现隔离，但这种做法忽略了 **计算资源（SM 占用、内存带宽）的动态竞争**。实验
  发现，即便固定显存，BE 请求的长度波动仍会导致 LS 服务的每 Token 延迟增加 1.47 倍，且 BE 服务在高负载下极易因
  GPU 资源被 LS 占满而发生饥饿。
- **核心动机**：作者观察到两个关键机会：(1) **CPU 资源空闲**。在 GPU 推理任务中，CPU 主要负责控制流，大部分核心处
  于空闲状态，且 CPU 拥有巨大的内存空间，适合存放 BE 服务的 KV Cache。(2) **计算特性差异**。Attention 模块是访存
  密集型的，而 MLP 等 Dense 模块是计算密集型的。将 BE 的 Attention 卸载到 CPU 可以缓解 GPU 带宽竞争。
- **设计目标**：利用空闲 CPU 算力和显存来承载 BE 负载，同时必须解决 CPU 和 GPU 之间巨大的计算性能鸿沟（实验显示
  Dense 计算上 GPU 比 CPU 快 498 倍），避免 GPU 因等待 CPU 结果而产生阻塞。

### 2. 关键技术 (Key Technologies)
- **干扰量化分析**：论文对 Llama-70B 进行了细化分解。发现随着 BE 请求增加，Attention 和 MLP 的延迟均呈梯度上升，
  尤其是 Attention 模块，由于其不涉及模型权重读取，其延迟完全受限于显存带宽竞争。
- **CPU-GPU 性能鸿沟量化**：通过对 Prefill 和 Decode 阶段的详细测试（见 Table 1），明确了 CPU 处理 Attention 的
  速度虽然慢于 GPU，但在小 Batch 场景下差距可接受（如单请求仅差 2.34 倍），这为 Attention 卸载提供了可行性基础。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **对比分析**：传统 offloading 方案倾向于将整个层或模型卸载到 CPU，这会引入严重的 PCIe 传输瓶颈。OmniServe 选择
  **选择性卸载 Attention**，保留计算密集的 Dense 模块在 GPU 上，这是一种更精细的计算划分（Partitioning）。

## Chapter 3: OmniServe System

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：如果同步卸载，GPU 必须等待 CPU 完成 Attention 计算才能进行后续的 Proj 模块计算，由于 CPU 较慢，
  这会直接拖慢 GPU 的整体步调，抵消卸载收益。
- **核心动机**：提出 **“搭便车（Piggybacking）”**。在 $l$ 层的计算中，GPU 将 BE 的 QKV 向量发给 CPU 后，不原地
  等待，而是继续完成 LS 服务的后续计算及其他层的迭代。等 CPU 处理完 $l$ 层的 Attention 后，GPU 在之后的某次迭代
  中，当再次执行到 $l$ 层时，顺带（搭便车）把 CPU 返回的结果合并进来。

### 2. 关键技术 (Key Technologies)
- **注意力搭便车机制（Attention Piggybacking）**：
    - **卸载（Offloading）**：在 $l$ 层完成 QKV 计算后，BE 请求的中间张量传给 CPU。
    - **异步交互**：GPU 计算流和 CPU 计算流彻底解耦。GPU 无需应用传统的 Token 级连续批处理同步。
    - **后注意力搭便车（Post-Attention Piggybacking）**：CPU 结果返回后，GPU 使用“层级批处理（Layer-wise 
      Batching）”技术。在执行 $l$ 层的 Dense 模块（如 `proj`）时，将原本 LS 的结果与 CPU 返回的 BE 结果拼接。
- **精确延迟模型**：为了保障 SLO，系统内置了 Profiler。它对 Dense 模块和 Attention 模块分别建模。Dense 模块建模
  考虑了 GPU 的硬件特性（如线程块分配导致的“梯形”延迟增长，见 Alg 1）；Attention 模块则根据计算负载线性建模。
- **在线调度策略**：
    - **优先级控制**：LS Decoding > LS Chunk Prefill > BE Chunk Prefill > BE Decoding。
    - **准入控制（Admission Control）**：对于新来的 LS 请求，系统会模拟其加入后的整体延迟，若预估会导致现有请求
      违约，则提前拒绝，避免“全盘崩溃”。

### 3. 实现细节 (Implementation Details)
- **状态参数追踪**：调度器实时监控 $c_{PA}(t)$（预处理 Attention 负载）、$c_{DA}(t)$（解码 Attention 负载）、
  $g(t)$（解码请求数）等参数，利用二分查找快速求解最优的 BE 批处理大小。

## Chapter 4: System Implementation

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：CPU 计算慢、PCIe 通信开销、残差连接（Residual Connection）的一致性保证、多 GPU 环境下的资源分配。
- **设计目标**：实现低延迟、高并发的 CPU 注意力计算引擎，并确保推理结果的数学等价性。

### 2. 关键技术 (Key Technologies)
- **CPU 加速引擎**：使用 OpenMP 进行多核并行化，并调用 Intel AVX-512 指令集进行向量化加速。在多 GPU 主机上，为每
  个 GPU Worker 分配私有的 CPU 核心和内存分区，减少跨 Socket 竞争。
- **显存驻留队列（GPU-resident Queues）**：为了极速通信，OmniServe 将 CPU Attention 的输入输出队列直接实现在
  GPU 显存中（作为特殊的 Tensor），通过头尾指针维护。CPU 利用 CUDA IPC 和 MPS 服务直接读写显存队列，完全不阻塞 
  GPU 的计算流。
- **残差管理（Residual Management）**：这是保证正确性的关键。由于 Attention 被卸载，而残差连接需要将 
  Attention 之前的输入与 Attention 的输出相加。系统设计了 **Residual Store**，BE 请求在进入 QKV 模块前将其激活
  值存入该 Store，由 `req_id` 和 `layer` 索引。等到 CPU 返回结果并完成 `out_proj` 后，再从 Store 中取出并合并。

### 3. 实现细节 (Implementation Details)
- **分布式 CPU 注意力**：支持跨机扩展。当本地主机内存存满 BE 的 KV Cache 后，通过 RAY 框架将任务分发给远端的 
  CPU-only 服务器，进一步提升系统的水平扩展能力。
- **调度器重写**：基于 vLLM 调度器进行深度定制，将原有的单队列调度改为支持 LS/BE 分级、支持异步搭便车反馈的复杂逻辑。

## Chapter 5: Experimental Evaluation

### 1. 实验设置 (Experiment Setup)
- **硬件**：1 台 GPU 服务器（4x A100 80GB）+ 4 台 CPU 服务器（Intel Xeon Gold 6342）。
- **模型**：Yi-34B (TP=2), Llama-2-70B (TP=4)。
- **基准方案**：Baseline A (Llumnix + CPU vLLM)、Baseline B (NEO，另一种 CPU-GPU 协同方案)、Baseline C 
  (Sarathi-Serve，最优的 GPU 单体方案)。

### 2. 核心结论
- **SLO 保障（LS 服务）**：OmniServe 的 SLO 达成率与单跑 LS 的 Baseline C 几乎持平，且显著优于 Llumnix。这是因
  为其精细的延迟建模防止了过度批处理，且卸载 BE Attention 减少了对 LS 带宽的抢占。
- **吞吐提升（BE 服务）**：在重负载下，OmniServe 的 BE 吞吐量比 Baseline A 高出 **9.85 倍**。这主要归功于两点：
  (1) 利用了 4 台 CPU 主机的海量内存存放 KV Cache；(2) 异步搭便车机制消除了 GPU 的等待时间。
- **网络敏感度**：即使在 10 Gbps 的普通网络下，由于系统只传输中间张量（中间 Token 向量）而非完整的模型权重或庞
  大的 KV Cache，通信开销非常低，依然能保持高效的吞吐。
- **消融实验**：验证了准入控制对维持 TTFT SLO 的重要性（提升了 43.3% 的合规率），以及 Residual Store 和队列操作
  引入的额外延迟极低（微秒级），证明了设计的工业实用性。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **开销分析**：主要的额外开销在于 CPU 计算带来的结果返回延迟。但论文证明，通过延迟搭便车，这一延迟被隐藏在 GPU 
  的正常流水线中，只要 BE 任务的实时性要求不高，这种“以延迟换吞吐”的策略在 BE 场景下是极优的选择。
