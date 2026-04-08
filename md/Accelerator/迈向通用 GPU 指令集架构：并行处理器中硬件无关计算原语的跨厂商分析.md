# 2603.28793v1.pdf - Toward a Universal GPU Instruction Set Architecture: A Cross-Vendor Analysis of Hardware-Invariant Computational Primitives in Parallel Processors

## 中文标题：迈向通用 GPU 指令集架构：并行处理器中硬件无关计算原语的跨厂商分析

### 中文摘要
本文提出了首个系统性的跨厂商 GPU 指令集架构（ISA）分析，涵盖了 NVIDIA（PTX v1.0 至 v9.2，从 Fermi 到 Blackwell）、
AMD（RDNA 1-4 和 CDNA 1-4）、Intel（Gen11, Xe 系列）以及 Apple（G13，逆向工程）四大主流厂商。通过对超过 5,000 页的官方
手册、白皮书、专利和社区逆向工程成果的研究，作者识别出了在 16 种不同微架构中共同存在的 10 个硬件无关计算原语、6 个参数化
方言以及 6 处真正的架构分歧。基于这些发现，本文提出了一种遵循“薄抽象”原则、受物理约束驱动的厂商中立 GPU ISA 抽象执行模型。
研究表明，GPU 架构的趋同源于物理必然性（如取指成本、存储带宽间隙等）。通过在 NVIDIA T4 和 Apple M1 上的基准测试验证，
该抽象模型在 6 个测试对中的 5 个达到了原生性能，并发现“内波洗牌”（intra-wave shuffle）是确保高性能的必要原语。

### 各章节主旨内容
- **Chapter 1: Introduction** - 分析了 CUDA 导致的软件锁入现状，提出建立类似 ARM 的通用 GPU ISA 的设想及其物理驱动因素。
- **Chapter 2: Background and Related Work** - 回顾了 GPU 硬件现状、现有的 API/编译器移植方案（如 OpenCL, SPIR-V）及局限。
- **Chapter 3: Methodology** - 介绍了研究所涵盖的 16 种微架构数据源及八个维度的分析框架。
- **Chapter 4: Cross-Vendor Analysis** - 核心章节。识别了 10 个不变量（Invariants），并从物理学角度论证了其存在的必然性。
- **Chapter 5: Proposed Abstract Execution Model** - 提出了厂商中立的抽象执行模型，定义了线程层次、内存模型和指令集核心。
- **Chapter 6: Mapping Analysis** - 展示了如何将抽象模型映射到四大厂商的具体架构中。
- **Chapter 7: Experimental Validation** - 通过 GEMM、Reduction 和 Histogram 算子验证了抽象模型的高效性。
- **Chapter 8: Discussion** - 探讨了“薄抽象”原则的优越性以及 Shuffle 指令对 NVIDIA 平台的重要性。
- **Chapter 9: Conclusion** - 总结了通用 GPU ISA 的可行性及其作为计算基础设施的战略意义。

### 中文结论
研究证明，尽管各 GPU 厂商在市场竞争中采取了不同的技术路线，但在物理约束（能效、面积、延迟）的驱动下，其核心计算原语
已呈现出高度收敛。通过识别这些硬件无关的不变量，可以构建一个既能保持高性能、又能跨厂商兼容的通用 GPU ISA。这种“薄抽象”
模型避开了过度设计的陷阱，通过查询硬件常数而非硬编码参数，实现了 50%-80% 到接近 100% 原生性能的跨越，为打破单一厂商的
软件生态垄断提供了技术路径。

---

# 章节详细解读

## 一、 引言：打破 CUDA 垄断的物理路径

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：当前 GPU 领域被 NVIDIA 的 CUDA 垄断。虽然存在 OpenCL, SYCL 等跨平台 API，但它们在 API/IR 层抽象，
  故意隐藏了硬件执行模型。这导致高性能代码必须针对每个厂商进行手工调优，跨平台损耗通常在 20%-50% 之间。
- **核心动机**：借鉴 ARM 在 CPU 领域的成功经验。ARM 并不提供一个屏蔽硬件差异的 API，而是定义了一个精确的执行模型。
  作者认为 GPU 架构已经足够成熟，可以定义类似的“通用契约”。
- **设计目标**：寻找所有 GPU 架构共享的、由物理必然性驱动的底层原语，并将其形式化为通用 ISA。

## 二、 跨厂商分析：物理约束下的架构收敛

### 1. 硬件无关原语 (Hardware-Invariant Primitives)
作者识别了 10 个核心不变量，并给出了其物理合理性（Physical Rationale）：
- **Lockstep Thread Group (Warp/Wavefront)**：存在的原因是现代工艺节点下，取指（Fetch）能耗比单道算术运算高 10-100 倍。
  因此，跨 W 个通道分摊取指成本是能效的必然选择。
- **Mask-based Divergence**：在同步执行模型中，掩码控制是唯一不依赖分支预测（耗费大量面积）而保证正确性的机制。
- **Register-Occupancy Trade-off**：公式化表示为 $O = \lfloor F / (R \times W \times w) \rfloor$。这是由 SRAM 面积固定的
  物理现实决定的。
- **Programmer-managed Scratchpad**：因为高速缓存（Cache）无法预测并优化某些并行访问模式，显式数据布局必不可少。

### 2. 参数化方言 (Parameterizable Dialects)
作者指出，有些概念是共通的，但参数不同。在通用 ISA 中，这些应作为“可查询常量”：
- **Wave Width (W)**：NVIDIA 恒定为 32，AMD 在 32/64 间切换，Intel 则在 8-16 之间。
- **Max Registers (R)**：各平台在 128 到 256 之间波动。
- **Scratchpad Size (S)**：从 Apple 的 60KB 到 Intel 的 512KB 不等。

## 三、 提议的抽象执行模型：薄抽象原则

### 1. 设计思想 (Design Philosophy)
- **薄抽象原则 (Thin Abstraction)**：定义硬件“必须做什么”，而不是“怎么做”。不预设 W 的大小，而是允许程序查询。
- **层次化结构**：定义了三层强制结构（Thread, Wave, Workgroup）和一个可选层（Cluster）。

### 2. 关键技术 (Key Technologies)
- **核心规格**：
    - **Compute Unit**：包含多个 Core，每个 Core 驻留多个 Wave，拥有私有 32 位寄存器 R。
    - **内存模型**：定义了严格的层次结构（私有寄存器 -> 工作组 Scratchpad -> 全球设备内存）。
    - **同步机制**：采用 Scoped acquire/release 语义，涵盖 wave, workgroup, device 和 system 四个级别。
- **控制流**：仅支持结构化控制流（if/else, loop, call/return），将底层复杂的掩码栈或执行位管理隐藏在编译器后端。

## 四、 实验验证：通用模型 vs 原生性能

### 1. 实验设计
- **平台对比**：选取了架构差异最大的 NVIDIA T4（独立 GPU，GDDR6）和 Apple M1（统一内存，LPDDR4X）。
- **基准测试**：GEMM（计算密集）、Reduction（带宽密集）、Histogram（原子操作密集）。

### 2. 核心发现与权衡 (Trade-offs)
- **GEMM 表现**：在 NVIDIA 上，抽象模型达到 0.91 TF，竟然超过了原生手写的 0.72 TF（126%）。
  - **分析**：原生实现使用了 TILE+1 的 bank-conflict 填充，导致了额外内存浪费；而抽象模型由编译器优化，路径更优。
- **Reduction 瓶颈**：在 NVIDIA 上仅为原生性能的 62.5%。
  - **权衡分析**：这是因为原生代码使用了 `__shfl_down_sync` 洗牌指令，而最初的抽象模型通过 Scratchpad 交换数据。
  - **修正方案**：作者据此将 **intra-wave shuffle** 从可选原语提升为**强制原语**。加入 shuffle 后，预期性能可达 95% 以上。

## 五、 讨论：为什么是现在？

### 1. 时代背景 (Timeline)
作者认为建立通用 GPU ISA 的时机已经成熟，原因有三：
- **AI 需求驱动**：单一厂商的主导已成为全球计算基础设施的单点故障风险。
- **地缘因素**：多国将自主可控的 AI 计算视为国家优先事项。
- **架构成熟度**：跨厂商 15 年的演进证明，底层原语已基本稳定。

### 2. 对现有标准的关系
- 本模型并非要取代 SPIR-V 或 OpenCL，而是补充它们。
- **区别**：现有标准规定了“如何与 GPU 对话”，而本研究规定了“GPU 本质上是什么”。这种从硬件定义的视角有助于构建
  真正高性能的编译器后端。
