# ProServe：面向LLM服务的统一多优先级请求调度.md

---

## 第一部分：论文整体骨架

**论文文件名**: 2512.12928v1.pdf
**英文标题**: ProServe: Unified Multi-Priority Request Scheduling for LLM Serving
**中文标题**: ProServe：面向LLM服务的统一多优先级请求调度

### 中文摘要
随着大型语言模型（LLMs）在交互式应用中的广泛部署，系统需要处理成千上万个具有不同服务水平目标（SLO）要求的并发请求。在此背景下，一个关键但常被忽视的维度是**客户端之间固有的优先级差异**；例如，业务关键型功能需要更高的性能保证，因为满足此类请求能产生显著更高的业务价值。然而，现有的LLM服务调度器未能联合优化SLO达成率和客户端级优先级。
为了弥补这一差距，我们首先将**多优先级请求调度**形式化为一个**服务增益最大化（Service Gain Maximization）**问题，即满足不同优先级请求的延迟要求会贡献不同程度的增益。随后，我们提出了 **ProServe**，这是一个统一的双层调度框架，旨在最大化整体服务增益。
在引擎层（Engine Level），**SlideBatching** 根据负载条件动态调整批处理形成和请求排序，采用滑动边界机制来平衡“密度优先”（Density-First）和“截止时间优先”（Deadline-First）策略。
在服务层（Service Level），**GoRouting** 在分布式实例之间执行面向增益（Gain-Oriented）和感知能力（Capability-Aware）的分发，主动为未来的高优先级或长请求预留容量。
在四个开源数据集和一个真实工业轨迹上的广泛评估表明，ProServe 始终优于最先进的基线方法，将系统增益提高了 35%，并将 SLO 达成率提升了 52%。

### 各章节主旨内容

*   **1. 简介 (Introduction)**
    *   阐述了LLM服务中同时处理SLO约束和不同客户端优先级的挑战。
    *   指出现有调度器在区分优先级和优化整体系统增益方面的不足。
    *   概述了ProServe框架及其核心贡献（形式化增益问题、SlideBatching、GoRouting）。

*   **2. 请求优先级与服务目标特征化 (Characterizing Request Priority and Service Objectives)**
    *   将多优先级调度问题定义为最大化服务增益。
    *   分析了现有指标（如加权SLO达成率）的缺陷（如丢弃/推迟策略、对Token级延迟不敏感）。
    *   提出了新的增益函数 **TDG (Token-level Deadline-aware Gain)**，将每个Token的及时交付作为二元增益，更准确地反映用户体验。

*   **3. 动机 (Motivation)**
    *   **总体延迟与请求优先级的权衡**：分析了严格优先级策略（导致资源分配不均）与FCFS（无法区分优先级）的局限性。
    *   **动态负载下静态调度器的适应性缺陷**：展示了EDF、SJF和FCFS在不同负载和批处理容量下的性能差异，说明静态策略无法适应波动的工作负载。
    *   **现有全局调度器的局限性**：指出了“最少负载”（Least-Load）策略在多优先级场景下的“过度平衡”（Over-balancing）问题，即可能因盲目平衡负载而牺牲了为未来高优先级请求预留资源的能力。

*   **4. 设计 (Design)**
    *   详细介绍了ProServe的架构。
    *   **SlideBatching (本地调度器)**：一种基于动态负载诊断的批处理算法，将请求分为“紧急”（Urgent）和“正常”（Normal），分别采用密度优先和截止时间优先策略。
    *   **GoRouting (全局调度器)**：一种面向增益的请求路由器，利用本地调度器的状态信息，采用双阈值策略选择最佳实例，避免过度负载平衡。
    *   **批处理延迟估计器**：基于回归模型预测Prefill和Decode的执行时间。

*   **5. 评估 (Evaluation)**
    *   **实验设置**：使用ShareGPT, Azure, BurstGPT, QwenTrace数据集，在xLLM框架上对比了vLLM-FCFS, Weighted VTC, Sarathi等基线。
    *   **单节点性能**：ProServe在TDG和SLO达成率上均优于基线，尤其在高负载下表现稳健。
    *   **多节点性能**：验证了GoRouting在PD分离和PD共置部署模式下的有效性。
    *   **消融研究**：证明了自适应紧急度划分、延迟感知等模块的必要性。

*   **6. 相关工作 (Related Work)**
    *   回顾了LLM服务优化（内核、缓存管理）和调度策略（Chunked Prefill, SLO感知调度, 优先级调度）的相关文献。
    *   指出ProServe是首个显式处理在线客户端固有优先级差异并联合优化SLO的工作。

*   **7. 结论 (Conclusion)**
    *   总结了文章的核心思想：将多优先级调度视为服务增益最大化任务。
    *   重申了ProServe（SlideBatching + GoRouting）的有效性。

### 中文结论
本文首先将多优先级调度问题形式化为服务增益最大化任务，其中满足高优先级请求的延迟要求通常比满足低优先级请求产生更大的服务增益。为了解决这个问题，我们提出了 **ProServe**，这是一种新颖的调度框架，由两部分组成：**SlideBatching**（本地调度器），它根据负载和优先级自适应地重新排序请求；以及 **GoRouting**（全局调度器），它执行面向增益和感知能力的请求分发。在多个数据集上的广泛实验验证了我们方法的有效性。

---

## 第二部分：指定章节详解

### 第2章：请求优先级与服务目标特征化 (Characterizing Request Priority and Service Objectives)

这一章的核心任务是**重新定义LLM服务的优化目标**。传统的SLO达成率（SLO Attainment）无法区分不同客户的重要性，也无法细粒度地反映Token生成的流畅度。

#### 1. 问题形式化：服务增益最大化
作者提出，系统目标应该是最大化所有请求的**总增益** $ \sum_{r \in R} f(r) $。其中 $f(r)$ 是单个请求 $r$ 在满足其延迟目标时产生的增益。
关键在于如何定义 $f(r)$。

#### 2. 现有指标的缺陷
作者通过“稻草人提案”（Strawman Proposals）分析了现有方法的不足：
*   **提案1：加权SLO达成率 (Weighted SLO Attainment)**
    *   公式：$ f_w(r) = w_{p(r)} \cdot \mathbb{I}[TTFT \le SLO \& TPOT \le SLO] $
    *   **缺陷**：
        1.  **Discard-or-Postpone Trick**：一旦TTFT超时，系统就可以直接丢弃请求或无限推迟它，因为增益已经为0。这极大地伤害用户体验。
        2.  **对Token延迟不敏感**：TPOT是平均指标，可能掩盖了中间某些Token生成极慢的情况（高方差）。
*   **提案2：基于TBT的Token级增益 (Token-level Gain with TBT)**
    *   引入了 **TBT (Time Between Tokens)** 这一更细粒度的指标。
    *   **缺陷**：存在 **Postponed Decoding Trick**。如果系统检测到当前Token已经要超时了，它可以故意进一步推迟该Token的输出，以便让*下一个*Token的TBT更容易满足（因为TBT是基于上一个Token的输出时间计算的）。这导致了“负单调性”：更早完成一个Token反而可能降低后续Token的达标率。

#### 3. 最终提案：Token级截止时间感知增益 (TDG)
为了解决上述所有问题，作者提出了 **TDG**。
*   **核心思想**：为每一个Token设定一个**绝对截止时间 (Absolute Deadline)**。
    *   $ deadline_{r, i} = TTFT_{SLO}^r + (i-1) \cdot TPOT_{SLO}^r $
*   **增益计算**：
    *   $ f_{TDG}(r) = \sum_i w_r(i) \cdot \mathbb{I}[t_{r, i} < deadline_{r, i}] $
    *   即：只要Token $i$ 在其绝对截止时间前送达，就获得增益 $w_r(i)$；否则为0。
*   **优势**：
    1.  **正单调性 (Positive Impact of Early Completion)**：提前完成一个Token虽然不直接增加该Token的增益（是二值的），但为后续Token留出了更多的时间缓冲（Slack），降低了后续违约风险。
    2.  **消除投机取巧**：由于截止时间是固定的，推迟当前Token只会压缩后续Token的可用时间，系统无法通过推迟来获益。

### 第3章：动机 (Motivation)

本章深入探讨了为什么简单的调度策略（如FCFS, EDF, SJF）和现有的负载均衡策略不足以解决多优先级问题。

#### 1. 静态调度器的“自适应缺陷”
作者通过实验发现，不同的静态策略适用于不同的负载阶段，没有一种策略能通吃：
*   **低负载 (Low Load)**：**EDF (最早截止时间优先)** 表现最好。因为资源充足，优先处理快过期的请求可以最大化SLO达成率。
*   **高负载 (High Load)**：EDF 表现崩溃。由于资源不足，EDF会试图拯救所有即将超时的请求，结果导致“多米诺骨牌效应”，所有请求都超时，均无增益。此时，**SJF (最短作业优先)** 或密度优先策略更好，因为它优先处理短请求，能快速释放资源，保住一部分请求的SLO，避免“全军覆没”。
*   **结论**：调度器必须根据实时负载**动态切换**策略。

#### 2. 全局调度中的“过度平衡” (The Over-Balancing Issue)
在多节点集群中，常见的 **Least-Load (最少负载)** 路由策略存在问题。
*   **场景**：实例A中等负载，实例B空闲。新来了一个长请求R2。
*   **Least-Load行为**：将R2发给B。
*   **问题**：如果稍后又来了一个**高优先级**的长请求R3，此时B已经被R2占据，A的剩余资源也不够R3，导致R3违约。
*   **更优解**：将R2发给A（只要A还能在SLO内处理R2）。这样保留了B的空闲状态，作为“储备”以应对未来可能突发的高优先级大请求。
*   **结论**：全局调度需要是 **Capability-Aware (能力感知)** 的，而不仅仅是负载均衡。

### 第4章：设计 (Design)

ProServe 采用双层架构：引擎层的 **SlideBatching** 和服务层的 **GoRouting**。

#### 4.1 批处理延迟估计器 (Batch Latency Estimator)
为了做出调度决策，必须能准确预测一个Batch的执行时间。
*   **模型**：分别对 Prefill 和 Decode 阶段建立线性回归模型。
    *   Prefill：$ \tilde{T}_p(r) = a_p \cdot l_q(r)^2 + b_p \cdot l_q(r) \cdot l_{kv}(r) + c_p \cdot l_q(r) $ （考虑注意力机制的二次复杂度和线性部分）
    *   Decode：$ \tilde{T}_d(r) = a_d \cdot l_{kv}(r) + b_d $ （主要受限于内存带宽和KV Cache长度）
*   **在线更新**：利用动量更新（Momentum Update）机制，根据实际执行时间不断修正预测系数，以适应运行时干扰。

#### 4.2 本地调度器：SlideBatching
这是ProServe的核心创新之一，旨在解决“静态策略无法适应动态负载”的问题。

**算法逻辑**：
1.  **预算设定**：根据延迟预测模型，计算当前Batch能容纳的最大时间预算 $t_{budget}$。
2.  **紧急度划分 (Adaptive Urgency Partition)**：
    *   计算每个请求的剩余松弛时间 $r.remain$。
    *   定义一个负载判断函数 $\phi(r, Q)$，预估请求在队列中等待的时间。
    *   判断条件：如果 $r.remain < \gamma \cdot \phi(r, Q)$，则标记为 **URGENT (紧急)**，否则为 **NORMAL (正常)**。
    *   $\gamma$ 是一个激进系数。
3.  **混合排序策略**：
    *   **URGENT 请求**：按 **密度 (Density)** 降序排列。
        *   $Density = Gain / ExecutionTime$。在资源极度紧缺（大家都快超时）时，优先做单位时间内增益最高的请求（通常是短请求或高权重请求），放弃那些“救不活”的长请求。这类似于背包问题的贪心解法。
    *   **NORMAL 请求**：按 **剩余时间 (Remaining Time)** 升序排列。
        *   即EDF策略。在资源相对充足时，优先处理截止时间近的，防止它们变成紧急状态。
4.  **滑动边界 (Slide Mechanism)**：
    *   随着负载增加，$\\phi(r, Q)$ 变大，更多请求被划分为 URGENT，调度器行为自然从 EDF 向 Density-First 平滑过渡。
    *   这种机制自动解决了第3章提到的动静态权衡问题。

#### 4.3 全局调度器：GoRouting
GoRouting 负责将请求分发到最合适的实例。

**关键机制**：
1.  **增益预估 (EstimateGain)**：
    *   对于每个候选实例，GoRouting 模拟如果将请求发过去，该实例能产生的**增量增益 (Incremental Gain)** $\Delta_p$。
    *   这不仅考虑当前请求能否满足SLO，还考虑它是否会挤占队列中其他请求的资源导致它们违约（负增益）。
2.  **双阈值能力感知分发**：
    *   计算所有实例的最大可能增益 $\Delta_{max}$。
    *   筛选出“好实例”集合 $C = \{ p \in P | \Delta_p \ge \alpha \cdot \Delta_{max} \}$。
    *   在集合 $C$ 中，根据负载水平进一步划分为 $L$ (低负载) 和 $H$ (高负载)。
    *   **策略**：
        *   优先选择 $L$ 集合中**最忙**的那个（Most Loaded in Low-Load set）。为什么？为了**Packing (堆叠)**。只要能满足SLO，尽量把请求塞给已由负载的实例，从而保留那些完全空闲的实例给未来可能的高难请求。这直接解决了“Over-balancing”问题。
        *   如果 $L$ 为空，才退化为在 $C$ 中选负载最小的。

### 第5章：评估 (Evaluation)

#### 5.1 实验亮点
*   **TDG 提升**：在各种数据集（ShareGPT, Azure等）和模型（Qwen2-7B, Qwen3-32B）上，ProServe 相比 vLLM-FCFS 和 Sarathi-Priority 等基线，TDG 提升明显。
*   **SLO 达成率**：特别是在高负载下，ProServe 能够维持较高的 SLO 达成率，而基线方法通常会急剧下降。
*   **鲁棒性**：在 BurstGPT 这种突发性极强的数据集上，ProServe 展现了极强的适应能力。

#### 5.2 优先级隔离效果
*   实验显示，ProServe 能够有效区分优先级。随着高优先级权重的增加（Priority Weight Scaling），高优先级请求的 SLO 达成率显著上升，而整体系统的 SLO 保持稳定。这证明了它不是简单地牺牲低优先级，而是通过更聪明的调度挤出了系统潜能。

#### 5.3 调度时间线分析 (Timeline Analysis)
*   图表展示了 SlideBatching 如何随时间动态调整 URGENT 和 NORMAL 请求的比例。在负载波峰到来时，URGENT 请求比例激增，系统自动切换到“保大局、弃长尾”的模式；波峰过后，迅速切回 EDF 模式清理积压，证明了设计的有效性。
