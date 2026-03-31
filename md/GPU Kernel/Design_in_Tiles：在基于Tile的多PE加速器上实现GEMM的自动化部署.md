# Design in Tiles: Automating GEMM Deployment on Tile-Based Many-PE Accelerators

---

## 第一部分：论文整体骨架

### 1. 论文基本信息
*   **英文标题**：Design in Tiles: Automating GEMM Deployment on Tile-Based Many-PE Accelerators
*   **中文标题**：Design in Tiles：在基于Tile的多PE加速器上实现GEMM的自动化部署
*   **作者**：Aofeng Shen, Chi Zhang, Yakup Budanaz, Alexandru Calotoiu, Torsten Hoefler, Luca Benini
*   **机构**：ETH Zurich (苏黎世联邦理工学院)
*   **发布年份**：2025 (根据参考文献推断，文中标注2018似乎为模板残留，参考文献有2025年条目)

### 2. 中文摘要
基于Tile（瓦片式）的多处理单元（Many-PE）加速器在通用矩阵乘法（GEMM）上具有实现高性能的潜力，但由于其最优软件映射与硬件设计紧密耦合，导致手动开发极其困难，编程难度极大。
本文提出了 "Design in Tiles (DiT)"，这是一个将部署工具链与这些加速器的可配置执行模型连接起来的自动化框架。
在评估方面，我们将该框架应用于GEMM，针对相当于NVIDIA GH200的大规模加速配置（例如 32x32 tiles，FP8精度下1979 TFLOPS，4 TB/s带宽）进行了测试。
结果显示，我们的方案比使用专家级调优GEMM库的GH200实现了更高的PE（处理单元）利用率，在不同的矩阵形状上实现了1.2到2.0倍的加速。

### 3. 各章节主旨内容
*   **1. Introduction (引言)**：
    阐述了AI和HPC对算力需求的爆炸式增长。指出传统GPU架构在扩展时面临缓存抖动和利用率低的问题（以GH200 vs A100为例）。
    介绍了基于Tile的Many-PE加速器（如Dojo, Tenstorrent）作为解决方案，它们通过软件管理的暂存器（SPM）和片上网络（NoC）替代传统缓存层次结构。
    然而，这类架构编程极其复杂，缺乏自动化部署工具，本文旨在填补这一空白。

*   **2. Design Overview (设计概览)**：
    介绍了框架的两个核心部分：
    (1) **SoftHier**：一个基于GVSoC的建模和仿真框架，支持硬件级集合通信原语，可配置各种Tile架构参数。
    (2) **DaCe (Data-Centric Parallel Programming)**：作为部署和优化工具，使用SDFG（有状态数据流多图）作为中间表示（IR），负责代码生成和优化。

*   **3. Tile-based Deployment Schedule Abstraction (基于Tile的部署调度抽象)**：
    详细描述了DiT框架的核心抽象层，包括：
    (1) **Tiling and Mapping (分块与映射)**：定义计算分配，支持2D、3D分块及集群索引重映射。
    (2) **Data Layout (数据布局)**：显式管理分布式HBM通道的数据分布。
    (3) **Dataflow (数据流)**：定义数据移动模式（如SUMMA, Systolic, Hierarchical），利用NoC集合通信。

*   **4. Evaluation (评估)**：
    在模拟的GH200级别硬件上进行了详细的性能分解分析。
    提出了四个关键洞察（Insights），涉及数据布局优化、硬件多播的使用、3D分块对不规则形状的优势，以及集群重映射技术。
    验证了框架在不同硬件配置（类A100和类GH200）下的可移植性和高利用率。

*   **5. Conclusion (结论)**：
    总结了DiT框架通过提供高层调度抽象，消除了手动编写Kernel的需求，实现了跨多种硬件配置的高性能、便携式GEMM部署。
    证明了利用NoC上的硬件集合通信可以显著提升性能，超越了商业级GPU上的专家优化库。

### 4. 中文结论
本文介绍了DiT，这是一个专为基于Tile的多PE加速器设计的自动化GEMM部署框架。
通过提供可配置的、高层的部署调度抽象，该框架消除了手动开发内核的需求，并在广泛的Tile基加速器参数范围内实现了可移植的、高利用率的GEMM部署。
利用DiT，我们强调了最优GEMM配置如何随矩阵形状变化，并证明了NoC上的硬件支持集合通信可以进一步提升性能。
在与现有最先进解决方案（SoA）的对比中，我们在配置为匹配GH200峰值性能的SoftHier上评估了GEMM性能，实现了比在GH200上运行的专家调优GEMM内核高出1.2到2.0倍的硬件利用率。

---

## 第二部分：指定章节详解

### 1. Introduction (引言)：为何需要Tile-based架构与自动化？

*   **背景挑战**：
    随着大模型的发展，对算力的需求激增。虽然GPU不断堆砌算力（如GH200），但实验数据（图1）显示，在处理相同GEMM形状时，更大更强的GH200的平均利用率反而低于老款的A100。
    核心瓶颈在于GPU传统的**硬件管理缓存层次结构**。在大规模并行下，维持缓存局部性和避免缓存抖动（Thrashing）变得极其困难，导致性能不可预测。

*   **架构变革**：
    为了解决利用率问题，工业界（如Tesla Dojo, SambaNova, Tenstorrent）转向了**基于Tile的多PE架构**。
    *   **特征**：放弃统一的共享缓存，采用大量计算Tile。每个Tile拥有软件管理的本地L1暂存器（SPM）。
    *   **通信**：通过高带宽、可编程的片上网络（NoC）连接。
    *   **优势**：程序员可以直接控制HBM到L1的数据流，消除缓存冲突，将更多晶体管预算用于计算而非缓存控制逻辑。

*   **核心痛点**：
    这种架构虽然理论性能强，但**编程极其痛苦**。手动管理成千上万个核心的数据移动、同步和内存布局不仅繁琐，而且难以优化。
    一旦硬件配置改变（如Tile数量变化），手动编写的代码几乎需要重写。因此，急需一个自动化的部署框架。

### 2. Design Overview (设计概览)：DiT框架的双引擎

DiT框架由两部分组成：作为“靶场”的仿真器 **SoftHier** 和作为“指挥官”的编译器 **DaCe**。

#### 2.1 SoftHier：可配置的硬件仿真模型
*   **基础**：建立在GVSoC事件驱动模拟器之上，校准于开源RTL的周期精确模拟。
*   **架构模型**：
    *   **Tile**：包含PE（标量、向量、矩阵引擎）、DMA和L1 SPM。
    *   **NoC**：连接所有Tile，支持高效的数据传输。
    *   **HBM**：分布在网格边缘，通过内存控制器连接。
*   **杀手级特性**：**硬件支持的集合通信原语（Hardware-supported Collective Primitives）**。
    *   采用灵活的基于掩码（Mask-based）的寻址系统。
    *   允许一个Tile向自定义的Tile组（如一行、一列或矩形区域）广播数据，或进行规约（Reduce）。
    *   这种机制是实现高效GEMM数据流的关键。

#### 2.2 DaCe (Data-Centric Parallel Programming)：数据为中心的部署工具
*   **SDFG IR**：使用“有状态数据流多图”（Stateful Dataflow Multigraph）作为中间表示。它显式地捕获了所有数据移动，包括HBM到Tile的传输以及Tile之间的NoC通信。
*   **代码生成**：DaCe不仅支持CPU/GPU，本文将其后端扩展以支持SoftHier，能够将SDFG下译为C代码，最终编译为RISC-V二进制文件在SoftHier上执行。
*   **图3展示了SDFG原语**：
    *   (a) HBM到L1传输。
    *   (b) MMAD任务（矩阵乘加）。
    *   (c/d) PE间的Send/Recv操作。

### 3. Tile-based Deployment Schedule Abstraction (核心技术：部署抽象)

这是论文的核心贡献，定义了如何将高层数学运算映射到底层硬件。它包含三个维度的参数化描述：

#### 3.1 Tiling and Mapping (分块与映射)
*   **任务分解**：决定了每个Compute Tile负责计算输出矩阵的哪一部分。
*   **策略**：
    *   **2D GEMM**：一个Tile计算一个完整的输出块。
    *   **3D (Split-K) GEMM**：多个Tile协作计算一个输出块（在K维度切分），需要后续的规约操作。
*   **Cluster Index Remap (集群索引重映射)**：
    *   **问题**：物理网格是固定的（如4x4），但某些工作负载可能适合1x16或2x8的逻辑布局。
    *   **方案**：引入重映射机制，将物理网格虚拟化为逻辑网格。框架会自动处理底层的物理掩码生成，这对处理不规则形状矩阵至关重要。

#### 3.2 Data Layout (数据布局)
*   **显式HBM管理**：SoftHier的HBM是分布式的，每个通道有独立地址空间。
*   **Split Scheme**：将大矩阵逻辑切分为Block网格（如图5）。
*   **Placement Scheme**：决定这些Block如何具体存放在各个HBM通道中（通常采用轮询Round-Robin以最大化带宽）。
*   **意义**：正确的数据布局能防止内存通道拥塞（Contention），是跑满4TB/s带宽的前提。

#### 3.3 Dataflow (数据流设计)
定义了数据如何在HBM、NoC和L1之间流动。框架支持多种经典和创新的数据流模式（图6）：
*   **SUMMA**：经典的广播式算法。行广播A，列广播B。
*   **Systolic (脉动阵列)**：数据仅在邻居间流动，像波浪一样推进。减少了长距离通信压力。
*   **Hierarchical (分层调度)**：
    *   **Systolic over SUMMA**：宏观上Systolic，微观组内SUMMA。
    *   **SUMMA over Systolic**：宏观上SUMMA，微观组内Systolic。
    *   这种分层设计能更好地利用片上带宽和局部性。
*   **Split-K**：支持K维度的切分和后续的Partial Result规约。
*   **BSP抽象**：使用“整体同步并行”（Bulk Synchronous Parallel）模型来描述调度，每个Superstep包含计算、通信和栅栏同步，并显式编码双缓冲（Double Buffering）以实现计算与通信的重叠。

### 4. Evaluation (评估详解)

#### 4.1 实验设置
*   **目标硬件**：模拟NVIDIA GH200的规格。
    *   Tile阵列：32x32 (1024个Tile)。
    *   算力：1979 TFLOPS (FP8)。
    *   带宽：4096 GB/s (4 TB/s)。
    *   NoC：4096-bit 链路宽度。
*   **对比基准**：NVIDIA GH200实测数据，使用CUTLASS 3.9和DeepGEMM库。

#### 4.2 关键洞察 (Insights)与性能分析
通过对DeepSeek V3模型中常用GEMM形状的测试，论文得出了四个关键优化方向：

*   **Insight 1: 数据布局至关重要**
    *   对比Baseline（无优化布局）和优化布局，发现优化HBM数据分布可以显著提升带宽利用率，解决内存受限（Memory-bound）问题。

*   **Insight 2: 硬件多播与流水线深度**
    *   对于计算密集型（Compute-bound）任务，流水线（Pipelining）能掩盖等待时间。
    *   但对于存储密集型（Store-intensive）任务，过深的流水线会导致HBM写入拥塞。
    *   **结论**：尽可能使用硬件多播（Multicast），并根据任务类型调整流水线级数。

*   **Insight 3: 3D Tiling (Split-K) 应对不规则形状**
    *   在2D Tiling下，如果M或N维度很小（如N=2112，分给32个Tile，每个才66），会导致Tile过小，填充率低。
    *   **优化**：使用3D Tiling切分K维度，可以让每个Tile负责更大的N维度块（例如由8个Tile共享一个N），显著提升矩阵引擎的利用率。

*   **Insight 4: Cluster Remap 应对 "Flat GEMM"**
    *   LLM解码阶段常见的Flat GEMM（如 64 x 2112 x 7168），M非常小。
    *   物理上的32x32网格很难切分。
    *   **优化**：将逻辑网格重映射为 1x1024，配合3D Tiling，使得每个PE能计算更大、更规整的数据块，大幅提升性能。

#### 4.3 最终战果
*   **Compute-Bound GEMM**：DiT框架生成的调度比GH200上的CUTLASS/DeepGEMM快 **1.2 - 1.5倍**。
*   **Flat GEMM (Memory-Bound)**：通过优化带宽利用率，实现了 **1.2 - 2.0倍** 的加速。
*   **可移植性**：无论是模拟A100配置还是GH200配置，SoftHier上的利用率都保持在极高水平，而真实的GH200相比A100利用率有明显下降。这证明了软件管理内存架构在规模扩展时的优越性，以及DiT框架的有效性。
