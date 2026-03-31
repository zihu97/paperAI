# FLYING SERVING: On-the-Fly Parallelism Switching for Large Language Model Serving
2602.22593v1.pdf
## 中文标题：FLYING SERVING：大语言模型服务中的在线并行策略动态切换系统

### 中文摘要
生产级大语言模型（LLM）推理服务必须在非稳态流量和混合请求需求下，同时提供高吞吐量、低延迟和充足的上下文容量。
数据并行（DP）通过运行独立副本最大化吞吐量，而张量并行（TP）则降低单请求延迟并聚合内存以支持长上下文推理。
然而，现有的推理栈通常在部署时固定并行配置；为应对突发流量、优先级请求或长上下文而进行的调整往往是破坏性的且缓慢。
本文提出了 Flying Serving，这是一个基于 vLLM 的系统，能够在不重启推理引擎工作进程的情况下实现在线 DP-TP 切换。
Flying Serving 通过虚拟化那些强制数据移动的状态来实现轻量级重构：(i) 零拷贝模型权重管理器，按需暴露 TP 分片视图；
(ii) KV 缓存适配器，跨 DP/TP 布局保留请求的 KV 状态；(iii) 预先初始化的通信池，分摊集合通信设置开销；
(iv) 无死锁调度器，协调执行偏斜下的安全转换。在三种主流 LLM 和真实推理场景下，Flying Serving 在高负载下提升性能达 
4.79 倍，在低负载下提升 3.47 倍，同时支持延迟敏感和内存驱动的请求。

### 各章节主旨内容
- **Chapter 1: Introduction** - 探讨 LLM 推理在生产环境中的挑战，指出静态并行策略（DP/TP）在应对非稳态负载时的局限性。
- **Chapter 2: Background and Motivation** - 分析 Transformer 架构、DP 和 TP 的特点，以及动态切换并行策略的必要性。
- **Chapter 3: An Overview of Flying Serving** - 介绍 Flying Serving 架构，将其作为协调多个引擎工作进程的中间层。
- **Chapter 4: Runtime Substrate for Dynamic Parallelism** - 详解实现动态切换的三大核心组件：模型权重管理器、KV 缓存适配器和通信池。
- **Chapter 5: Dynamic Scheduler** - 介绍动态调度协议及三种自适应切换策略（默认顺序、软抢占、硬抢占）。
- **Chapter 6: Evaluation** - 在多种模型和工作负载下测试系统的性能、延迟、吞吐量和长上下文处理能力。
- **Chapter 7: Related Works** - 综述分布式 LLM 推理和动态并行变换相关的研究。
- **Chapter 8: Conclusion** - 总结系统贡献及在实际生产环境中的应用前景。

### 中文结论
Flying Serving 证明了静态并行配置不应成为 LLM 服务的部署约束。通过虚拟化模型权重、KV 状态和通信连接，系统能够实现
轻量级的在线 DP-TP 重构。在真实工作负载下，该系统显著提升了吞吐量和响应速度，并能灵活处理混合优先级和长上下文请求。

---

# 章节详细解读

## Chapter 4: Runtime Substrate for Dynamic Parallelism (运行时基座)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：动态切换并行模式通常涉及重新分配内存、重新加载大尺寸参数张量以及重新初始化通信组，这些操作产生的开销（
秒级甚至分钟级）远超单次推理步骤的时间预算（毫秒级），导致服务停顿。
- **核心动机**：作者认为“状态虚拟化”是解决之道。通过解耦逻辑表示与物理存储，使权重和 KV 缓存常驻内存，仅改变其访问视图
。
- **设计目标**：在 15ms 内完成 DP 与 TP 模式的切换，同时不损失任何已生成的 KV 缓存状态。

### 2. 关键技术 (Key Technologies)
- **模型权重管理器 (Model Weights Manager)**：
    - **逻辑重分片 (Logical Weight Resharding)**：采用 Megatron-LM 风格的并行策略。在 DP 模式下，每个引擎持有全
量权重；切换至 TP 时，通过逻辑视图（View）激活特定的分片。
    - **零拷贝视图实现**：利用 PyTorch 的视图操作，确保 TP 运行时的活动权重在虚拟内存中连续，但物理上映射到原有的 DP
 副本。公式表示为：$W_{active}^{(r)} = View(W_{full}, dim, r, m)$，其中 $r$ 为秩，$m$ 为 TP 程度。
- **KV 缓存适配器 (KV Cache Adaptor)**：
    - **挑战**：DP 模式下每 token 存储完整隐藏维度 $D$，而 TP 模式下每设备仅存储 $D/p$。
    - **自适应块大小 (Adaptive Block Sizing)**：通过反向缩放块内的 token 数量 $B$ 来保持物理块字节数常量：
$M_{block} = B \cdot D_{local} \cdot P_{size}$。其中 $B(p) = p \cdot B_{base}$。
- **通信池 (Communicator Pool)**：
    - **拓扑感知初始化**：在启动时预先创建所有合法的物理相邻设备组（如 [0,1], [2,3] 为 2-TP，[0,1,2,3] 为 4-TP），
避免运行时调用 `new_group` 的高昂开销。

### 3. 实现细节 (Implementation Details)
- **技术栈**：基于 vLLM v1，对 `linear.py` 进行补丁处理以支持秩感知视图。
- **执行流程**：切换时，调度器只需从通信池提取句柄，并更新 KV 适配器的逻辑映射表，无需物理数据移动。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **优势**：消除了重构瓶颈，比冷重启快约 10,000 倍。
- **权衡**：预初始化通信组会占用少量主机内存（每个进程组约 2MB），但相比释放出的 GPU 内存和性能收益，开销微乎其微。

---

## Chapter 5: Dynamic Scheduler (动态调度器)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：异构请求长度会导致不同 DP 引擎在不同时间到达调度边界（执行偏斜），简单的同步等待会造成严重的资源浪费
。
- **核心动机**：提供多种抢占和投机执行策略，以平衡系统的吞吐量与实时响应能力。

### 2. 关键技术 (Key Technologies)
- **调度协议 (Algorithm 1)**：
    - 包含输入处理、全局同步（确保所有参与 TP 的引擎观察到相同的请求序列）、资源调度、KV 参数化、模式信号广播和模型执行
。
- **三种切换策略**：
    - **默认顺序切换 (Default Sequential)**：等待所有 DP 引擎空闲后再切换，简单但会导致短任务被长任务阻塞。
    - **软抢占 (Soft Preempt)**：面向吞吐量。允许空闲引擎在等待长任务结束时，先行以 DP 模式投机执行 TP 请求的部分 
decode 步骤，减少切换后的计算量。
    - **硬抢占 (Hard Preempt)**：面向延迟敏感任务。收到高优先级请求时，立即中断所有活动 DP 任务，利用统一 KV 池保留
状态，任务完成后无缝恢复 DP 执行。

### 3. 实现细节 (Implementation Details)
- **控制平面**：使用 Gloo 进行跨引擎的轻量级同步和模式切换信号传递。
- **数据平面**：在切换点，通过分布式 RPC（RpcBroadcast）原子化地配置所有引擎的执行模式。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **软抢占权衡**：投机执行生成的 KV 布局与 TP 不兼容，切换时需重新计算，但由于 decode 是访存密集型而重计算是计算密
集型，在现代 GPU 上这一权衡通常是正向的。
- **硬抢占优势**：得益于 KV 缓存适配器，中断的任务状态无需导出至 CPU，留在原地即可，恢复速度极快。

---

## Chapter 6: Evaluation (实验评估)

### 1. 实验配置
- **硬件**：8x NVIDIA H200 GPU。
- **模型**：Llama-3-70B（密集型）、GPT-OSS-120B（MoE 稀疏型）、Nemotron-8B（长上下文型）。
- **基准方案**：Static DP、Static TP、Shift-Parallelism (SoTA)。

### 2. 关键发现
- **突发流量表现**：Flying Serving 能够在高负载时切换至 DP 以维持吞吐量，避免了 TP 模式下的队列积压。相对于 
Static TP，P90 TTFT 降低了 1.66x 至 4.79x。
- **混合优先级**：在处理高优先级请求时，其延迟接近 Static TP（仅 1.09x-1.17x 差距），同时保持了 96% 的 DP 基准
吞吐量。
- **长上下文能力**：通过动态合并内存，支持的上下文长度从 4DPx2TP 的 264K 扩展到 1.9M，提升了 7.2 倍，且切换延迟
仅为 15ms（相比冷重启的数百秒）。

### 3. 权衡与对比
- **VS Shift-Parallelism**：在低负载下性能相当，但在处理突发流量时，由于 Flying Serving 更好的调度策略，其 TTFT
 降低了 3.39x，队列时间缩短了 2.48x。
- **内存优化**：动态 TP 释放了几乎所有可用的剩余 GPU 内存用于 KV 缓存，消除了 DP 模式下的权重冗余。
