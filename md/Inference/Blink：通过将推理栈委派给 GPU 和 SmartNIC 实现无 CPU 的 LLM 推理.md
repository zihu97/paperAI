# 2604.07609v1.pdf - Blink: CPU-Free LLM Inference by Delegating the Serving Stack to GPU and SmartNIC

## 中文标题：Blink：通过将服务栈委派给 GPU 和 SmartNIC 实现无 CPU 的 LLM 推理

### 中文摘要
大语言模型（LLM）推理正迅速成为核心的数据中心服务，但目前的推理引擎（serving stacks）仍将主机 CPU 置于
编排和标记级控制（token-level control）的关键路径上。这使得 LLM 性能对 CPU 干扰非常敏感，削弱了应用混部
（colocation）的能力，迫使运营商保留大量 CPU 裕量，导致资源利用率低下。本文提出了 Blink，一种端到端的
推理架构，通过将职责重新分配给 SmartNIC 和 GPU，将主机 CPU 从稳态推理路径中完全移除。Blink 将请求处理
卸载到 SmartNIC，利用 RDMA 将输入直接送入 GPU 显存，并用常驻 GPU 内核（persistent GPU kernel）取代了
由主机驱动的调度，该内核在没有 CPU 参与的情况下执行批处理、调度和 KV 缓存管理。评估结果表明，即使在隔离
环境下，Blink 的性能也优于 TensorRT-LLM、vLLM 和 SGLang，预饱和 P99 首字延迟（TTFT）降低了高达 8.47 倍，
P99 逐字延迟（TPOT）降低了高达 3.40 倍。在 CPU 干扰下，Blink 仍能保持稳定的性能，而现有系统的性能则下降
多达两个数量级。

### 各章节主旨内容
- **Chapter 1: Introduction** - 阐述了当前 LLM 推理对 CPU 的高度依赖性，指出混部环境下的 CPU 干扰已成为
  性能瓶颈，并提出了 Blink 的核心理念：彻底移除关键路径上的主机 CPU。
- **Chapter 2: Background and Motivation** - 分析了主机 CPU 作为延迟瓶颈的原因，特别是在自回归解码
  （autoregressive decoding）这种长程、有状态的过程中，现有的 SmartNIC 卸载方案仅针对数据路径，未能解决
  复杂的控制路径问题。
- **Chapter 3: Limitations of CPU-Based Mitigations** - 通过实验证明，现有的操作系统级优化（如巨页、
  CPU 绑定、缓存分区等）无法根治 CPU 瓶颈，微架构层面的 LLC 竞争和 TLB 失效是性能波动的深层原因。
- **Chapter 4: Blink Design** - 详细介绍了 Blink 的双平面架构：DPU 前端负责请求管理和分词，GPU 后端
  利用常驻调度内核和 CUDA Graph 实现全自动推理。
- **Chapter 5: Implementation** - 描述了系统的实现细节，包括 16k 行 GPU 端 CUDA 代码和 17k 行 DPU 端
  C++ 代码，以及如何利用 DOCA SDK 进行 RDMA 编排。
- **Chapter 6: Evaluation** - 对比了 Blink 与主流推理引擎在隔离和干扰下的性能表现，验证了其在延迟、吞吐
  量和能源效率方面的显著优势。
- **Chapter 7: Discussion** - 讨论了 Blink 的可扩展性（多 GPU 扩展）、服务优化集成以及在模型权重卸载
  场景下的应用前景。
- **Chapter 8: Related Work** - 回顾了 SmartNIC 卸载和 LLM 服务优化的相关研究，强调了 Blink 在端到端
  重构推理栈方面的首创性。
- **Chapter 9: Conclusion** - 总结了 Blink 的贡献，指出通过软硬件协同设计实现 CPU-Free 推理是解决
  数据中心资源整合与隔离性能的关键。

### 中文结论
Blink 证明了通过将持久的加速器驻留控制逻辑（persistent accelerator-resident control）与 SmartNIC 驻留
的 I/O 相结合，可以完全取代主机介导的编排，用于处理对延迟敏感的工作负载。Blink 在四种主流模型上的表现
均优于现有最先进系统，不仅提升了推理效率，更重要的是实现了推理性能与主机 CPU 负载的彻底解耦。这种架构
革新为云服务商在保证服务质量协议（SLO）的前提下，实现高密度的服务器整合和降低运营成本提供了可能。

---

# 章节详细解读

## 第一章 & 第二章：背景与痛点分析

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：当前 LLM 推理系统（如 vLLM, SGLang）在每一轮迭代（iteration-level）中都需要将控制权交还
  给主机 CPU 进行批处理决策、KV 缓存管理和内核启动。在高负载或混部环境下，CPU 上的其他进程会竞争 LLC
  和 TLB 资源，导致调度抖动直接传导至推理延迟。
- **核心动机**：作者观察到，尽管计算主要在 GPU 上完成，但“每 Token 一次 CPU 交互”的模式让 CPU 成了
  性能的天花板。Blink 的直觉是：如果能让 GPU 像 SmartNIC 一样直接处理网络请求并自主调度计算，就能将
  受干扰的 CPU 彻底踢出局。

### 2. 关键技术 (Key Technologies)
- **控制路径瓶颈识别**：论文指出，自回归解码将推理转变为一个长程的有状态过程，每一轮的微调（fine-grained）
  决策必须在微秒级完成。
- **干扰放大效应**：由于解码是迭代的，主机端的微小抖动会在数百个输出 Token 中累积，产生显著的尾部延迟。

---

## 第三章：CPU 级缓解方案的局限性

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：为什么简单的 CPU 绑定（Pinning）或缓存分区（CAT）不管用？
- **核心动机**：通过硬件性能计数器（perf stat）深挖微架构层面的根因。

### 2. 关键技术 (Key Technologies)
- **地址翻译放大（Address Translation Amplification）**：Python 推理栈在碎片化的虚拟地址空间运行，
  干扰进程通过内存频繁操作（如 madvise）导致 TLB 频繁失效，强制 CPU 进行昂贵的 Page Walk，从而污染 LLC。
- **LLC 竞争**：即使分配了专用的缓存 Way 给推理任务，由于 Page Walk 本身也要经过 LLC，干扰依然会通过
  这一路径渗透。

### 3. 实现细节 (Implementation Details)
- **实验配置**：在 H100 GPU 和 Xeon Gold 服务器上运行 vLLM，使用 pbzip2 和 Ninja 编译任务作为干扰源。
- **数据发现**：在 24 倍干扰下，vLLM 的 P99 TTFT 膨胀了 139 倍。

---

## 第四章：Blink 架构设计

### 1. 设计思想 (Design Philosophy)
- **设计目标**：将主机 CPU 转变为“供应平面”（provisioning plane）而非“数据平面”。即主机只负责启动时
  加载模型和捕获 CUDA Graph，一旦运行起来，主机 CPU 就从路径上退出。

### 2. 关键技术 (Key Technologies)
- **双平面架构**：
    - **DPU 前端（Frontend）**：运行在 BlueField DPU 的 ARM 内核上。负责请求接收、分词、管理请求生命
      周期，并通过一侧 RDMA（one-sided RDMA）将输入直接写入 GPU 端的共享环形缓冲区（Ring Buffer）。
    - **GPU 后端（Backend）**：运行在 GPU 上的常驻调度内核（Persistent Scheduler Kernel）。它持续轮询
      环形缓冲区，自主决策批处理方案并启动 CUDA Graph。
- **常驻调度内核**：占用一个 GPU 线程块（256 线程），执行无限循环逻辑：轮询、原子操作认领任务、启动
  CUDA Graph。
- **窗口化尾部启动恢复（Window-based tail-launch recovery）**：针对 CUDA 运行时对单父节点发起的
  Fire-and-forget 启动限制在 120 次的问题，Blink 通过单次 Tail Launch 自动刷新执行实例，实现了无限
  Token 生成的能力。

### 3. 实现细节 (Implementation Details)
- **共享环形缓冲区**：位于 GPU 显存中，是 DPU 和 GPU 唯一的同步点。利用 atomic CAS 实现无锁化管理。
- **CUDA Graph 缓存**：预先捕获针对不同批大小和序列长度的计算图（约 650-1000 个），每个仅占用 2-3 MB，
  总开销在 2-4 GB 显存。
- **并行槽扫描（Parallel slot scanning）**：调度内核的 256 个线程并行扫描 4096 个槽位，全扫描仅需
  1-5 微秒。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **DPU-GPU 分工**：DPU 擅长网络传输和请求解析，GPU 擅长高频决策和大规模计算。这种分工避开了 DPU
  ARM 核在处理高频调度决策时的计算瓶颈。

---

## 第六章：实验评估

### 1. 设计思想 (Design Philosophy)
- **性能验证**：验证在隔离环境下是否依然有优势，以及在干扰下是否真的“免疫”。

### 2. 关键技术 (Key Technologies)
- **三项指标**：TTFT (首字延迟), TPOT (逐字延迟), Throughput (吞吐量)。
- **隔离测试**：Blink 在 Llama-3 8B 上比 TRT-LLM 提升了 1.35 倍的 TTFT，这主要归功于消除了主机侧的
  调度延迟。
- **混部测试**：当混部 CPU 密集型任务时，vLLM 等系统的吞吐量跌至孤立状态的 30% 以下，而 Blink 几乎
  保持 100% 的性能保留率（99%-100% retention）。

### 3. 实现细节 (Implementation Details)
- **能源效率**：由于 Blink 的吞吐量更高，其每 Token 能耗（mJ/tok）在干扰下比现有系统降低了高达 70.7%。

---

## 第七章：讨论与扩展

### 1. 设计思想 (Design Philosophy)
- **普适性证明**：探讨 Blink 是否能兼容现有的主流优化技术（如 Chunked Prefill, Paged KV Cache）。

### 2. 关键技术 (Key Technologies)
- **多 GPU 支持**：通过在每个 GPU 上实例化持久调度器，并使用 GPU 引发的 RDMA（IBGDA）进行点对点通信，
  可以实现无 CPU 参与的张量并行（Tensor Parallelism）。
- **模型卸载支持**：对于显存装不下的模型，未来的 DPU RDMA 引擎可以直接管理权重从主机 DRAM 到 GPU
  的迁移，从而保持 CPU-Free 路径。

### 3. 实现细节 (Implementation Details)
- **软件兼容性**：Blink 的 DPU 前端提供了兼容 OpenAI 的 HTTP API，使其能够作为现有服务的无缝替代品。
