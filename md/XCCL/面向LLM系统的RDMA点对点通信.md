# 面向LLM系统的RDMA点对点通信

## 文件名和英文标题
*   **文件名**: 2510.27656v1.pdf
*   **英文标题**: RDMA POINT-TO-POINT COMMUNICATION FOR LLM SYSTEMS

## 中文摘要
新兴的大型语言模型（LLM）系统模式，如分离式推理（disaggregated inference）、专家混合（MoE）路由和异步强化学习微调，需要超越简单集体操作的灵活点对点通信。现有的实现方案被锁定在特定的网络接口控制器（NICs）上，这阻碍了它们在推理引擎中的集成和跨硬件提供商的可移植性。我们提出了 **TransferEngine**，它桥接了通用 NICs 的功能，以提供一个统一的接口。TransferEngine 提供了单侧的 `WRITEIMM` 操作和一个用于完成通知的 `IMMCOUNTER` 原语，且不依赖网络传输的顺序假设，并能透明地管理每个 GPU 的多个 NIC。我们在 NVIDIA ConnectX-7 和 AWS Elastic Fabric Adapter (EFA) 上都展示了 400 Gbps 的峰值吞吐量。我们通过三个生产系统展示了 TransferEngine 的能力：（1）用于动态扩展的分离式推理中的 KvCache 传输；（2）在万亿参数模型上实现 1.3 秒完成的强化学习（RL）权重更新；（3）在 ConnectX-7 上实现了超越 DeepEP 解码延迟的 MoE 分发/合并（dispatch/combine）实现，并在 EFA 上首次实现了可行的延迟。我们证明了，我们可移植的点对点通信在补充集体操作的同时，避免了厂商锁定。

---

## 各章节主旨内容

*   **引言 (Introduction)**：指出现代LLM系统（如MoE、分离式推理）对灵活的点对点通信有强烈需求，而现有基于集体操作的库（如NCCL）和特定于硬件的RDMA库存在厂商锁定和可移植性差的问题。为解决此问题，论文提出了一个名为 **TransferEngine** 的可移植RDMA通信库。
*   **背景与相关工作 (Background and Related Work)**：介绍了RDMA技术（包括其操作、传输协议如RC/SRD）和现有的编程接口（如NCCL、GPUDirect）。回顾了在分离式推理、点对点网络库和计算-通信重叠方面的相关研究，明确了TransferEngine在可移植性和解决异构硬件（特别是ConnectX和EFA）差异方面的独特定位。
*   **TransferEngine**：详细阐述了TransferEngine的设计理念、架构和API。核心思想是构建一个统一的抽象层，屏蔽底层RDMA硬件（NVIDIA ConnectX-7和AWS EFA）的差异。它通过一个新颖的`IMMCOUNTER`原语来处理乱序网络中的完成通知，并透明地管理多NIC聚合，以实现高性能和可移植性。
*   **KvCache传输 (KvCache Transfer)**：展示了TransferEngine在**分离式推理**场景下的第一个应用。通过TransferEngine连接prefill和decode集群，实现了动态扩展和高效的KvCache分页数据传输，并支持在CUDA Graph中进行层级传输。
*   **强化学习轮换权重传输 (RL Rollout Weight Transfer)**：展示了TransferEngine在**异步强化学习微调**中的第二个应用。通过点对点的单侧`RDMA WRITE`，实现了从训练GPU到推理GPU的超大规模模型（万亿参数级别）权重的快速更新（1.3秒），速度远超现有框架。
*   **MoE分发/合并 (MoE Dispatch/Combine)**：展示了TransferEngine在**MoE模型**中的第三个应用。设计了基于主机代理（host proxy）的低延迟MoE dispatch/combine核，利用TransferEngine进行跨节点通信。该实现不仅在ConnectX-7上达到了业界顶尖水平，而且首次在AWS EFA上实现了有竞争力的性能。
*   **评估 (Evaluation)**：在配备NVIDIA H200 GPU、ConnectX-7和EFA NIC的硬件上，对TransferEngine本身以及上述三个应用场景进行了全面的性能评测。结果表明，TransferEngine成功地在不同硬件上实现了接近硬件极限的吞吐量，并且其应用在各自领域都达到了最先进（SOTA）的性能。
*   **结论 (Conclusion)**：总结了论文的贡献。TransferEngine通过识别异构RDMA硬件的共同功能，提供了一个无序但可靠的抽象层，成功解决了LLM系统中RDMA通信的厂商锁定问题。通过三个生产级别的应用案例，证明了该方法在实现可移植、高性能的点对点通信方面的有效性。

## 中文结论
现有的用于LLM系统的RDMA解决方案存在厂商锁定的问题，在像AWS EFA这样的定制云硬件上没有可行的实现方案。**TransferEngine** 通过识别异构RDMA硬件间的共同功能来解决这个问题。通过在底层协议之上叠加一个无序但可靠的抽象层，我们透明地将支持扩展到多个RDMA NIC，特别关注了EFA和ConnectX。我们通过三个生产系统展示了这种方法：用于分离式推理的KvCache传输、实现1.3秒万亿参数模型更新的RL权重更新，以及在ConnectX-7上具有最先进延迟并在AWS EFA上首次兼容可行的MoE分发/合并。TransferEngine为现代LLM架构实现了可移植的点对点通信，避免了厂商锁定，同时补充了集体库在云原生部署中的不足。

---

## 各章节详解

### **第1章：引言 (Introduction)**

随着大型语言模型（LLM）的发展，新的系统架构模式不断涌现，例如专家混合（MoE）、分离式推理（Disaggregated Inference）等。这些新模式的共同特点是，它们依赖于**灵活、动态的点对点（Point-to-Point, P2P）通信**，而不仅仅是传统的、静态的集体（Collective）通信（如All-Reduce, All-Gather）。

然而，当前的LLM生态系统面临一个严峻挑战：**通信库的厂商锁定（vendor lock-in）**。
1.  **集体通信库的局限性**：像NCCL或`torch.distributed`这样的库虽然在数据并行等场景中表现出色，但它们的P2P功能有限，且其设计（如固定成员、同步初始化）不适合新兴的动态工作负载。
2.  **RDMA库的碎片化**：高性能计算长期以来使用基于RDMA的P2P原语，但LLM框架很少直接使用它们。关键障碍是硬件多样性缺乏统一的抽象。云服务商部署了不同的RDMA硬件：NVIDIA ConnectX网卡使用传统的RC（Reliable Connection）传输协议，提供有序传输；而AWS的EFA（Elastic Fabric Adapter）则使用自家的SRD（Scalable Reliable Datagram）协议，只保证可靠交付，不保证顺序。现有的RDMA库（如DeepEP, NVSHMEM）要么只支持特定硬件（如ConnectX），要么在EFA上性能严重下降或根本不支持。

这导致了一个后果：没有一个可行的、跨云服务商的P2P通信解决方案来支持LLM推理。

为了解决这个问题，本文提出了 **TransferEngine**，一个可移植的RDMA通信库。其核心洞察是：**尽管底层硬件不同，但它们都支持可靠但无序的交付**。ConnectX的RC可以配置为忽略顺序，而EFA的SRD本身就是无序的。

TransferEngine基于这一共同点，构建了一个统一的抽象层，并提供：
*   **统一接口**：同时支持ConnectX和EFA。
*   **透明的多NIC管理**：自动聚合多个NIC（如4个100Gbps的EFA）以达到与单个高性能NIC（如1个400Gbps的ConnectX-7）相当的带宽。
*   **新颖的完成通知机制**：引入`IMMCOUNTER`原语，以在无序网络中可靠地进行完成通知。

论文通过三个在生产环境中经过验证的系统，展示了TransferEngine的强大能力，证明了可移植的P2P通信是现代LLM工作负载中对集体通信的重要补充，并且可以避免厂商锁定。

### **第2章：背景与相关工作 (Background and Related Work)**

本章首先铺垫了理解本文所需的技术背景，并回顾了相关领域的研究。

**技术背景**:
*   **RDMA (Remote Direct Memory Access)**：一种网络技术，允许一台计算机的内存直接被另一台计算机访问，无需操作系统内核介入。这带来了极高的吞吐量和极低的延迟。RDMA操作分为两类：
    *   **双边操作 (Two-sided)**：如`SEND/RECV`，需要通信双方都参与。
    *   **单边操作 (One-sided)**：如`WRITE`，只需源头发起，直接写入目标内存，对端无需感知。`WRITEIMM`是`WRITE`的扩展，可以在写入数据的同时附带一个32位的立即数，用于通知。
*   **RDMA传输协议**:
    *   **RC (Reliable Connection)**：可靠、有序的连接。这是NVIDIA ConnectX网卡使用的标准协议。
    *   **SRD (Scalable Reliable Datagram)**：可靠、**无序**的数据报。这是AWS EFA的专有协议。
*   **GPUDirect RDMA**: 允许NIC直接访问GPU内存，避免了数据在CPU内存中的中转，是现代GPU集群通信的标配。

**相关工作**:
*   **集体通信库 (Collectives)**：如NCCL，虽然高效，但其P2P能力受限，存在固定成员关系、同步初始化、统一数据形状等约束，不适用于动态、稀疏的通信模式。
*   **分离式推理 (Disaggregated Inference)**：像Splitwise, DistServe, Mooncake等工作通过将LLM推理的prefill和decode阶段分离到不同集群来优化服务。本文的KvCache传输案例就是对这一模式的实现和扩展，提供了可移植的RDMA通信能力。
*   **P2P网络库**:
    *   **NVSHMEM**: 支持灵活的P2P通信，但在EFA上性能严重下降。
    *   **NIXL**: NVIDIA推出的新库，目标是LLM的P2P通信，但其EFA支持仍处于早期阶段。
    *   **Mooncake**: 提供了RDMA传输引擎，但不支持EFA。
    *   **UCCL, MSCCL++**: 主要关注对集体操作的网络层优化，而非通用的P2P通信。

与这些工作相比，TransferEngine的独特贡献在于它**专注于提供一个跨异构硬件（特别是ConnectX和EFA）的可移植、高性能的P2P通信抽象层**。

### **第3章：TransferEngine**

本章是论文的核心，详细介绍了TransferEngine的设计与实现。

**设计目标与核心思想**:
TransferEngine的目标是提供一个最小化的API，以屏蔽底层RDMA硬件的复杂性和异构性。它的核心设计决策是**不依赖于网络传输的顺序保证**。所有完成通知都通过一个名为 **`IMMCOUNTER`** 的新颖原语来处理。`IMMCOUNTER`是一个专用的计数器组件，它原子性地增加与特定立即数（immediate value）关联的计数。当发送方使用`WRITEIMM`发送数据时，接收方的NIC在收到数据和立即数后，会触发一个完成事件，`IMMCOUNTER`捕获这个事件并增加相应的计数。接收方应用通过查询这个计数器就能知道有多少个特定的`WRITEIMM`操作已完成。

**架构 (图1)**:
*   **多线程模型**: TransferEngine为每个GPU启动一个工作线程（Worker），该线程管理与此GPU关联的所有NIC。
*   **DomainGroup 和 Domain**: 一个`DomainGroup`对应一个GPU，管理该GPU下的所有`Domain`。每个`Domain`对应一个物理NIC，负责具体的RDMA操作，如队列对管理、工作请求提交和完成事件轮询。
*   **地址交换**: 每个TransferEngine实例暴露一个主地址。通信双方通过交换这个地址来发现彼此的所有NIC信息，从而实现多NIC间的负载均衡或分片传输。这对于EFA至关重要，因为它需要聚合多个低带宽NIC。

**API设计 (图2)**:
API被设计为异步和零拷贝的，主要包括：
*   **内存注册 (`reg_mr`)**: 注册GPU或CPU内存，使其能用于RDMA操作。返回一个可序列化的`MrDesc`（包含远程访问所需的密钥）和一个本地句柄`MrHandle`。
*   **P2P传输**:
    *   `submit_send`/`submit_recvs`: 封装了`SEND/RECV`，用于RPC式的短消息通信。
    *   `submit_single_write`/`submit_paged_writes`: 核心的单边写操作，支持连续或分页（非连续）的数据写入。可以附带一个可选的`imm`值来触发`IMMCOUNTER`。
*   **组操作**: `submit_scatter`（向一组对等方分发数据）和`submit_barrier`（仅通知）。
*   **GPU同步 (`alloc_uvm_watcher`)**: 提供一个基于UVM（统一虚拟内存）的机制，允许CPU线程通过轮询（使用GDRCopy以降低延迟）来感知GPU内核的进度，从而触发后续的RDMA传输。

**硬件特定优化**:
*   **AWS EFA**: 通过`libfabric`库实现支持。由于EFA的特殊要求，即使是零字节的`WRITEIMM`也需要一个有效的目标描述符。
*   **NVIDIA ConnectX-7**: 通过`libibverbs`库实现支持。为每个对等方创建两个RC队列对，一个用于`SEND/RECV`，一个用于`WRITE/WRITEIMM`，避免干扰。同时启用了工作请求链（WR chaining）和乱序PCIe传输（`RELAXED_ORDERING`）来降低延迟。

### **第4章：KvCache传输 (KvCache Transfer)**

本章介绍了TransferEngine的第一个生产应用：在**分离式推理**中传输KV Cache。

**场景描述**:
在分离式推理架构中，LLM服务被拆分为两个集群：
1.  **Prefill集群**: 处理用户输入的prompt，生成初始的KV Cache。这是一个计算密集型任务。
2.  **Decode集群**: 接收Prefill集群生成的KV Cache，并进行后续的自回归解码（一次生成一个token）。这是一个内存带宽密集型任务。

这种分离允许根据不同阶段的资源需求独立扩展集群，提高系统整体的吞吐量和利用率。这两个集群之间的关键纽带就是**高效的KV Cache传输**。

**使用TransferEngine的实现 (图3)**:
1.  **请求调度**: 全局调度器接收请求，并为之分配一个Prefill GPU和一个Decode GPU。
2.  **资源预分配与请求分发**: Decode GPU预先分配好存储KV Cache的内存页。然后，它通过`submit_send`向指定的Prefill GPU发送一个请求，告知其应该将生成的KV Cache写入到哪些内存页地址。
3.  **分层/分块传输**: 在Prefill阶段，模型逐层计算。每当一层的KV Cache计算完成，Prefill GPU上的**UVM Watcher**机制就会被触发。CPU侧的TransferEngine工作线程检测到这个变化后，立即通过`submit_paged_writes`将这一层的KV Cache页传输到Decode GPU预先指定的地址。这实现了计算和通信的流水线化。
4.  **完成与解码**: 当所有KV Cache页都传输完毕后，Prefill GPU通过`submit_single_write`传输最后的上下文信息（如logits）。Decode GPU上的TransferEngine通过`expect_imm_count`得知所有传输已完成，随即开始解码。

这种设计的优点是：
*   **动态扩展**: 节点可以随时加入或离开，无需像集体通信库那样进行全局同步。
*   **低延迟**: 通过分层流水线传输和零拷贝RDMA，最大化地重叠了计算和通信。
*   **CUDA Graph兼容**: UVM Watcher机制可以无缝集成到CUDA Graph中，进一步减少CPU开销。

### **第5章：强化学习轮换权重传输 (RL Rollout Weight Transfer)**

本章介绍了TransferEngine的第二个生产应用：在**大规模异步强化学习（RL）微调**中进行权重更新。

**场景描述**:
在异步RL微调中，同样存在两个角色集群：
1.  **训练集群**: 不断地使用新数据对模型进行微调，产生新版本的模型权重。
2.  **推理（Rollout）集群**: 使用最新的模型权重与环境互动，生成新的训练数据。

这里的核心挑战是：如何**快速地**将训练集群产生的新权重同步到推理集群，以减少策略延迟（policy lag），从而提高训练效率。对于万亿参数级别的模型，一次权重更新可能涉及数百GB甚至TB级别的数据传输。

**使用TransferEngine的实现 (图4, 5)**:
*   **传统方法的瓶颈**: 现有框架通常将所有训练和推理GPU组成一个大的通信组。权重更新时，首先在训练集群内部将权重汇集到Rank 0，然后由Rank 0广播到所有推理节点。这使得训练集群Rank 0的NIC成为整个系统的瓶颈。
*   **TransferEngine的点对点方法**:
    1.  **静态调度**: 在初始化时，控制器计算出一个静态的传输计划，明确每个持有部分权重的训练GPU应该将其数据直接发送到哪个（或哪些）推理GPU。
    2.  **并行单边写入**: 在每个训练步骤之后，控制器发出信号，所有训练GPU根据预定的计划，同时通过并行的单边`RDMA WRITE`操作，将自己的那部分权重直接写入到目标推理GPU的内存中。
    3.  **无感更新**: 整个过程对推理GPU是**无感的**（one-sided），它们无需参与通信协调，从而可以不间断地进行推理任务。

**流水线执行 (图5)**:
为了进一步优化，权重传输的每个任务被拆分为四个流水线阶段：
1.  **H2D Memcpy**: 如果权重被FSDP卸载到了CPU，先将其拷回GPU。
2.  **参数准备**: 在GPU上对权重进行必要的处理，如反量化、融合等。
3.  **RDMA传输**: 通过TransferEngine进行零拷贝的`WRITE`操作。
4.  **全局屏障**: 使用Gloo在以太网上进行同步，确保所有传输完成。

通过这种P2P和流水线化的方法，论文在万亿参数模型上实现了**1.3秒**的权重更新，比现有框架快了**超过100倍**。

### **第6章：MoE分发/合并 (MoE Dispatch/Combine)**

本章介绍了TransferEngine的第三个，也是最复杂的一个生产应用：为**专家混合（MoE）模型**实现低延迟的token分发（Dispatch）和合并（Combine）。

**场景描述**:
在MoE模型的解码阶段，每个token需要根据路由（routing）结果被发送到指定的“专家”（Expert）GPU进行处理，处理完后再将结果合并回来。这个过程对通信延迟极其敏感。

**设计挑战**:
*   **延迟**: 必须在微秒（µs）级别完成通信。
*   **可移植性**: 需要同时在ConnectX和EFA上高效运行。
*   **与SOTA竞争**: DeepEP是当时最先进的MoE通信库，但它深度绑定了ConnectX硬件特性（IBGDA），无法在EFA上运行。

**使用TransferEngine的实现 (图6, 7)**:
本文设计了一套基于**主机代理（host proxy）**的MoE核，它虽然引入了GPU-CPU-NIC的交互路径，但通过一系列精巧的设计将代理开销降至最低。

**架构与流程**:
1.  **两阶段分发**:
    *   **第一阶段（路由交换与推测性发送）**: 所有GPU首先交换路由信息（每个token要去哪个专家）。这个过程延迟很小。为了隐藏后续处理路由信息的CPU延迟，每个GPU会**推测性地**向其他每个GPU发送一小部分（例如24个）token到私有缓冲区（private buffer）。
    *   **第二阶段（批量发送）**: CPU侧的代理线程在收到路由信息后，计算出每个GPU需要接收的token的确切位置，然后发起第二次、更大规模的RDMA `scatter`操作，将剩余的token直接写入到对端GPU的连续接收缓冲区（contiguous receive buffer）中。
2.  **主机代理协调**: GPU核负责准备数据和计算路由，然后通过UVM信令通知CPU侧的代理线程。代理线程接收到信号后，调用TransferEngine的API来发起RDMA传输。
3.  **NVLink优化**: 在节点内部，token的交换优先通过低延迟的NVLink进行，减少对NIC的压力。
4.  **批量与流水线**: 所有的RDMA操作都被设计为批量的`scatter`写，以最大化网络利用率。整个过程（GPU计算、CPU代理、NIC传输）被流水线化，以隐藏各部分的延迟。

**结果**:
*   **SOTA性能**: 尽管有主机代理的开销，该实现在ConnectX-7上依然超越了专门优化的DeepEP，成为新的SOTA。
*   **首次在EFA上可行**: 更重要的是，这是**第一个在AWS EFA上实现了有竞争力性能**的MoE通信实现，证明了TransferEngine的可移植性和有效性。

### **第7章：评估 (Evaluation)**

本章提供了详尽的实验数据来支撑论文的论点。实验环境为配备NVIDIA H200 GPU、400Gbps ConnectX-7或2x200Gbps EFA NIC的顶级服务器。

**7.1 点对点通信性能 (图8, 表2)**:
*   **吞吐量**: TransferEngine在ConnectX-7和EFA上都能达到接近硬件理论峰值（约400Gbps）的吞吐量。与NVIDIA的NIXL库相比，性能相当，甚至略快。
*   **饱和点**: 结果显示，使用分页写（paged writes）比单次大块写（single write）能更快地达到饱和带宽。例如，64KB的分页写就能基本跑满带宽，而单次写则需要16MB以上。这对于传输KV Cache这样的小块分页数据非常有利。
*   **硬件差异**: EFA比ConnectX需要更大的消息尺寸才能饱和，这解释了为什么在MoE这种小消息场景下，EFA的延迟会略高。

**7.2 MoE分发/合并性能 (图9, 10, 11, 12)**:
*   **解码延迟 (Decode Latency, 图11)**: 这是评估的重点。
    *   在**跨节点**场景（16, 32, 64 GPU）下，本文提出的基于TransferEngine的MoE核在ConnectX-7上**全面优于**最先进的DeepEP。
    *   在EFA上，虽然延迟比ConnectX-7高约30%（符合硬件特性），但性能依然非常出色，并且是**唯一可行的**高性能方案（NVSHMEM在EFA上慢到无法使用）。
    *   与另一个可移植库`pplx-kernels`（基于NVSHMEM）相比，本文方法快了一个数量级。
*   **私有缓冲区大小的影响 (图9)**: 实验证明，推测性发送少量token（24-32个）确实能有效隐藏路由交换的延迟，验证了设计的有效性。
*   **预填充延迟 (Prefill Latency, 图12)**: 在预填充（大批量token）场景下，DeepEP由于其针对性的优化（如NVLink预聚合）表现更好。但本文的方法无需任何调整，依然保持了良好的性能，显示了其通用性。

**总结**: 评估结果强有力地证明了TransferEngine不仅成功地实现了跨硬件的可移植性，而且在性能上达到了业界顶尖水平，甚至在某些关键场景下超越了特定于硬件的优化库。

### **第8章：结论 (Conclusion)**

论文最后对整个工作进行了总结。当前LLM系统的RDMA通信方案普遍存在**厂商锁定**的问题，缺乏在AWS EFA等定制云硬件上的高性能实现。

本文通过提出**TransferEngine**，成功地解决了这一痛点。其核心贡献在于：
1.  **识别并利用了异构RDMA硬件的共同点**：即它们都支持**无序但可靠**的传输。
2.  **构建了一个可移植的抽象层**：该抽象层通过`IMMCOUNTER`等创新机制，屏蔽了底层硬件（ConnectX vs EFA）和协议（RC vs SRD）的差异，并能透明地管理多NIC聚合。
3.  **通过三个生产级应用验证了其有效性**：
    *   **KvCache传输**：实现了高效、动态的分离式推理。
    *   **RL权重更新**：在万亿参数模型上实现了创纪录的1.3秒更新。
    *   **MoE通信**：在ConnectX-7上刷新了SOTA延迟记录，并首次在EFA上实现了可行的高性能方案。

最终，TransferEngine证明了为现代LLM架构构建**可移植、高性能的点对点通信**是完全可行的。它不仅避免了厂商锁定，还作为集体通信库的重要补充，为云原生环境下的LLM部署提供了更灵活、更强大的通信基础。
