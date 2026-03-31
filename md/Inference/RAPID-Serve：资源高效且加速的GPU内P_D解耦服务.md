# RAPID-Serve：资源高效且加速的GPU内P/D解耦服务

---

## 第一部分：论文整体骨架

### 1. 论文基本信息
*   **英文标题**：RAPID-Serve: Resource-efficient and Accelerated P/D Intra-GPU Disaggregation
*   **中文标题**：RAPID-Serve：资源高效且加速的GPU内预填充/解码（P/D）解耦服务
*   **作者**：Amna Masood, Pratishtha Gaur, Nuwan Jayasena (Advanced Micro Devices - AMD)
*   **发布日期**：2026年1月16日 (arXiv:2601.11822v1)

### 2. 中文摘要
目前，LLM推理服务中广泛采用的两种技术是混合批处理（Hybrid Batching）和解耦服务（Disaggregated Serving）。
混合批处理通过将不同请求的预填充（Prefill）和解码（Decode）token合并到同一批次中来提高资源利用率和吞吐量，但代价是
增加了每个token的延迟。相比之下，解耦服务通过在不同硬件上分离计算密集型的预填充阶段和带宽受限的解码阶段，以此来
优化服务水平目标（SLO），但这会导致资源利用率不足和KV缓存传输开销。

为了解决这些技术的局限性，本文提出了 **RAPID-Serve**：一种在同一GPU上并发执行预填充和解码的技术。它旨在满足延迟
SLO的同时，保持高吞吐量和高效的资源利用率。此外，本文还提出了 **自适应资源管理（Adaptive Resource Management）**，
用于运行时计算资源的动态分配，并利用 AMD Instinct™ GPU 上的 **CU Masking（计算单元掩码）** 功能进行硬件隔离。

实验表明，与最先进的方法相比，RAPID-Serve 在资源受限的环境中表现尤为出色，提供了高达 4.1 倍（平均 1.7 倍）的无
约束吞吐量提升，以及 32 倍甚至更高（平均 4.9 倍）的 SLO 约束下吞吐量（Goodput）提升。

### 3. 各章节主旨内容总结

*   **1. Introduction (引言)**
    *   随着LLM应用从简单的聊天转向复杂的推理和多步思考，系统设计目标从单纯的流式传输速度转向高吞吐量的Token生成。
    *   现有的混合批处理会导致请求间的延迟干扰（Inter-Token Latency, ITL），而解耦服务在资源受限（如少量GPU）
        场景下存在KV缓存传输瓶颈和资源浪费。
    *   本文提出RAPID-Serve，核心洞察是：通过在同一GPU上并发执行预填充和解码，可以同时获得可预测的延迟和高吞吐量。

*   **2. Background (背景)**
    *   **LLM Inference**：介绍了推理的两个阶段——计算密集型的Prefill（预填充）和内存带宽密集型的Decode（解码）。
    *   **Hybrid Batching**：介绍了Orca、vLLM、Sarathi等系统如何通过合并批处理来提升吞吐，但指出了其“锁步（lockstep）”
        执行导致的延迟问题。
    *   **Disaggregated Serving**：介绍了DistServe等系统将P/D分离到不同GPU的思路，指出了其在小规模集群中的局限性
        （网络传输开销、内存容量不平衡）。

*   **3. Design Considerations (设计考量)**
    *   **Overheads of Hybrid Batching**：分析了Chunked Prefill（分块预填充）在吞吐量和延迟之间的权衡困境。
    *   **Overheads of Disaggregated Serving**：量化了KV缓存传输的开销（在MI300X上增加了1.4x-1.9x的开销）以及
        内存利用率的不平衡。
    *   **Phase Specific Resource Requirements**：通过实验证明Prefill需要大量计算单元（Compute Units），而Decode
        是带宽受限的，对计算单元数量不敏感，这为两者在同一GPU上共存提供了理论基础。
    *   **Memory Subsystem Interference**：实验表明，并发执行时的内存子系统干扰对性能影响很小（<5%）。

*   **4. RAPID-Serve Implementation (RAPID-Serve 实现)**
    *   详细介绍了系统架构，基于vLLM v0.10.2rc3开发。
    *   **CU Masking**：利用AMD GPU特性，将计算单元在空间上划分给不同的流，实现物理隔离。
    *   **Multiprocessing**：采用多进程架构规避Python GIL锁，实现真正的CPU端并发处理。
    *   **Request Flow**：设计了全异步的请求流，Prefill和Decode进程通过共享内存和队列通信，无锁设计。
    *   **Adaptive Resource Manager**：根据实时负载和SLO动态调整分配给Prefill和Decode的计算单元数量。

*   **5. Evaluation (评估)**
    *   在AMD Instinct MI300X GPU上，使用LLaMA-3 70B和Mixtral 8x7B模型进行评估。
    *   **Throughput & Goodput**：RAPID-Serve在满足SLO的前提下（Goodput），相比混合批处理和解耦服务有数倍提升。
    *   **Tail Latencies**：显著降低了长尾延迟（p95 TTFT和ITL）。
    *   **Resource Utilization**：通过并发执行填补了GPU流水线气泡，显著提升了计算和内存利用率。

*   **6. Related Work (相关工作)**
    *   对比了PODAttention、Nanoflow、DRIFT、Semi-PD等相关工作，指出了RAPID-Serve在解决耦合执行和资源分配方面
        的独特优势。

*   **7. Conclusions (结论)**
    *   RAPID-Serve通过在同一设备上解耦但并发执行P/D阶段，有效解决了现有技术的性能限制。
    *   其无锁、无同步的设计以及自适应资源管理，使其在资源受限场景下成为一种高效的推理服务方案。

### 4. 中文结论
RAPID-Serve 提供了一种有效的解决方案，旨在解决当前 LLM 推理服务系统的性能局限性，同时满足严格的服务水平目标（SLO）。
与混合批处理（Hybrid Batching）将预填充和解码请求耦合在一起（可能导致请求超出延迟 SLO）不同，RAPID-Serve 在同一
阶段内重叠执行预填充和解码，而不将它们合并。这使得 RAPID-Serve 能够在满足 SLO 的同时保持高吞吐量。此外，RAPID-Serve
还避免了传统解耦服务（Disaggregated Serving）中的 KV 缓存传输开销和内存容量利用率不足的问题。评估结果表明，
RAPID-Serve 提供了显著的性能提升：无约束吞吐量最高提升 4.1 倍，Goodput（满足 SLO 的有效吞吐量）提升高达 32 倍。
这些结果展示了并发执行预填充和解码以有效利用可用 GPU 资源的功效。

---

## 第二部分：指定章节详解

### 1. 核心设计思想与动机 (基于 Chapter 1 & 3)

**为什么现有的方案不够好？**
传统的 **混合批处理 (Hybrid Batching)** 就像是让一辆公交车（GPU）同时载着快客（Decode请求）和慢客（Prefill请求）。
为了照顾慢客上车（处理Prefill），整辆车必须停下来，这导致快客的体验很差（延迟增加）。虽然可以通过“分块上车”
（Chunked Prefill）来缓解，但这会降低整体运力（吞吐量），且依然存在互相等待的“锁步”效应。

**解耦服务 (Disaggregated Serving)** 则是派了两辆车，一辆专门拉慢客，一辆专门拉快客。这解决了互相等待的问题，
但带来了新问题：
1.  **换乘成本 (KV Transfer)**：慢客处理完变成快客时，需要从一辆车转移到另一辆车，这需要时间和带宽。
2.  **空载浪费 (Underutilization)**：如果今天慢客很少，慢车就空着；或者快车满了，慢车却帮不上忙。特别是对于
    只有少量GPU的节点，这种静态划分极其浪费。

**RAPID-Serve 的洞察**
RAPID-Serve 发现：
1.  **资源需求互补**：Prefill 是“计算密集型”（吃算力），Decode 是“带宽密集型”（吃显存带宽，算力常有富余）。
2.  **空间隔离可行**：现代 GPU（如 AMD MI300X）允许将计算单元（CU）物理切分。

因此，RAPID-Serve 的核心思想是 **“车内隔断”**：在同一辆车（GPU）上，划出一部分座位（CUs）专门给 Prefill，
另一部分给 Decode。它们在同一块显存上跑，既不需要“换乘”（零 KV 传输开销），又互不干扰（硬件隔离），还能互相填补
空闲时间。

### 2. 关键实现技术 (基于 Chapter 4)

#### 2.1 硬件级隔离：CU Masking (计算单元掩码)
这是 RAPID-Serve 的技术基石。
*   **机制**：利用 AMD Instinct GPU 的驱动特性，可以为每个命令队列（Command Queue）关联一个位掩码（Bit Mask）。
    每一位对应一个具体的计算单元（CU）。
*   **效果**：当内核（Kernel）被提交到带有掩码的队列时，它的工作组（Workgroups）只会被调度到指定的 CU 上执行。
    这实现了 Prefill 和 Decode 内核在硬件层面的空间隔离，避免了争抢计算资源导致的抖动。
*   **透明性**：这对上层应用是透明的，不需要修改模型代码或 CUDA/HIP 内核代码。

#### 2.2 软件架构：多进程与无锁设计
为了实现真正的并发，RAPID-Serve 必须克服 Python 的 GIL（全局解释器锁）限制。
*   **多进程 (Multiprocessing)**：系统启动两个独立的进程，一个负责 Prefill，一个负责 Decode。
    *   **Prefill 进程**：专注于处理新请求的 Prompt 阶段。
    *   **Decode 进程**：专注于生成后续 Token。
*   **共享内存**：虽然进程独立，但模型权重和 KV Cache 是驻留在同一 GPU 显存中的。它们通过 IPC（进程间通信）句柄
    共享显存地址，因此不需要数据拷贝。
*   **KV Cache 管理权**：为了避免两个进程同时操作 KV Cache 分配表导致需要加锁，RAPID-Serve 采用了一个巧妙的设计：
    **只有 Decode 进程拥有 KV Cache 管理器**。
    *   当新请求到来，Decode 进程先计算需要的块数并分配好。
    *   将分配好的 Block IDs 传给 Prefill 进程。
    *   Prefill 进程只负责往这些 Block 里填数据，填完通知 Decode。
    *   这种设计实现了 **Zero-Lock（零锁）** 操作，消除了同步开销。

#### 2.3 异步调度 (Async Scheduling)
为了极致的性能，CPU 的调度逻辑不能拖累 GPU 的执行。
*   **预判执行**：调度器（Scheduler）总是比 GPU 执行“快一步”。在 GPU 生成当前 Token 时，调度器已经开始准备下一个
    时间步的批次数据。
*   **占位符**：对于正在生成的 Token，调度器假设它会成功生成，并作为下一轮的输入占位。
*   **收益**：这种流水线化的调度完全隐藏了 CPU 的 Python 逻辑开销，最大化了 GPU 的利用率。

#### 2.4 自适应资源管理器 (Adaptive Resource Manager)
静态划分 CU（例如 50% 给 Prefill，50% 给 Decode）并不总是最优的。
*   **动态调整**：系统包含一个运行时组件，监控当前的 Decode 负载和 SLO 满足情况。
*   **策略**：
    *   **低负载模式 (Overallocation)**：当 Decode 请求少，预留的 CU 用不完时，允许 Prefill 使用所有 CU，
        或者允许两者竞争。因为此时 GPU 硬件调度器能很好地填补空隙，且不会违反 SLO。
    *   **高负载/严格模式**：当 Decode 压力大，ITL（Token间延迟）接近 SLO 红线时，强制执行严格的 CU 隔离。
        确保 Decode 获得足以维持 SLO 的最小 CU 数量，剩下的给 Prefill。
    *   这种机制保证了在任何负载下，系统都能在“保延迟”和“冲吞吐”之间找到最佳平衡点。

### 3. 实验评估与成效 (基于 Chapter 5)

*   **测试平台**：单节点 8x AMD Instinct MI300X GPU。
*   **对比基准**：
    *   **vLLM (Hybrid Batching)**：代表目前主流的开源推理引擎。
    *   **DistServe (Disaggregated)**：代表学术界最先进的分离式架构。
*   **关键数据**：
    *   **Goodput (有效吞吐量)**：定义为在满足延迟 SLO（如 TTFT < 1s, ITL < 100ms）前提下的最大请求速率。
        RAPID-Serve 在此指标上实现了 **32倍** 的提升（对比 Chunked Prefill）。这说明在必须保证用户体验的场景下，
        RAPID-Serve 能够承载的并发量是巨大的。
    *   **延迟**：95分位的首字延迟（TTFT）降低了 **220倍**。这是因为 Prefill 不再被切分成小块排队，也不需要跨设备传输。
    *   **资源利用率**：计算单元利用率提升了 77%-111%，显存利用率提升了 37%（通过共享 KV Cache）。

### 总结
RAPID-Serve 是一项针对 AMD GPU 架构优化的系统级创新。它打破了“要么混合批处理，要么硬件分离”的二元对立，
证明了通过精细的软硬件协同设计（CU Masking + 多进程无锁架构），可以在单设备内部实现完美的 Prefill/Decode 解耦。
这对于资源受限的边缘侧推理或私有化部署场景具有极高的实用价值。
