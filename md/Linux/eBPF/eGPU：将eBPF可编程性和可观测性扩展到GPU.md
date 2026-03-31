# eGPU：将 eBPF 可编程性和可观测性扩展到 GPU

---

## 第一部分：论文整体骨架

### 1. 论文基本信息
*   **英文标题**：eGPU: Extending eBPF Programmability and Observability to GPUs
*   **中文标题**：eGPU：将 eBPF 可编程性和可观测性扩展到 GPU
*   **作者**：Yiwei Yang, Tong Yu, Yusheng Zheng, Andrew Quinn
*   **发表会议/时间**：HCDS '25: 4th Workshop on Heterogeneous Composable and Disaggregated Systems (March 2025)

### 2. 中文摘要
在 AI 工作负载和其他计算密集型高性能计算（HPC）应用中，精确的 GPU 可观测性和可编程性对于优化性能至关重要。本文介绍了 **eGPU**，这是第一个通过动态 PTX（Parallel Thread Execution）注入将 eBPF 字节码动态卸载到 GPU 上运行的框架和 eBPF 运行时。eGPU 主要为可观测性而设计，它利用实时 GPU 遥测、基于 eBPF 的动态插桩以及自动化的性能分析，在多个层面（包括内核执行、内存传输和异构计算编排）定位瓶颈，而不会产生显著的开销或中断活跃的 GPU 内核。

通过将 eBPF 程序动态编译为 PTX 片段并直接注入到正在执行的 GPU 内核中，eGPU 提供了细粒度、低开销的插桩，能够与现有的 eBPF 生态系统无缝集成。我们的微基准测试评估表明，eGPU 在提供高分辨率性能洞察的同时，实现了极低的插桩开销。最终，eGPU 为未来的可编程 GPU 基础设施开辟了道路，使其能够动态适应多样化和不断演变的工作负载需求。

### 3. 各章节主旨内容

*   **1. 引言 (Introduction)**
    *   **背景**：数据密集型和计算要求高的工作负载（如大规模机器学习、HPC 模拟、LLM）极其依赖 GPU 加速。随着系统规模扩大，精确的可观测性和低开销插桩对于诊断瓶颈、优化资源至关重要。
    *   **问题**：现有的 GPU 插桩方法（如 NVIDIA CUPTI、NVBit）通常带来巨大的开销或需要侵入式修改，中断 GPU 内核。Linux 的 eBPF 虽然在 CPU 上表现出色（轻量级、沙箱化），但由于 GPU 的异步执行模型、并行线程执行和专用内存层次结构，无法直接应用于 GPU。
    *   **贡献**：提出了 eGPU，通过运行时 PTX 注入将 eBPF 插桩扩展到 GPU 内核。它利用用户态 eBPF uprobes、共享内存映射和实时 GPU 遥测，提供最小运行时开销的细粒度插桩。

*   **2. 背景 (Background)**
    *   **bpftime**：介绍了 bpftime，这是一个用户态 eBPF 运行时。它通过将 uprobes 等操作移至用户空间，避免了频繁的内核上下文切换，提供了高达 10 倍的性能提升，并支持非特权级运行。
    *   **PTX JIT**：介绍了 PTX（Parallel Thread Execution）作为 NVIDIA GPU 的中间表示。PTX Just-In-Time (JIT) 编译允许开发者实时将 PTX 代码转换为机器指令，实现动态插桩和优化。

*   **3. 相关工作 (Related Work)**
    *   讨论了 Meta 的 AI 可观测性基础设施（如 Dynolog, Kineto, Strobelight/BPF）以及现有的 GPU 分析工具。指出了现有工具在应对设备特定复杂性和大规模遥测时的局限性。

*   **4. eGPU 设计 (eGPU Design)**
    *   展示了 eGPU 的系统设计图。
    *   **左侧**：现有的 eBPF 工具链（clang, bpftool 等）生成字节码。
    *   **中间**：eBPF 用户态应用层，包含 `bpftime-syscall.so`（提供用户态验证器和 JIT 编译器）和共享内存区域（用于存放 eBPF map，供 CPU 和 GPU 共同访问）。
    *   **右侧**：CUDA 环境。eBPF 字节码被编译为 PTX 或其他 GPU 兼容格式，并通过 PTX 注入技术插入到 CUDA 代码中的特定挂钩点（inlineHook）。

*   **5. eGPU 实现 (eGPU Implementation)**
    *   **自修改代码 (Self Modifying Code)**：借鉴 ParallelGPU OS (POS) 的思想，利用 PTX 注入技术在不停止内核的情况下拦截或插入指令。
    *   **共享内存 (Shared Memory)**：使用 `boost::managed_shared_memory` 在 CPU 和 GPU 之间建立透明的数据交换通道，减少数据编组和拷贝开销。
    *   **同步 (Synchronization)**：利用原子操作和内存栅栏（memory fences）确保 CPU 线程和 GPU 内核在访问共享数据结构时的数据一致性。
    *   **PTX 生成与注入 (PTX Generation & Injection)**：将 eBPF 逻辑转换为专用的 PTX 片段，通过设备驻留钩子（device-resident hooks）动态注入到运行中的内核。

*   **6. 用例 (Use Case)**
*   
    *   **6.1 GPU 内存观察器与 CXL.mem 模拟器**：通过插桩 GPU 的加载（LD）和存储（ST）指令，实现对内存带宽利用率、访问模式和争用的细粒度观察。并模拟 CXL.mem 的行为（动态延迟插入）。
    *   **6.2 LLM CPU-GPU 协同缓存**：针对大语言模型推理中的显存限制，利用 eGPU 提供的访问模式洞察（如哪些权重被频繁访问），优化 CPU-GPU 之间的缓存驱逐和预取策略。

*   **7. 评估 (Evaluation)**
    *   **微基准性能**：对比了 eGPU 和基于 NVBit 的工具（gpumemtrace）。结果显示 eGPU 的插桩开销显著低于 NVBit，且在不同内存访问大小下保持稳定的低延迟。

*   **8. 结论 (Conclusion)**
    *   总结了 eGPU 的优势：避免频繁内核启动、支持通过共享内存的快速数据交换、支持不中断活跃内核的动态代码注入。强调了其在流式处理和不规则任务中的潜力。

### 4. 中文结论
我们的早期测试表明，eGPU 通过避免频繁的内核启动并支持通过共享内存进行更快的数据交换，有助于减少开销。它还可以处理动态代码注入，而不会中断活跃的内核。通过让 GPU 保持占用状态来处理长时间运行的内核，我们简化了计算流水线并更有效地平衡了工作负载，这对于流式处理或不规则任务特别有用。eGPU 证明了收紧同步和添加更灵活的基于 PTX 的插桩可以为构建响应迅速、专为 GPU 驱动的工作负载量身定制的高吞吐量框架奠定基础。

---

## 第二部分：指定章节详解

### 1. eGPU 核心设计思想与架构 (Design & Architecture)

eGPU 的核心目标是打破传统 eBPF 仅限于 CPU 内核态或用户态的限制，将其强大的可编程性和可观测性能力“注入”到 GPU 这一异构计算单元中。其设计体现了 **统一性** 和 **动态性**。

*   **统一的用户态 eBPF 环境**：
    eGPU 并没有重新发明一套全新的语言，而是复用了现有的 eBPF 工具链（如 clang, bpftool）。这意味着开发者可以使用熟悉的 C 语言编写 eBPF 程序，原本用于追踪 CPU 系统调用的逻辑，现在可以被扩展用于追踪 GPU 内部事件。
    *   **bpftime 的角色**：eGPU 构建在 `bpftime` 之上。`bpftime` 是一个用户态 eBPF 运行时，它允许 eBPF 程序在非特权级用户空间运行，通过二进制重写（binary rewriting）挂钩函数。eGPU 利用这一特性，将“用户空间”的概念延伸到了 GPU 设备端。

*   **跨架构的共享内存通信**：
    为了解决 CPU 和 GPU 之间通信高延迟的问题，eGPU 设计了一个基于 **Shared Memory Maps** 的通信机制。
    *   **eBPF Map 的扩展**：在传统 eBPF 中，Maps 用于内核与用户态交换数据。在 eGPU 中，这个 Map 被放置在 CPU 和 GPU 都能访问的统一内存（Unified Memory）或固定内存（Pinned Memory）区域。
    *   **零拷贝遥测**：GPU 内核中注入的 eBPF 代码收集到的数据（如内存访问计数、延迟指标）直接写入这个共享 Map，CPU 端的用户态应用可以实时读取，无需昂贵的 PCIe 数据传输或显式的 CUDA Memcpy 操作。

### 2. 关键技术实现细节 (Implementation Details)

本章节详细阐述 eGPU 如何在底层实现“在飞（On-the-fly）”的 GPU 代码修改和执行。

#### 2.1 动态 PTX 注入 (Dynamic PTX Injection)
这是 eGPU 最具创新性的部分。传统的 GPU 性能分析工具（如 NVBit）通常需要在内核启动前对二进制进行重写，这会导致巨大的启动开销和运行中断。
*   **运行时编译**：eGPU 包含一个 JIT 编译器，能够将 eBPF 字节码翻译成 NVIDIA GPU 的中间指令集 PTX (Parallel Thread Execution)。
*   **注入机制**：
    *   eGPU 利用了类似于“自修改代码”（Self Modifying Code）的技术。它不依赖于停止内核、重编译整个代码库再重启的传统流程。
    *   它通过 **设备驻留钩子（Device-resident hooks）** 将生成的 PTX 片段动态插入到正在运行的内核中。这被称为“PTX 注入”。
    *   这种方法允许在不拆除现有内核上下文的情况下拦截或插入指令，极大地减少了开销。

#### 2.2 异构同步机制 (Heterogeneous Synchronization)
当 CPU 线程和 GPU 线程同时操作共享内存中的 eBPF Map 时，数据竞争是一个主要挑战。
*   **CPU 端**：使用标准的无锁数据结构（如 `std::atomic`）或 Boost 进程间原语。
*   **GPU 端**：使用 CUDA 的原子操作（Atomic Operations）和设备级内存栅栏（Device-wide Memory Fences）来序列化对全局或共享内存的更新。
*   **最小化冲突**：为了减少 GPU 内部 Warp 级别的冲突，eGPU 采用了优化策略，例如只让 Warp 中的一小部分线程执行原子操作，而其他线程继续处理数据，从而降低对并行度的影响。

### 3. 典型应用场景解析 (Use Cases)

论文详细描述了两个场景，展示了 eGPU 超越简单“监控”的能力，进入了“模拟”和“优化”的领域。

#### 3.1 GPU 内存微观观测与 CXL 模拟
*   **细粒度监控**：eGPU 通过 hook GPU 的 `LD`（加载）和 `ST`（存储）指令，能够捕获每一次内存访问的元数据（地址、大小、时间戳）。公式 $(1)$ 定义了实时带宽的计算方法：
    $$ BW = \frac{\sum_{i=1}^{N} \text{Bytes}_i}{T_{\text{window}}} $$
    这让开发者能看到内存访问的“纹理”——是顺序的还是随机的，是跨 Warp 合并的还是发散的。
*   **CXL.mem 模拟**：利用注入能力，eGPU 不仅能“看”，还能“改”。它通过在内存操作中动态插入延迟，来模拟 CXL（Compute Express Link）连接的远端内存行为。这对于评估未来硬件架构（如 CXL 内存池）对现有软件的影响极具价值。

#### 3.2 大语言模型（LLM）的协同缓存
LLM 推理极其消耗显存。当显存不足时，需要将部分数据（如 KV Cache 或模型权重）卸载到 CPU 内存。
*   **问题**：传统的驱逐策略（如 LRU）可能不适合 LLM 的特异性访问模式。
*   **eGPU 方案**：eGPU 可以实时追踪显存中哪些页被频繁访问（热数据），哪些处于空闲（冷数据）。
*   **优化**：基于 eGPU 提供的精确访问图谱，系统可以实施更智能的 **混合驱逐策略**。例如，结合 LFU（最近最少使用）和静态分析，确保“专家网络”（Expert Networks，在 MoE 模型中）或关键层权重常驻 GPU，而将非活跃专家的权重通过零拷贝缓冲区或流式预取技术高效地在 CPU 和 GPU 间调度。

### 4. 性能评估 (Evaluation)

*   **对比对象**：NVBit（NVIDIA 的二进制插桩工具）。
*   **测试方法**：通过微基准测试测量 `LD` 和 `ST` 操作的访问延迟。
*   **结果分析**：
    *   NVBit 由于其重型的插桩机制，随着内存访问量的增加，延迟呈现显著增长。
    *   eGPU 表现出 **极低且稳定的开销**。即使在插桩逻辑存在的情况下，其对原始内核执行速度的影响也远小于 NVBit。
    *   这验证了“动态 PTX 注入”+“共享内存遥测”的设计选择是实现生产级 GPU 实时监控的正确方向。
