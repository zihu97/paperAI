# 2512.22219v1.pdf - Mirage Persistent Kernel: A Compiler and Runtime for Mega-Kernelizing Tensor Programs

## 中文标题：Mirage 持久化内核：张量程序大内核化的编译器与运行时系统

### 中文摘要
本文介绍了 Mirage 持久化内核（Mirage Persistent Kernel, MPK），这是首个能够自动将多 GPU 模型推理转换
为单个高性能“大内核”（mega-kernel）的编译器和运行时系统。MPK 引入了一种流式多处理器（SM）层级的图
表示（tGraph），该表示能在 SM 粒度捕捉数据依赖，从而实现跨算子软件流水线、细粒度内核重叠等以往无法
实现的 GPU 优化。MPK 编译器将张量程序降低为高度优化的 SM 级任务图，并为所有任务生成优化的 CUDA 实现；
同时，MPK 内核并行运行时在单个大内核中使用去中心化的 SM 调度来执行这些任务。实验表明，MPK 在 LLM 服务
场景中显著优于现有的“每个算子一个内核”系统，将端到端推理延迟降低了高达 1.7 倍，性能逼近硬件极限。

### 各章节主旨内容
- **Chapter 1: Introduction** - 阐述了当前“算子即内核”模式在跨算子优化、计算通信重叠及启动开销方面的局限性
，引出 MPK 的大内核方案。
- **Chapter 2: Background** - 回顾了 GPU 编程模型（SM、线程块、内核屏障）以及内核融合与大内核技术的发展现状。
- **Chapter 3: SM-Level Graph Representation** - 介绍了 tGraph 表示法，通过任务和事件在 SM 维度描述计算与同步
。
- **Chapter 4: The MPK Compiler** - 详述了 tGraph 的生成、依赖分析、事件融合、图规范化及线性化等编译器技术。
- **Chapter 5: In-Kernel Parallel Runtime** - 介绍了内核中并行运行时的架构，包括调度器/工作者划分、混合任务
启动（JIT/AOT）及分页共享内存。
- **Chapter 6: Evaluation** - 在 A100/H100/B200 上测试了多种 LLM，证明了 MPK 在延迟和吞吐量上的巨大优势。
- **Chapter 7: Related Work** - 对比了手动设计的内核、ML 编译器以及现有的大内核研究工作。
- **Chapter 8: Conclusion** - 总结了 MPK 的贡献，指出其为构建高性能编译器驱动的推理系统开辟了新路径。

### 中文结论
MPK 是第一个将多 GPU 模型推理自动转换为完全融合的大内核的编译器和运行时系统。通过引入 SM 级任务图和内
核并行运行时，MPK 克服了“每个算子一个内核”模式的根本限制，实现了跨算子软件流水线、细粒度的计算通信重叠
，并消除了内核启动和 CPU 端调度的开销。评估结果显示，MPK 使 LLM 服务的延迟接近硬件性能下限，并在不同模型
和 GPU 架构上显著提升了吞吐量。

---

# 章节详细解读

## 1. Introduction & Background

### 1.1 设计思想 (Design Philosophy)
- **痛点分析**：现有的深度学习系统采用“每个算子一个内核”的任务执行模式。这种模式存在三大痛点：
    1. **内核屏障（Kernel Barriers）**：GPU 在连续启动内核时会强制插入屏障，导致算子只能顺序执行，无法实现
       跨算子的软件流水线（Software Pipelining）。
    2. **粗粒度依赖**：依赖关系仅在算子级别捕获。例如，即便 All-Reduce 算子只需要前一个矩阵乘法（MatMul）的
       部分结果即可开始，系统也必须等待整个 MatMul 完成，导致计算与通信无法进行细粒度的重叠。
    3. **启动开销**：每次推理迭代需要启动成百上千个内核，CUDA Graphs 虽然能缓解但灵活性不足，且仍受限于 CPU
       端调度。
- **核心动机**：将所有计算和通信融合进一个“持久化内核”（Persistent Kernel/Mega-Kernel）。在内核启动后，
  GPU SM 之间通过设备内存同步，持续处理整个模型的所有层，从而彻底消除内核切换开销并释放细粒度优化空间。

### 1.2 关键技术 (Key Technologies)
- **软件流水线与重叠**：对比图 2 和图 3。传统模式下算子间存在“气泡”（Bubbles），而 MPK 允许 SM 1 在执行
  Task A 的计算时，SM 2 已经开始 Task B 的预取或计算。
- **细粒度内核重叠**：以 LLM 中常见的 MatMul 后接 All-Gather 为例，MPK 识别出 All-Gather 的每个块只依赖
  MatMul 的特定块，从而允许它们在不同 SM 上并行执行。

## 2. SM-Level Graph Representation (tGraph)

### 2.1 设计思想 (Design Philosophy)
- **核心动机**：为了打破内核边界，需要一种比传统算子图（Computation Graph）更细粒度的表示。作者提出了
  tGraph，将操作拆解到单个 SM 能够独立处理的任务（Task）级别。

### 2.2 关键技术 (Key Technologies)
- **任务（Task）与事件（Event）**：
    - **Task**：在单个 SM 上执行的计算或通信单元（如 MatMul 的一个 Tile 或一个数据的搬运）。
    - **Event**：跨任务的同步点。只有当一个事件的所有前驱任务完成时，该事件才会被激活，进而启动其后继任务。
- **与 CUDA Graphs 的对比**：CUDA Graphs 捕捉的是内核启动顺序（Stream Semantics），而 tGraph 显式建模了
  SM 之间的 intra-operator（算子内）和 cross-operator（跨算子）依赖。

## 3. The MPK Compiler

### 3.1 设计思想 (Design Philosophy)
- **痛点分析**：手动编写大内核需要极高的 GPU 编程经验且难以跨模型通用。MPK 旨在提供一种编译器驱动的自动化
  方案，用户只需几行 PyTorch 代码即可生成大内核。

### 3.2 关键技术 (Key Technologies)
- **算子分解（Operator Decomposition）**：将张量算子划分为多个不相交的任务，默认任务数量与 GPU SM 数量
  成比例，以保证负载均衡。
- **依赖分析与事件融合**：
    - **细粒度分析**：通过分析输入输出张量的重叠区域（Overlap）来插入同步事件。
    - **Successor-set & Predecessor-set Fusion**：合并具有相同前驱或后继的任务/事件，消除冗余同步，减小图
      的复杂度和内存占用（见算法 4.1 和 4.2）。
- **图规范化（tGraph Normalization）**：为了降低运行时的调度开销，通过引入“空任务”（Dummy Tasks）确保
  每个任务最多只有一个前置和后置事件。
- **线性化（Linearization）**：使用 BFS 算法将图转换为连续的任务列表，使得一个事件触发的多个任务在内存中
  是连续的，从而可以用（First Task ID, Last Task ID）紧凑表示。

### 3.3 实现细节 (Implementation Details)
- **超级优化（Superoptimization）**：MPK 利用 Mirage 编译器对每个 SM 任务进行线程块级的搜索优化，自动生成
  高性能的 CUDA 代码，包括软件流水线、寄存器重用和访存布局优化。

## 4. In-Kernel Parallel Runtime

### 4.1 设计思想 (Design Philosophy)
- **核心动机**：在大内核内部实现高效的任务分发，避免全局锁和昂贵的 CPU-GPU 交互。

### 4.2 关键技术 (Key Technologies)
- **角色划分**：将 SM 划分为 **Worker**（工作者，执行计算/通信任务）和 **Scheduler**（调度器，维护事件队列
  并分发任务）。调度器以 Warp 为粒度运行，每个 SM 托管 4 个调度器 Warp。
- **混合任务启动（Hybrid Task Launch）**：
    - **JIT 模式**：任务完成后通知调度器，调度器激活后续任务。适用于执行时间高度不确定（如数据相关的
      Attention）的算子，能动态均衡负载。
    - **AOT 模式**：在内核启动前就将任务预分配给特定 Worker。Worker 完成前驱后直接通过内存标志触发后续任务
      ，减少了与调度器的通信延迟。适用于耗时稳定的算子（如线性层）。
- **分页共享内存抽象（Paged Shared-Memory）**：
    - 传统模式下每个内核独占 SM 的 Shared Memory。
    - MPK 将 Shared Memory 划分为固定大小的页。任务可以动态申请和释放页，这使得 SM 可以在当前任务计算时
      ，利用剩余空间为下一个任务预取数据，实现跨任务的软件流水线。

## 5. Evaluation & Case Study

### 5.1 对比与权衡 (Trade-offs & Comparison)
- **性能表现**：在单批次推理中，MPK 相比 SGLang 和 vLLM 提升了 1.0-1.7 倍吞吐量。在 B200 这种通信带宽
  极高的硬件上，MPK 的优势更加明显，因为它能更有效地重叠计算与通信。
- **消融实验**：
    - **跨任务流水线**：为线性层带来了 1.2-1.3 倍的性能提升，甚至超过了高度优化的 cuBLAS 编译内核。
    - **计算通信重叠**：使端到端延迟降低了约 10%。

### 5.2 混合专家模型（MoE）优化
- **Hybrid Workload Balancer**：MoE 的专家负载是不确定的。MPK 在编译时静态划分，在运行时根据 top-k
  softmax 产生的元数据动态细化分配，比纯静态或纯动态（Persistent Grouped-GEMM）方案更优。
- **Fused Gather-GEMM**：利用 Hopper/Blackwell 的 TMA，将数据汇聚（Gather）直接集成到 GEMM 加载阶段，
  消除了独立的 Gather 内核启动和中间调度开销。

## 6. 总结与反思

### 6.1 关键创新点
- **SM 粒度调度**：首次在大内核内部实现了去中心化的 SM 任务调度，解决了大内核扩展性差的难题。
- **自动化大内核化**：通过编译器技术将大内核从“手工艺术品”转变为“通用基础设施”。

### 6.2 优劣势考量
- **优势**：极低的延迟、极高的硬件利用率、易于集成的 PyTorch 接口。
- **开销/限制**：tGraph 规范化会引入少量空任务（实测开销 < 1%）；目前主要针对 LLM 这种深层顺序模型效果
  最佳，对于非常“宽”的并行计算图，收益可能受限。

---
