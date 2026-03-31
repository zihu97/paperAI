# ROLLMUX：面向解耦强化学习后训练的阶段级复用技术

**论文文件名**: 2512.11306v2.pdf
**英文标题**: ROLLMUX: Phase-Level Multiplexing for Disaggregated RL Post-Training
**中文标题**: ROLLMUX：面向解耦强化学习后训练的阶段级复用技术

## 中文摘要
强化学习（RL）后训练（Post-Training）中的解耦架构正成为一种标准范式，该架构将内存受限的Rollout阶段和计算受限的Training阶段物理分离到专用的集群中，以最大化硬件效率。然而，在线策略算法（On-Policy Algorithms）所需的严格同步引入了严重的“依赖气泡（Dependency Bubbles）”，迫使一个集群在另一个集群运行依赖阶段时处于空闲状态。
本文提出了 **ROLLMUX**，这是一种针对解耦RL后训练的跨集群调度框架，旨在通过填补这些气泡来回收资源。ROLLMUX 基于这样一个洞察：一个作业的结构性空闲可以被另一个作业的活跃阶段有效利用。为了实现这一点，本文引入了 **共执行组（Co-execution Group）** 抽象，将集群划分为隔离的局部性域（Locality Domains）。这种抽象支持 **双层调度架构**：一个 **组间调度器（Inter-group Scheduler）**，使用保守的随机规划优化作业放置；以及一个 **组内调度器（Intra-group Scheduler）**，编排可证明最优的轮询（Round-Robin）调度。该组抽象还施加了 **常驻约束（Residency Constraint）**，确保大规模模型状态缓存在主机内存中，以实现“热启动”上下文切换。
我们在一个包含328个H20 GPU和328个H800 GPU的生产级测试床上评估了ROLLMUX。与标准解耦相比，ROLLMUX将成本效率提高了1.84倍；与最先进的共置（Co-located）基线相比，提高了1.38倍，且所有作业均实现了100%的服务水平目标（SLO）达成率。

## 各章节主旨内容

### 1. 引言 (Introduction)
介绍了大语言模型（LLM）开发重心向强化学习后训练（如推理、代码生成）的转移。指出了RL后训练包含Rollout（生成）、Training（训练）和Synchronization（同步）三个阶段。虽然解耦架构能解决资源不匹配问题，但引入了依赖气泡。现有异步方案虽能提升利用率但牺牲了模型质量。ROLLMUX作为一种跨集群调度框架被提出，通过共调度（Co-scheduling）来隐藏气泡，同时面临工作负载异构性、运行时随机性和上下文切换开销三大挑战。

### 2. 背景与动机 (Background and Motivation)
详细描述了RL后训练的工作负载特征：Rollout阶段内存带宽受限，Training阶段计算密集，Synchronization阶段网络受限。分析了解耦RL架构的优势（TCO降低）及其核心缺陷——依赖气泡。指出现有异步算法会导致样本陈旧（Sample Staleness），影响On-Policy任务的收敛性和准确性，因此同步执行仍是生产环境的首选，但这导致了严重的资源浪费。

### 3. 调度机遇与挑战 (Scheduling Opportunities and Challenges)
*   **机遇**：利用多租户集群中不同作业的依赖气泡，交错执行一个作业的计算阶段和另一个作业的内存阶段（共执行）。
*   **挑战 C1（异构性）**：RL作业在模型大小、响应长度和交互模式上差异巨大，简单的分时复用会导致严重干扰，这是一个NP-hard的作业车间调度（Job Shop Scheduling）问题。
*   **挑战 C2（随机性）**：RL Rollout时长服从长尾分布，少数“掉队者（Stragglers）”会导致偏斜气泡（Skewness Bubbles），使静态规划失效。
*   **挑战 C3（上下文切换）**：RL是有状态的，频繁加载巨大的模型权重和优化器状态会导致高昂的冷启动开销。

### 4. ROLLMUX 调度设计 (The ROLLMUX Scheduling Design)
提出了核心抽象 **共执行组（Co-execution Group）**，将全局调度问题分解为独立的子问题。设计了双层调度器：
*   **组间调度器**：处理新作业的放置，目标是最小化边际配置成本。采用保守的准入控制应对随机性，并通过修剪饱和组来加速搜索。
*   **组内调度器**：在组内执行循环轮询调度（Cyclic Round-Robin），并引入 **长尾迁移（Long-Tail Migration）** 机制，动态整合Rollout阶段的尾部请求，释放大部分资源给下一个作业。

### 5. ROLLMUX 执行平面 (The ROLLMUX Execution Plane)
描述了系统层面的实现机制：
*   **以阶段为中心的控制（Phase-Centric Control）**：将RL阶段视为一等调度实体，使用 `@rollmux.phase` 装饰器和运行时Hook，配合主机内存缓存实现毫秒级的热启动切换。
*   **拓扑感知模型同步（Topology-Aware Model Sync）**：针对跨集群带宽瓶颈，设计了分层传输策略（Inter-cluster Scatter + Intra-cluster Broadcast），利用本地高速网络（如NVLink/InfiniBand）加速广播。

### 6. 系统实现 (System Implementation)
ROLLMUX 基于 ROLL 框架实现，包含约5200行代码。详细介绍了其闭环工作流：Profiler -> Inter-Group Scheduler -> Group Placement -> Runtime Controller -> Phase Execution -> Done Signal。

### 7. 评估 (Evaluation)
在异构集群（H20用于Rollout，H800用于Training）上进行了大规模评估。
*   **微基准测试**：展示了ROLLMUX在时域复用、处理Rollout密集型作业和空域复用方面的优势。
*   **消融实验**：验证了长尾迁移和拓扑感知同步的有效性。
*   **规模化测试**：使用微软Philly生产痕迹回放，ROLLMUX相比Solo-D节省1.84倍成本，相比veRL节省1.38倍成本。
*   **调度器性能**：与离线最优解相比差距极小（成本仅高6%），且决策延迟极低，具备良好的可扩展性。

### 8. 相关工作 (Related Work)
对比了通用的深度学习调度器（通常关注公平性或作业级吞吐）、单作业RL优化框架（缺乏全局视角）以及其他解耦系统（主要针对LLM推理而非训练）。ROLLMUX是首个针对随机性RL后训练的阶段级共调度系统。

### 9. 结论 (Conclusion)
ROLLMUX 通过双层调度机制和紧密编排的执行模式，有效消除了解耦RL后训练中的依赖气泡，显著降低了集群配置成本，同时保证了严格的SLO。

## 中文结论
本文提出了ROLLMUX，这是一种专为Rollout与Training解耦的RL后训练设计的多租户集群调度器。ROLLMUX引入了一种双层调度机制，能够近乎最优地将RL作业划分到共执行组中，并以紧密编排的模式执行它们。这种方法有效地减少了由解耦架构引起的依赖气泡，从而最小化了集群的配置成本。在包含多达328个GPU的解耦集群上使用真实生产痕迹进行的广泛评估表明，ROLLMUX在保持100%性能SLO的同时，实现了高达1.84倍的成本节约。

---

# 指定章节详解

## 第三章：调度机遇与挑战 (Scheduling Opportunities and Challenges)

本章深入剖析了在解耦架构下进行资源复用的理论可行性与实际困难。

### 3.1 共调度机遇 (The Co-Scheduling Opportunity)
*   **核心痛点**：在单作业视角下，由于On-Policy算法要求Rollout和Training严格同步，依赖气泡（即一个阶段等待另一个阶段完成时的闲置时间）是不可避免的。
*   **解决方案**：在多租户环境下，利用不同作业的“互补性”。通过 **共执行组（Co-execution Groups）**，调度器可以将作业A的计算密集型Training阶段与作业B的内存密集型Rollout阶段交错执行。
*   **预期效果**：这种“编织（Weaving）”式的执行不仅填补了气泡，还让系统能同时饱和低成本的Rollout集群（H20）和高性能的Training集群（H800），理论上能最大化TCO效益。

### 3.2 三大核心挑战
虽然共调度理念直观，但在生产环境中落地面临三个主要障碍：

1.  **C1：工作负载异构性与调度难解性 (Workload Heterogeneity)**
    *   **现象**：生产环境中的RL作业差异巨大。模型大小从3B到32B不等，响应长度从4k到32k tokens，交互模式包含单轮和多轮。这导致各阶段持续时间（50s到900s+）和资源需求极度多样。
    *   **问题**：简单的分时复用（如将两个Rollout密集的作业配对）会导致严重的资源争用和性能下降（如图3所示）。
    *   **本质**：寻找最优配对本质上是一个 **作业车间调度问题（Job Shop Scheduling Problem, JSSP）**，这是一个已知的NP-Hard问题。

2.  **C2：运行时随机性与偏斜气泡 (Stochastic Runtime)**
    *   **现象**：与预训练不同，RL Rollout的时长极不稳定，取决于生成的Token数量，且服从 **长尾分布**。
    *   **问题**：少数“掉队”请求（Stragglers）会拖慢整个Batch的完成时间，导致GPU在等待期间闲置，形成“偏斜气泡（Skewness Bubbles）”。
    *   **后果**：静态的调度计划会因运行时的波动而失效，导致严重的内部资源碎片化。

3.  **C3：上下文切换开销与内存常驻 (Context Switching & Residency)**
    *   **现象**：RL后训练是 **有状态（Stateful）** 的，涉及数百GB的模型权重、优化器状态和KV Cache。
    *   **问题**：如果在作业切换时从磁盘或通过慢速网络重新加载状态（冷启动），开销可能高达80秒，这将完全抵消共调度带来的时间收益。
    *   **约束**：必须采用 **热启动（Warm-start）**，即保持状态在主机内存（Host DRAM）中。但这受到主机内存容量的限制（通常1-2TB），限制了能共存的作业数量（Residency Constraint）。

## 第四章：ROLLMUX 调度设计 (The ROLLMUX Scheduling Design)

为了解决上述挑战，ROLLMUX 设计了一个双层调度系统。

### 4.1 共执行组抽象 (Co-Execution Group)
*   **定义**：一组通过分时复用共享特定Rollout和Training资源池的作业集合。
*   **作用**：
    1.  **降低复杂度**：将全局调度问题分解为多个独立的、小规模的组内调度问题，使其在生产规模下可解。
    2.  **局部性域**：通过将作业“钉”在特定的节点组上，强制执行 **内存常驻约束**，确保所有切换都能通过高速PCIe从Host内存加载，实现热启动。

### 4.2 组间调度器 (Inter-Group Scheduler)
负责处理新作业的 **放置（Placement）** 问题。
*   **目标**：最小化引入新作业带来的 **边际配置成本（Marginal Provisioning Cost, $\Delta$）**。
*   **策略**：
    1.  **保守规划（Conservative Planning）**：为了应对随机性（C2），调度器假设所有请求都会达到最大Token长度（最坏情况）来估算作业时长，确保在任何情况下都满足SLO。
    2.  **放置策略搜索**：
        *   **直接打包 (Direct Packing)**：塞入现有组的气泡中（$\Delta = 0$）。
        *   **Rollout 扩容 (Rollout Scaling)**：如果Training资源有空余但Rollout不足，仅增加廉价的Rollout节点。
        *   **隔离配置 (Isolated Provisioning)**：创建一个新组（保底策略）。
    3.  **修剪饱和组**：通过计算组的“瓶颈负载”与“自然周期时间”，快速过滤掉已经饱和的组，保证搜索效率。

### 4.3 组内调度器 (Intra-Group Scheduler)
负责 **运行时编排（Runtime Orchestration）**。
*   **轮询策略 (Round-Robin Policy)**：
    *   定义一个 **元迭代（Meta-iteration）**，在此期间每个作业执行一次Rollout和一次Training。
    *   **理论最优性**：论文证明了对于未饱和的组，这种简单的轮询调度能最大化资源利用率。
*   **长尾迁移 (Long-Tail Migration)**：
    *   针对长尾分布导致的碎片化，当一个Rollout阶段大部分请求完成（例如80%），系统会中断执行。
    *   将剩余的少数“掉队”请求 **迁移** 到一小部分Worker上继续运行。
    *   **效果**：释放出来的大部分Worker可以立即开始下一个作业的Rollout，从而将原本串行的尾部等待转化为并行的流水线执行。

## 第五章：ROLLMUX 执行平面 (The ROLLMUX Execution Plane)

本章介绍了支持上述调度策略的底层系统机制。

### 5.1 以阶段为中心的控制 (Phase-Centric Control)
*   **理念转变**：打破传统的以“作业”为单位的资源分配，转变为以 **阶段（Phase）** 为单位。
*   **声明式API**：用户使用 `@rollmux.phase` 装饰器标记代码中的Rollout、Train等函数。
*   **透明运行时 (Transparent Runtime Shim)**：
    *   **拦截与许可**：在阶段开始前向调度器申请资源。
    *   **热启动加载**：获批后，从Host内存快速加载状态到GPU。
    *   **轻量级挂起**：阶段结束后，将状态卸载回Host内存（保留控制平面上下文，如NCCL通信器），进程进入睡眠而非销毁。这避免了昂贵的控制平面重建开销。
*   **运行时钩子 (Runtime Hooks)**：暴露内部状态（如Token生成进度），允许调度器触发长尾迁移。

### 5.2 拓扑感知模型同步 (Topology-Aware Model Sync)
*   **背景**：解耦架构中，Rollout和Training集群通常跨机架甚至跨数据中心，通过较慢的以太网连接。
*   **传统问题**：简单的 `AllGather` 会导致N个Training GPU向M个Rollout GPU发送N*M份数据，拥塞慢速链路。
*   **ROLLMUX 方案**：**分层两阶段传输**。
    1.  **阶段一（跨集群散射, Inter-cluster Scatter）**：将模型切分为N个分片，每个Training GPU只通过慢速链路发送一个分片给对应的Rollout GPU（点对点传输，无冗余）。
    2.  **阶段二（集群内广播, Intra-cluster Broadcast）**：接收到分片的Rollout GPU利用集群内部极高带宽的 NVLink 或 InfiniBand 将分片广播给其他所有节点。
*   **效果**：确保慢速跨集群链路上 **只有一份完整的模型数据传输**，极大减少了同步时间（在测试中加速了约8倍）。
