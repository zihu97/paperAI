# HillInfer：基于 SmartSSD 层次化 KV 驱逐的高效边缘端长文本 LLM 推理

#### 第一部分：论文整体骨架

**论文文件名与英文标题**
- 文件名：`2602.18750v1.pdf`
- 英文标题：HillInfer: Efficient Long-Context LLM Inference on the Edge with Hierarchical KV Eviction using SmartSSD

**中文标题**
HillInfer：基于 SmartSSD 层次化 KV 驱逐的高效边缘端长文本 LLM 推理

**中文摘要**
在个人电脑（PC）等边缘设备上部署大语言模型（LLM）可以提供低延迟推理和强隐私保障，但长文本推理受限于内存和计算资源。除了模型参数外，键值（KV）缓存随上下文长度线性增长，成为主要瓶颈。虽然已有工作利用上下文稀疏性来驱逐不重要的 KV 数据，但这些方法主要针对内存丰富的平台，在应用到带有外部存储的资源受限边缘设备时，会产生过高的模型数据传输开销。本文提出 HillInfer，这是一个边缘端重要性感知长文本 LLM 推理框架，利用 SmartSSD 辅助的层次化 KV 缓存管理。HillInfer 共同管理 CPU 和 SmartSSD 之间的 KV 缓存池，并执行存储内重要性评估以减少不必要的数据移动。此外，我们设计了一个自适应的预取流水线，重叠 GPU、CPU 和 SmartSSD 之间的计算与 KV 数据传输。实验表明，HillInfer 在保持模型准确率的同时，相比基线实现了高达 8.56 倍的加速。

**各章节主旨内容**
1. **引言 (Introduction)**：阐述边缘端 LLM 推理的意义及长文本带来的 KV 缓存瓶颈，指出现有 KV 驱逐和 SSD 卸载技术的局限性。
2. **背景与动机 (Background and Motivations)**：介绍 LLM 推理过程、KV 缓存机制以及可计算存储（CSD/SmartSSD）的技术原理。
3. **HillInfer 系统设计 (System Design of HillInfer)**：详细描述系统架构、层次化 KV 缓存管理器（HIE 和 BKP）以及自适应预取流水线（APP）。
4. **实现与实验 (Implementation and Experiments)**：介绍基于 Flex 框架的实现细节、硬件配置以及在多个模型和数据集上的评估结果。
5. **相关工作 (Related Work)**：回顾长文本推理框架、KV 驱逐策略以及利用 CSD 优化 LLM 的研究现状。
6. **结论 (Conclusion)**：总结 HillInfer 的贡献和性能提升。

**中文结论**
本文提出了 HillInfer，这是一个面向边缘设备的重要性感知长文本 LLM 推理框架，利用 SmartSSD 辅助的层次化 KV 缓存管理。通过结合存储内重要性评估和自适应预取流水线，HillInfer 有效减少了内存受限下的数据移动和推理延迟。在配备消费级 GPU 的 PC 上的实验证明，HillInfer 在保持模型准确性的同时，实现了比最先进基线高出 8.56 倍的加速。

---

#### 第二部分：章节详解

##### 1. 引言与动机：边缘端长文本推理的挑战

**设计思想**：
作者观察到，隐私需求推动了边缘端（如 PC）部署 LLM 的趋势，但长文本推理面临严重的内存瓶颈。例如，Qwen-7B 在 4K 上下文和 Batch Size 为 8 时需 30GB 显存，远超 RTX 4090 的 24GB。

**核心痛点分析**：
*   **KV 缓存爆炸**：KV 缓存随长度线性增长，成为比模型参数更大的负担。
*   **现有方案局限性**：像 H2O 这种驱逐策略是为数据中心 GPU 设计的，仅在 GPU 和 CPU 间同步；而现有的 SSD 卸载方案（如 FlexGen）由于 SSD 显存带宽低，在执行重要性评估时需要频繁搬运数据，导致 GPU 长时间闲置。

**创新切入点**：
利用 SmartSSD 的存储内计算（NDP）能力。如果能在 SSD 内部直接评估 KV 数据的重要性，就只需传输“被选中”的重要数据，从而大幅缓解 PCIe 带宽瓶颈。

##### 2. 技术背景：SmartSSD 与 KV 缓存

**关键技术拆解**：
*   **可计算存储 (CSD)**：以三星 SmartSSD 为代表，在传统的 NAND Flash 控制器之外集成了一个 FPGA 和本地 DRAM。
*   **Near-Data Processing (NDP)**：数据在存储设备内部被 FPGA 处理，无需经过 PCIe 总线到达 CPU 内存。这种架构解决了“数据搬运比计算更昂贵”的问题。
*   **KV 策略**：LLM 在解码时只需当前 Query 与历史 Key 匹配，通过计算注意力权重（Attention Score）来判断哪些历史 Token 更重要。HillInfer 正是利用这一点，将该计算下放到 SSD 的 FPGA 上。

##### 3. HillInfer 系统设计：层次化管理与流水线

**3.2 层次化 KV 缓存管理器 (Hierarchical KV Cache Manager)**
这是 HillInfer 的核心，包含两个子组件：
*   **层次化重要性评估 (HIE)**：
    *   **原理**：将 KV 缓存分布在 CPU 和 SmartSSD。SmartSSD 利用 FPGA 并行计算其本地 KV 块的重要性得分。
    *   **交互逻辑**：SmartSSD 将得分打包成“Score Blocks”传给 CPU。CPU 汇总自身评估结果和 SSD 返回的得分，选出全局 Top-α 的重要 KV 数据。
    *   **权衡**：通过这种方式，只有选中的重要数据才从 SSD 搬运到 GPU，而全量数据的评估在存储内完成，掩盖了传输延迟。
*   **双向 KV 缓存池 (BKP)**：
    *   **设计考量**：由于 Token 重要性是随查询动态变化的（Query-dependent），简单的驱逐会导致频繁的“乒乓效应”（数据在 SSD 和 CPU 间反复换入换出）。
    *   **策略**：HillInfer 同时考虑了时间局部性（保留最近生成的 Token）和缓存命中率（保留高频被选中的 Token），在 CPU 和 SmartSSD 间进行双向动态更新。

**3.3 自适应预取流水线 (Adaptive Prefetch-based Pipeline, APP)**
*   **设计思想**：为了彻底消除 GPU 等待数据的时间，需要让计算和传输“赛跑”。
*   **关键逻辑**：在 GPU 计算第 $i$ 层时，CPU 和 SmartSSD 并行评估第 $i+1$ 层的重要性并预取数据。
*   **数学模型**：HillInfer 定义了缓存容量比 $\beta \approx f_c / f_s$（其中 $f_c, f_s$ 为 CPU 和 SSD 的处理吞吐量）。通过自适应调整两者的分配比例，确保两端能同步完成评估，从而实现计算与预取的完美重叠。

##### 4. 实现细节与实验评估

**实现细节**：
*   **技术栈**：基于 Python 开发，扩展了 Flex 推理框架。FPGA 部分使用 HLS（高层次综合）编写 C++ 代码。
*   **配置**：Ubuntu 22.04，RTX 4090 (24GB)，64GB 内存，三星 SmartSSD (4GB FPGA DRAM)。
*   **交互**：使用 Xilinx Runtime (XRT) 实现主机与 SmartSSD 的通信。

**对比分析**：
*   **基线对比**：对比了 Full Cache、H2O-like、InfiniGen-style 预取以及 LeoAM-like 方案。
*   **性能增益**：在 LongBench 数据集上，HillInfer 减少了 76.25%~88.32% 的端到端延迟。相比于单纯的预取方案，其加速比在 4.21x 到 8.56x 之间。
*   **消融实验**：结果显示 HIE&BKP 和 APP 流水线均对延迟降低有显著贡献。敏感性分析证实了 $\beta$ 的理论最优值与实验结果一致。

##### 5. 局限性与未来方向

*   **平台限制**：目前仅支持三星 SmartSSD，未来需适配 ScaleFlux 等其他 CSD 厂商。
*   **硬件扩展**：目前的实验是在 X86 PC 上完成的，未来可以扩展到 NVIDIA Jetson 等 Arm 架构边缘平台。
*   **逻辑增强**：随着 LLM 推理逻辑（如推理链、复杂推理）的演进，可以进一步探索在 CSD 内部加速更复杂的计算任务。
