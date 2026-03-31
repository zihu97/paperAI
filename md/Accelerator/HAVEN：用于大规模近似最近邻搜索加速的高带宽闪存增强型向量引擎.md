# 2603.01175v1.pdf - HAVEN: High-Bandwidth Flash Augmented Vector ENgine for Large-Scale Approximate Nearest-Neighbor Search Acceleration

## 中文标题：HAVEN：用于大规模近似最近邻搜索加速的高带宽闪存增强型向量引擎

### 中文摘要
检索增强生成（RAG）依赖于大规模近似最近邻搜索（ANNS）来为大语言模型检索语义相关的上下文。在 ANNS 方法中，
IVF-PQ 在内存效率和搜索准确性之间提供了良好的平衡。然而，为了实现高召回率，通常需要重排（reranking）阶段，
这涉及到获取全精度向量进行精确比对。对于十亿级规模的向量数据库，由于 GPU HBM 的容量限制，这些全精度向量
必须存储在 CPU DRAM 或 SSD 中，导致离片（off-GPU）数据移动成为严重的延迟和吞吐量瓶颈。
本文提出了 HAVEN，这是一种增强了高带宽闪存（HBF）的 GPU 架构。HBF 采用新兴的堆叠 3D NAND 技术，
提供 TB 级容量和数百 GB/s 的带宽。通过将 HBF 和近存储搜索单元作为片上封装（on-package）的 HBM 补充，
HAVEN 使得全精度向量数据库可以完全驻留在设备端，消除了重排过程中的 PCIe 和 DDR 瓶颈。
通过对重构的 3D NAND 子阵列、功耗受限的 HBF 带宽以及端到端 IVF-PQ 流水线的详细建模，实验表明 HAVEN 
在十亿级数据集上的重排吞吐量提升了高达 20 倍，延迟降低了高达 40 倍。

### 各章节主旨内容
- **Chapter 1: 引言 (Introduction)** - 阐述 RAG 对大规模 ANNS 的需求，指出当前 GPU 内存层次结构在处理海量
  全精度向量重排任务时存在的容量与带宽失配问题。
- **Chapter 2: 背景 (Background)** - 介绍 ANNS 和 IVF-PQ 算法流程（粗筛选、压缩扫描、重排），并量化分析
  了当前 GPU-DRAM/SSD 架构在内存占用和延迟方面的挑战。
- **Chapter 3: 提出的 HAVEN 加速器 (Proposed HAVEN Accelerator)** - 详细描述 HAVEN 的硬件架构，包括
  HBF 存储层、逻辑层中的近存储搜索单元（NSP），以及针对高带宽需求对 3D NAND 进行的分布式子阵列重构。
- **Chapter 4: 评估 (Evaluation)** - 展示 HAVEN 在 BIGANN-1B 等真实世界大数据集上的性能表现，并进行
  了详尽的设计空间探索和与其他专用加速器的对比。
- **Chapter 5: 结论 (Conclusion)** - 总结 HAVEN 如何通过将海量存储集成至 GPU 封装内，有效解决了 RAG 
  检索中的数据移动瓶颈，并展望了内存中心化计算的前景。

### 中文结论
HAVEN 提出了一种创新的 GPU 内存架构，通过集成高带宽闪存（HBF）克服了大规模 ANNS 任务中重排阶段的容量和
数据移动瓶颈。通过将 3D NAND 重构为分布式子阵列并集成近存储重排单元，HAVEN 实现了 TB 级的片上向量存储。
实验证明，HAVEN 相比传统的 GPU-DRAM 和 GPU-SSD 系统，在吞吐量和延迟上都有数量级的提升，为未来以 RAG 
为代表的检索密集型 AI 加速器设计指明了方向。

---

# 章节详细解读

## Chapter 1 & 2: 引言与背景 (Introduction & Background)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：大模型 RAG 需要处理十亿级向量（Billion-scale）。IVF-PQ 算法虽然高效，但为了高召回率必须进行
  重排（Reranking）。重排需要读取未压缩的原始向量，其体积通常达到数百 GB。当前 GPU（如 A100/H100）的 HBM 
  容量仅为 80-192GB，无法容纳整个数据库。因此，系统被迫将原始向量存放在主机 DRAM 或 NVMe SSD 中。
- **核心动机**：跨 PCIe 总线获取数据的延迟比片上访问高出两个数量级，且带宽受限。作者观察到，重排阶段的延迟
  占用了查询总时间的 80% 以上。如果能将“存储海量向量”的能力集成到 GPU 封装内，就能彻底消除这一瓶颈。
- **设计目标**：在不显著改变 GPU 封装尺寸的前提下，提供 TB 级的存储容量和足够的随机访问带宽，以支持实时
  和批处理的向量检索。

### 2. 关键技术 (Key Technologies)
- **IVF-PQ 三阶段分解**：
    1. **粗量化探测（Coarse Probing）**：在 GPU HBM 中查找最近的聚类中心。
    2. **PQ 扫描（PQ Scan）**：在选定的列表中扫描压缩后的代码，获取候选集。
    3. **重排（Reranking）**：最耗时的一步，从存储中读取全精度向量并计算精确距离。
- **性能挑战量化**：
    - **挑战 1：吞吐量与召回率的失衡**。开启重排虽能提高召回率，但吞吐量会骤降 1-2 个数量级。
    - **挑战 2：内存占用挑战**。海量原始向量是内存需求的“元凶”。
    - **挑战 3：长重排延迟**。PCIe 数据传输成为决定性开销。

### 3. 实现细节 (Implementation Details)
- **实验配置**：作者使用 NVIDIA A100 GPU、DDR4 内存和 NVMe SSD 作为对比基准。
- **分析工具**：利用 Faiss 框架进行性能分析，揭示了数据移动对端到端延迟的具体影响。

---

## Chapter 3: 提出的 HAVEN 加速器 (Proposed HAVEN Accelerator)

### 1. 设计思想 (Design Philosophy)
- **混合内存体系**：HAVEN 并不是要取代 HBM，而是用 HBF（高带宽闪存）替换部分 HBM 堆栈。HBM 用于存放延迟敏感
  的 PQ 索引，HBF 用于存放容量敏感的原始向量数据库。
- **近存储处理（NSP）**：由于重排操作（距离计算）的算术强度（Arithmetic Intensity）较低，如果将海量数据
  传输到 GPU 核心处理会造成严重的带宽浪费。作者提出在 HBF 逻辑层直接部署计算单元。

### 2. 关键技术 (Key Technologies)
- **HBF 逻辑层中的近存储搜索单元 (NSP)**：
    - **重排队列 (Rerank Queue)**：存储来自 GPU 的候选向量 ID。
    - **地址生成模块 (Address Generation)**：将逻辑 ID 转换为 HBF 的物理地址。
    - **距离计算模块 (Distance Computation)**：集成 32 个乘加器（MACs），直接对从闪存读取的数据进行欧氏
      距离等计算。
    - **Top-k 单元**：使用 Bitonic 排序器实时筛选最佳结果。
- **3D NAND 重构**：传统的 3D NAND 为了高密度设计成大平面（Monolithic Plane），访问粒度粗且带宽低。HAVEN 
  将其重构为**分布式小子阵列（Distributed Subarrays）**，通过缩短位线（Bitline）长度降低寄生电容，从而
  大幅提升随机读取速度和内部并行度。

### 3. 实现细节 (Implementation Details)
- **硬件参数**：NSP 运行在 1GHz 频率，总面积约 4.11 $mm^2$，功耗 620.3 mW。
- **NAND 层数**：基准设计采用 256 层（WL layers）3D NAND，8 层芯片（8-Hi）堆叠可实现 512GB 的单堆栈容量。
- **封装集成**：通过 2.5D 中介层（Interposer）将 GPU 核心、HBM 和 HBF 互联。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **功耗权衡**：HBF 的带宽受限于热包络（约 30W/stack）。作者通过优化页面大小（4KB）和子阵列块数（64 blocks），
  在容量、读取能耗和延迟之间找到了最优平衡点。
- **数据流对比**：传统方案（GPU-DRAM/SSD）数据需经过 PCIe -> Host CPU -> DRAM；HAVEN 方案数据仅在
  封装内的 HBF -> NSP 之间流动，极大地减少了数据搬运能耗。

---

## Chapter 4 & 5: 评估与结论 (Evaluation & Conclusion)

### 1. 关键技术 (Key Technologies)
- **对比实验设置**：测试了 BIGANN-1B、SPACEV-1B（十亿级）和 Wiki-88M（高维）三个数据集。
- **评估指标**：每秒查询数（QPS）、查询延迟（ms）、Recall@100。

### 2. 实现细节 (Implementation Details)
- **性能结果**：
    - **吞吐量**：在十亿级数据集上，HAVEN 相比 DRAM 基准提升 3-8 倍，相比 SSD 提升一个数量级以上。在
      Wiki-88M（768 维）上优势扩大到 20 倍，因为其重排流量更大。
    - **延迟**：HAVEN 保持约 10ms 的稳定延迟，而 DRAM/SSD 随着 Batch Size 增加延迟呈指数级上升。
- **设计空间探索**：作者研究了 NAND 层数（128L/192L/256L）对性能的影响，验证了 256L 是兼顾容量和速度的
  最佳选择。

### 3. 权衡与对比 (Trade-offs & Comparison)
- **与专用加速器对比**：HAVEN 优于 ANNA（IVF-PQ ASIC）和 SmartANNS（基于 SmartSSD 的图搜索方案）。
  相比 ANNA，HAVEN 吞吐量提升 1.9 倍；相比 SmartANNS，提升 9 倍。
- **优势来源**：
    1. HBF 提供的 TB 级片上存储消除了 IO 瓶颈。
    2. 分布式 NAND 架构提供了远超传统 SSD 的片上读取带宽。
    3. NSP 实现了高效的任务卸载。

---
**提示**：本报告基于论文所述的架构模拟结果。HAVEN 展示了将新型存储技术（如 HBF）引入 GPU 封装对于解决
大模型检索瓶颈的巨大潜力。
