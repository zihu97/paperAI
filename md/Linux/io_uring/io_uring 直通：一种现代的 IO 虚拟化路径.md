# Part 1：论文整体骨架

**论文文件名**：LPC2022_uring-passthru.pdf
**英文标题**：uring-passthru: A modern I/O path for virtualization
**中文标题**：io_uring 直通：一种现代的 I/O 虚拟化路径
**中文摘要**：
随着存储硬件速度的飞速提升，传统的 I/O 虚拟化标准 `virtio` 已逐渐成为性能瓶颈。
`virtio` 协议栈中固有的软件开销，如频繁的虚拟机退出（VM-Exit）和逐个请求的处理模式，限制了虚拟机所能达到的 I/O 性能。
本文提出了一种名为 `uring-passthru` 的新型 I/O 虚拟化方案，旨在从根本上解决这一问题。
该方案巧妙地利用 Linux 内核中为高性能 I/O 设计的 `io_uring` 异步接口，创建了一条从虚拟机到宿主机物理设备的超低延迟路径。
其核心思想是将虚拟机内部产生的 `io_uring` 提交队列项（SQE）批量地、近乎零拷贝地“直通”到宿主机，
再由宿主机上的 `io_uring` 实例直接提交给物理设备。
通过这种方式，`uring-passthru` 极大地减少了软件开销和上下文切换，实现了接近原生硬件的 I/O 性能，
为云环境中的高性能存储应用提供了新的可能性。

**各章节主旨内容**：
*   **第一部分：动机 (Motivation)**
    *   分析了当前 I/O 虚拟化面临的挑战：硬件速度远超软件处理能力，`virtio` 成为瓶颈。并点出了 `io_uring` 作为解决方案的巨大潜力。

*   **第二部分：深入 virtio (virtio deep-dive)**
    *   回顾了 `virtio-blk` 的工作原理，详细剖析了其性能开销的来源，主要在于 virtqueue 的通知机制和请求-响应模型的软件处理路径。

*   **第三部分：io_uring 概述 (io_uring overview)**
    *   简要介绍了 `io_uring` 的核心设计：通过提交队列（SQ）和完成队列（CQ）两个环形缓冲区，实现了用户空间和内核之间的高效、批量化通信，最大限度地减少了系统调用。

*   **第四部分：uring-passthru 设计 (uring-passthru: the design)**
    *   本文的核心章节。详细阐述了 `uring-passthru` 的架构，包括一个新的 `virtio-uring` 设备、Guest 中的前端驱动以及 Host 中的 `vhost_uring` 后端，并描绘了 I/O 请求的全新处理流程。

*   **第五部分：性能 (Performance)**
    *   通过详实的性能测试数据（IOPS、延迟），将 `uring-passthru` 与 `virtio-blk` 和 SPDK（作为原生性能参考）进行对比，用数据证明了新方案的显著优势。

*   **第六部分：结论与未来工作 (Conclusion & Future work)**
    *   总结了 `uring-passthru` 的主要贡献和优势，并展望了未来的发展方向，如代码上游、支持更多 `io_uring` 操作以及在网络虚拟化等领域的应用。

**中文结论**：
`uring-passthru` 是一种面向未来的、高性能的 I/O 虚拟化路径。
它通过将 Guest 的 `io_uring` 指令直接映射到 Host 的 `io_uring`，成功地绕过了传统 `virtio` 协议栈中的诸多软件开销。
性能评估结果表明，该方案在 IOPS 和延迟方面均大幅超越了现有的 `virtio-blk`，达到了接近原生（bare-metal）的水平。
这证明了 `uring-passthru` 是一个极具前景的方向，能够有效弥合现代高速硬件与虚拟化软件之间的性能鸿沟。
未来的工作将聚焦于将该方案贡献到上游 Linux 内核，并探索其在更多场景下的应用潜力。

---

# Part 2：指定章节详解

### **第一部分：动机 (Motivation) 详解**

本节开宗明义，指出了当前虚拟化 I/O 领域的核心矛盾：
1.  **硬件发展迅猛**：现代 NVMe SSD 的性能已经达到数百万 IOPS（每秒读写操作次数）和数十微秒的延迟，硬件本身已经快如闪电。
2.  **软件成为瓶颈**：作为 I/O 虚拟化事实标准的 `virtio`，其设计初衷是为了通用性和兼容性，但在极致性能上已显疲态。
    Guest VM 通过 `virtio` 访问存储设备，其性能远不能发挥出物理硬件的全部潜力。

接着，文稿指出了 `io_uring` 带来的机遇。`io_uring` 是 Linux 近年来引入的最重要的特性之一，它是一个全新的异步 I/O 接口，其设计目标就是**极致的性能**。
它的优点在于：
*   **批量处理**：应用可以一次性向内核提交大量的 I/O 请求，而不是一次一个。
*   **零系统调用**：通过用户空间和内核共享的环形缓冲区（Ring Buffer），应用提交和接收 I/O 事件大多数情况下无需陷入内核，避免了昂贵的系统调用。
*   **灵活性**：`io_uring` 不仅支持 I/O 操作，还支持定时器、网络等多种异步事件。

因此，一个自然而然的想法诞生了：**能否利用 `io_uring` 的高效机制来改造 `virtio`，构建一条全新的虚拟化 I/O 通道？**
这便是 `uring-passthru` 的出发点。

### **第二部分：深入 virtio (virtio deep-dive) 详解**

为了说明 `uring-passthru` 的优越性，文稿首先对 `virtio` 的性能瓶颈进行了分析。以 `virtio-blk`（`virtio` 的块设备协议）为例，一个 I/O 请求的生命周期是：
1.  Guest VM 中的应用发起读/写请求。
2.  Guest 内核的块设备层将请求转换为一个 `virtio-blk` 请求。
3.  `virtio-blk` 前端驱动将请求描述符放入 `virtqueue`（一个共享的环形缓冲区）。
4.  前端驱动通过“踢”（kick）的方式（通常是一次 VM-Exit）通知 Host。
5.  Host 上的 `vhost` 内核模块被唤醒，从 `virtqueue` 中取出请求描述符。
6.  `vhost` 模块解析请求，并将其转发给 Host 内核的块设备层，最终提交给物理设备驱动。
7.  I/O完成后，Host 通过中断和相似的路径将完成事件通知回 Guest。

这里的性能开销主要在于：
*   **通知开销**：每一次“踢”和中断都可能涉及 VM-Exit 和 VM-Entry，这是主要的延迟来源。
*   **逐个处理**：`vhost` 后端通常需要逐个解析和处理 `virtqueue` 中的请求，难以进行深度优化。
*   **软件路径过长**：请求在 Guest 和 Host 的软件栈中穿梭，每一步都有软件处理开销。

### **第三部分：io_uring 概述 (io_uring overview) 详解**

本节简要介绍了 `io_uring` 的工作原理，以便听众理解 `uring-passthru` 的设计基础。
*   **核心组件**：`io_uring` 的核心是两个由用户空间和内核共享的环形缓冲区：
    *   **提交队列 (Submission Queue, SQ)**：应用将要执行的操作（封装在 SQE - Submission Queue Entry 结构体中）放入此队列。
    *   **完成队列 (Completion Queue, CQ)**：内核将操作完成的结果（封装在 CQE - Completion Queue Entry 结构体中）放入此队列，供应用读取。
*   **工作流程**：应用将一批 SQE 填入 SQ，然后通过一次 `io_uring_enter` 系统调用（或者在特定模式下无需调用）通知内核。
    内核从 SQ 中取出所有待处理的 SQE 并执行。完成后，将对应的 CQE 填入 CQ。应用可以随时检查 CQ 来获取已完成的事件，无需等待中断。
    这种模式将多次 I/O 操作的系统调用开销均摊到了**一次**，极大地提升了效率。

### **第四部分：uring-passthru 设计 (uring-passthru: the design) 详解**

本节是文稿的核心，展示了 `uring-passthru` 的创新架构。其目标是**让 Guest VM 中的应用感觉像在直接对 Host 的 `io_uring` 编程**。
1.  **新的 `virtio` 设备**：定义了一种新的 `virtio-uring` 设备类型。这个设备在 Guest 中看起来像一个块设备，但它接收的不是传统的 I/O 请求，而是 `io_uring` 命令。

2.  **Guest 前端驱动**：
    *   Guest VM 中的应用（如 QEMU、SPDK）正常使用 `io_uring` 接口。
    *   当这些 `io_uring` 命令的目标设备是 `virtio-uring` 块设备时，Guest 中的一个特殊前端驱动会拦截它们。
    *   该驱动**不会**在 Guest 内核中解释或执行这些 SQE，而是将它们**原封不动地**打包，放入 `virtqueue` 中。

3.  **Host 后端 (`vhost_uring`)**：
    *   Host 上的 `vhost_uring` 内核模块从 `virtqueue` 中取出成批的 SQE。
    *   关键步骤：`vhost_uring` **直接将这些来自 Guest 的 SQE 提交给 Host 自己的 `io_uring` 实例**，而这个 `io_uring` 实例的目标是真实的物理 NVMe 设备。
    *   由于 Guest 的 SQE 格式与 Host 的 SQE 格式基本兼容，这个过程几乎不需要任何转换，只是简单的内存复制或地址重映射。这便是“直通”（passthru）的含义。
    *   I/O 完成后，Host 的 `io_uring` 产生 CQE，`vhost_uring` 再将这些 CQE 打包送回 Guest。

**核心优势**：
*   **批量透传**：一次 VM-Exit 可以传递一大批 I/O 请求（SQEs），而不是一个。
*   **语义直通**：传递的是具有丰富语义的 `io_uring` 命令，而不是简单的读/写块号，Host 可以进行更深度的优化。
*   **软件路径最短**：绕过了 Host 上大部分的块设备和文件系统层，实现了从 `vhost_uring` 到物理设备的直接路径。

### **第五部分：性能 (Performance) 详解**

本节通过对比实验展示了 `uring-passthru` 的惊人效果。测试配置通常为在 Host 上创建一个 RAM-backed 的 null_blk 设备以消除物理硬件差异，专注于软件协议栈的性能。
*   **IOPS 对比**：
    *   在随机读场景下，随着并发请求（Queue Depth）的增加，`uring-passthru` 的 IOPS 持续飙升，轻松达到 `virtio-blk` 的 **2-3 倍**。
    *   `uring-passthru` 的性能曲线与 SPDK（一个用户态、轮询模式的 I/O 框架，常被用作原生性能的标杆）非常接近，证明它已经基本消除了虚拟化软件的开销。
*   **CPU 效率**：
    *   在达到相同的 IOPS 水平时，`uring-passthru` 所消耗的 CPU 资源远低于 `virtio-blk`。
    *   这意味着它是一种**更高效**的虚拟化方案，可以将更多的 CPU 资源留给客户的应用本身。
*   **延迟对比**：
    *   在延迟方面，`uring-passthru` 同样表现出色，其平均延迟和尾延迟（p99, p99.9）都显著低于 `virtio-blk`，表现出更好的稳定性和可预测性。

### **第六部分：结论与未来工作 (Conclusion & Future work) 详解**

文稿最后进行了总结并展望未来。
*   **结论**：`uring-passthru` 成功地将 `io_uring` 的高性能模型引入到虚拟化领域，是 `virtio` 的一次重大演进。它通过批量化和语义直通，构建了一条超高效的 I/O 路径，解决了长期存在的虚拟化 I/O 性能瓶颈。

*   **未来工作**：
    *   **代码上游**：将 `virtio-uring` 和 `vhost_uring` 的实现贡献到 Linux 内核主线和 QEMU 项目中，使其成为一个标准特性。
    *   **支持更多操作**：扩展该方案以支持 `io_uring` 提供的更丰富的操作，如网络 `send`/`recv`、`accept` 等，从而将该方案推广到网络虚拟化领域（`uring-net-passthru`）。
    *   **安全性增强**：进一步完善 `vhost_uring` 后端，对来自 Guest 的 `io_uring` 命令进行更严格的安全检查，防止恶意虚拟机发起危险操作。
