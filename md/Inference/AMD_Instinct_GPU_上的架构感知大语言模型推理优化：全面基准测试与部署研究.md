# 2603.10031v1.pdf - Architecture-Aware LLM Inference Optimization on AMD Instinct GPUs: A Comprehensive Benchmark and Deployment Study

## 中文标题：AMD Instinct GPU 上的架构感知大语言模型推理优化：全面基准测试与部署研究

### 中文摘要
在尖端规模下，大语言模型（LLM）推理需要模型架构、硬件能力和推理系统配置的协同优化。本文在 AMD Instinct 
MI325X GPU 上对生产级 LLM 推理进行了系统性的跨架构评估。研究涵盖了 2350 亿到 1 万亿参数规模的四个前沿模型，
涉及三种架构家族（MoE+MLA、Dense+GQA、MoE+GQA）。通过在拥有 2 TB HBM3e 的 8-GPU 集群上使用 vLLM 
v0.14.1 进行基准测试，结果表明架构感知优化至关重要：MLA 模型在当前 ROCm 栈上需要特定的块大小且不支持 KV 
缓存卸载，而 GQA 模型则能从中受益。AMD AI Tensor Engine (AITER) 是实现竞争性 MLA 推理吞吐量的必要条件。
研究还验证了活跃参数量是驱动吞吐量的核心因素，并确认了高并发下的主要瓶颈是内存带宽。

### 各章节主旨内容
- **Chapter 1: Introduction** - 阐述 LLM 推理从原型到生产基础设施的演进，指出在 AMD 硬件上缺乏系统性基准
  测试的现状，并概述本文在跨架构评估和优化策略方面的五大核心贡献。
- **Chapter 2: Related Work** - 综述了 LLM 服务系统（如连续批处理、分级调度、PagedAttention）、各种注意
  力机制（GQA、MLA）、混合专家模型（MoE）、硬件加速技术以及量化方法。
- **Chapter 3: System Architecture and Hardware Platform** - 详细描述了基于 CDNA 3 架构的 MI325X 硬件
  特性，以及 vLLM 推理引擎的核心机制（如 V1 引擎、分块预填充、多步调度）。
- **Chapter 4: Optimization Techniques** - 深入探讨了针对不同架构的量化策略（FP8、INT4 QAT）、KV 缓存
  管理、AITER 内核优化、张量并行（TP）配置以及并发调度参数的调优。
- **Chapter 5: Experimental Methodology** - 介绍了包含热身、基线、缩放、压力和饱和五个阶段的渐进式基准
  测试框架，以及涵盖并发缩放、长输出生成和多图像视觉负载的实验设计。
- **Chapter 6: Results and Analysis** - 报告了各模型在不同工作负载下的吞吐量、延迟和饱和点。重点分析了
  架构差异对性能的影响，并展示了 AITER 对不同注意机制的加速效果和测量变异性。
- **Chapter 7: Discussion** - 讨论了“一种配置无法适配所有架构”的核心结论，分析了 MI325X 平台的竞争优势，
  识别了内存带宽瓶颈，并指出了单集群配置和模型选择等研究局限性。
- **Chapter 8: Conclusion and Future Work** - 总结了研究发现，并提出了多节点缩放、更广泛模型覆盖和自动
  化架构感知配置等未来研究方向。

### 中文结论
本文通过详尽的实验证明，AMD Instinct MI325X 是万亿级参数模型推理的高效平台。然而，实现最佳性能必须
根据模型架构（特别是注意力机制和专家结构）定制推理配置。研究强调，简单的“一刀切”配置会导致严重的性能
损失甚至运行时错误。未来的推理系统应具备架构感知能力，自动根据模型特性选择最优的算子内核、缓存策略和
并行度。

---

# 章节详细解读

## 第一章：引言 (Introduction)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：随着 LLM 规模从数十亿扩展到万亿，高效推理成为部署的关键瓶颈。虽然训练侧的缩放定律（Scaling 
  Laws）已有深入研究，但在生产环境中如何同时优化吞吐量、延迟和硬件利用率仍缺乏系统性探索，尤其是在 
  AMD Instinct 加速器生态系统中。
- **核心动机**：现有的基准测试多集中于 NVIDIA 硬件，或仅针对单一模型。作者认为，由于模型架构（如 MoE、
  MLA、GQA）的多样化，需要一种系统性的、跨架构的评估方法来揭示硬件与软件栈之间的复杂交互。
- **设计目标**：在 AMD 最新的 MI325X GPU 上建立首个学术级的、涵盖万亿参数规模模型的生产级推理基准，并
  提炼出可指导实际部署的优化原则。

## 第三章：系统架构与硬件平台 (System Architecture and Hardware Platform)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：前沿模型（如 DeepSeek V3）对内存容量和带宽提出了极端要求。传统的 GPU 在处理这类模型时，
  往往因为 VRAM 不足而被迫使用昂贵的跨节点通信或 KV 缓存卸载到 CPU，导致延迟剧增。
- **核心动机**：MI325X 提供的 256 GB HBM3e 容量是解决上述痛点的关键硬件基础。设计重点在于如何最大化
  利用 6.0 TB/s 的单卡带宽和 48 TB/s 的集群总带宽。

### 2. 关键技术 (Key Technologies)
- **MI325X (CDNA 3)**：支持硬件级 FP8 矩阵运算，无需专用加速器即可进行量化推理。通过 RCCL 进行高效的 
  GPU 间通信。
- **vLLM 推理引擎**：
    - **PagedAttention**：模拟虚拟内存分页，彻底消除内存碎片，提升 KV 缓存利用率。
    - **V1 引擎与分块预填充 (Chunked Prefill)**：将长提示词切分并与解码步骤交织，防止长请求垄断 GPU，
      维持低序列间延迟。
    - **多步调度 (Multi-Step Scheduling)**：一次调度执行多个解码步骤，显著降低 CPU 侧的调度开销。

### 3. 实现细节 (Implementation Details)
- **环境配置**：ROCm 6.4.2 + RCCL 2.26.6。禁用 NUMA 平衡 (`numa_balancing=0`) 以防止系统迁移内存页导致
  的通信延迟波动。
- **模型镜像**：针对 Kimi-K2.5 使用了 nightly 版本的 vLLM 镜像以支持其特定的架构需求。

## 第四章：优化技术 (Optimization Techniques)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：不同的注意力机制对内核的要求完全不同。例如，MLA 压缩了 KV 缓存，但在当前软件栈上导致
  了特定的对齐和分布限制。
- **核心动机**：通过“架构感知”的优化路径，而非通用的默认设置，来应对不同模型的独特性。

### 2. 关键技术 (Key Technologies)
- **量化策略 (Quantization)**：
    - **FP8 (W8A8)**：在 MI325X 上实现 ~50% 的内存节省。针对 ROCm 引入了 Per-Token Per-Channel (PTPC) 
      缩放以提升精度。
    - **INT4 QAT**：Kimi-K2.5 采用这种更激进的量化方式，使 1T 参数模型能挤进单节点集群。
- **KV 缓存管理**：MLA 架构本质上减少了每 token 的内存需求，但也限制了块大小（必须为 1）。
- **AITER (AMD AI Tensor Engine)**：提供手写优化的 MoE 和注意力内核。
    - **创新点**：针对 CDNA 3 架构调优，声称在 DeepSeek 工作负载下有 2.1 倍的综合加速。

### 3. 权衡与对比 (Trade-offs & Comparison)
- **AITER 的代价**：在提升性能的同时，显著增加了测量的非确定性（变异系数 CoV 增加 2-16 倍）。对于 GQA 
  模型（如 Llama-405B），AITER 的收益仅为 3-5%，与其在 MLA/MoE 上的表现形成鲜明对比。
- **MLA 的限制**：虽然节省内存，但失去了 KV 缓存卸载到 CPU 的灵活性（导致 KeyError 崩溃）。

## 第六章：结果与分析 (Results and Analysis)

### 1. 设计思想 (Design Philosophy)
- **核心动机**：验证“活跃参数量（Active Parameters）而非总参数量决定吞吐量”的假设在生产环境下的真实表现。

### 2. 关键技术 (Key Technologies)
- **渐进式基准框架**：从单请求延迟（Warmup/Baseline）到极高并发（Saturation），系统地识别系统的“崩溃点”。
- **性能指标**：不仅看总吞吐量（Tokens/s），更区分了生成吞吐量（Output Tokens/s）以反映解码速度的差异。

### 3. 实现细节 (Implementation Details)
- **测量数据**：
    - Llama-3.1-405B (405B 活跃) 达到 15,944 tok/s。
    - DeepSeek V3.2 (37B 活跃) 达到 15,343 tok/s。
- **惊人发现**：Qwen3-VL 由于极低的活跃参数量 (22B) 配合高效的 GQA 架构，达到了 47,873 tok/s 的极高性能，
  尽管它还处理了复杂的图像 token。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **瓶颈识别**：所有模型在并发量超过 500 时，吞吐量均不再增长，且 FP8 计算单元的利用率远低于峰值（~14%）。这
  在逻辑上确证了当前大模型推理仍处于强烈的“内存带宽瓶颈（Memory-Bandwidth Bound）”状态。
- **可靠性**：即使在极高负载下，vLLM 的队列机制也保证了 100% 的 HTTP 成功率，而非直接拒绝服务。

---
注：以上分析基于 2026 年技术报告。
