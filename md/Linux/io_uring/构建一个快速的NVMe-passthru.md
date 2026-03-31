论文文件名：lpc-2021-building-a-fast-passthru.pdf
英文标题：Building a fast NVMe passthru
中文标题：构建一个快速的 NVMe Passthru 通道
中文摘要：本文探讨了如何利用 `io_uring` 构建一个高性能的 NVMe passthru 接口。传统的基于 `ioctl` 的 passthru 机制是同步阻塞的，对于高速 NVMe 设备而言效率低下。为了解决这个问题，本文提出了一种基于 `io_uring` 的异步 passthru 机制，通过新的 `URING_CMD` 操作码和 NVMe 字符设备 (`/dev/ngXnY`)，实现了从用户空间直接向 NVMe 设备提交命令的零拷贝、高并发路径，显著提升了 I/O 性能，使其接近 SPDK 等内核旁路方案的水平。
各章节主旨内容：
*   **NVMe Generic Device**：介绍了现有 NVMe 块接口的局限性，并引入了新的 NVMe Generic Character Device (`/dev/ngXnY`)，它作为一种更灵活的、始终可用的内核内 I/O 路径，为实现高性能 passthru 提供了基础。
*   **Async IOCTL 与 io_uring**：详细解释了 `io_uring` 作为可扩展异步 I/O 框架的核心思想，并介绍了为实现异步 `ioctl` 而引入的新用户接口 `IORING_OP_URING_CMD`，它允许用户将类 `ioctl` 命令提交到 `io_uring` 中异步执行。
*   **从同步到异步 Passthru**：对比了传统 `ioctl` passthru 和基于 `io_uring` 的异步 passthru。传统方式通过阻塞等待实现“同步”，严重限制了性能。新方法利用 `io_uring` 的 `uring_cmd` 机制，将命令提交与完成解耦，从而消除了阻塞，释放了 NVMe 设备的原生异步能力。
*   **性能优化**：探讨了如何利用 `io_uring` 的高级特性进一步优化 passthru 性能，例如使用 Fixed Buffers 减少每次 I/O 的内存映射开销，以及使用 Async Polling 实现无中断的 I/O 完成处理，最大化降低延迟。
*   **性能评估与结论**：通过基准测试展示了 `uring-passthru` 相较于传统 `ioctl` 和内核旁路方案 SPDK 的性能。结果表明，`uring-passthru` 的性能远超 `ioctl`，在启用 polling 后，其性能足以媲美 SPDK，证明了该方案在提供内核便利性的同时也能实现极致性能。

中文结论：本文成功设计并实现了一个基于 `io_uring` 的高性能 NVMe passthru 机制。该机制通过引入异步 `ioctl` 支持和利用 `io_uring` 的高级特性，有效解决了传统 `ioctl` 阻塞模型所带来的性能瓶颈。性能测试结果证实，该方案的 IOPS 表现卓越，在队列深度较低时便能使设备性能饱和，其性能水平可与内核旁路方案（如 SPDK）相媲美。这为需要直接访问 NVMe 设备的应用（如数据库、虚拟化）提供了一个高效、灵活且易于使用的内核内解决方案。未来的工作将继续探索 `bio-cache` 和直接 DMA 等优化路径。

---

## **各章节详解**

### **第一章：NVMe Generic Device：背景与动机**

传统的 Linux 内核 NVMe 接口是块设备（如 `/dev/nvme0n1`）。这种接口虽然通用，但存在一些限制：
1.  **受限于内核规则**：块设备可能因为某些原因（如不支持的格式、零容量等）被内核标记为隐藏或只读，导致用户空间无法访问。
2.  **抽象层次过高**：对于希望利用 NVMe 特定新功能（如 ZNS、KV command set）的应用程序，块设备抽象可能无法提供足够灵活的接口。

为了解决这些问题，内核从 5.13版本开始引入了 **NVMe Generic Character Device**（字符设备），设备节点命名为 `/dev/ngXnY`（X是控制器号，Y是命名空间ID）。这个接口具有以下特点：
*   **与命名空间一对一**：每个 NVMe 命名空间都有一个对应的字符设备。
*   **始终可用**：不受块设备层规则的限制，只要命名空间存在，该设备就可用。
*   **内核内路径**：与 SPDK 等完全绕过内核的方案不同，`ng` 设备提供的是一条内核内的 I/O 路径。这使得应用在享受高性能的同时，依然可以利用内核成熟的文件系统、网络栈等基础设施。
*   **Passthru 接口**：它允许应用程序通过 `ioctl` 直接向硬件发送 NVMe 命令，为早期技术采用者和需要特定功能的上层应用（如数据库）提供了一条“捷径”。

然而，最初这个 passthru 接口依赖于同步的 `ioctl` 调用，这成为了性能瓶颈。

### **第二章：`io_uring` 与异步 `ioctl` 接口**

`io_uring` 是 Linux 内核中一个高度可扩展的异步 I/O 框架。其核心设计思想是通过两个共享内存环形缓冲区（Submission Queue - SQ 和 Completion Queue - CQ）来最小化用户空间和内核空间之间的切换和数据拷贝。

为了将 `ioctl` 的功能异步化，`io_uring` 引入了一个新的操作码（opcode）：`IORING_OP_URING_CMD`。
*   **新的SQE类型**：这个操作码对应一种特殊的提交队列项（SQE），称为 `struct io_uring_cmd_sqe` (CSQE)。它内部有40字节的“载荷”空间（pdu），可以直接存放 `ioctl` 的命令和参数。
*   **命令提交流程**：
    1.  用户从 SQ 获取一个 SQE。
    2.  将其 `opcode` 设置为 `IORING_OP_URING_CMD`。
    3.  设置目标文件描述符 `fd`（例如 `/dev/ng0n1` 的 fd）。
    4.  在 CSQE 的载荷空间（`pdu`）中填充 `ioctl` 相关的命令码和参数。如果参数过大，也可以通过指针传递。
    5.  提交 SQE。
*   **内核处理**：
    1.  `io_uring` 框架收到 CSQE 后，不会像传统 `ioctl` 那样阻塞等待，而是会调用文件操作集（`file_operations`）中新增的一个函数指针：`uring_cmd`。
    2.  `int (*uring_cmd) (struct io_uring_cmd *, enum io_uring_cmd_flags);`
    3.  设备驱动（如 NVMe 驱动）实现这个 `uring_cmd` 方法，从中解析出命令并异步地发送给硬件。

这个机制将 `ioctl` 从一个同步调用转变成了一个可以被 `io_uring` 调度的异步任务。

### **第三章：实现异步 NVMe Passthru**

将 `io_uring` 的异步 `ioctl` 能力应用到 NVMe passthru 上，就构成了本文的核心。

*   **传统 `ioctl` Passthru 的问题**：
    *   NVMe 硬件本身是纯异步的（主机提交命令到 SQ，设备在未来的某个时间点将完成信息放入 CQ）。
    *   而 `ioctl` 是同步的。为了弥合这个鸿沟，NVMe 驱动在收到 `ioctl` passthru 命令后，会提交命令给硬件，然后**阻塞等待**，直到硬件返回完成信号。这种“同步转异步再转同步”的模式，使得应用程序无法并发提交多个命令，严重浪费了 NVMe 设备强大的并行处理能力。

*   **基于 `uring_cmd` 的异步 Passthru**：
    1.  NVMe 驱动实现了 `nvme_ns_chr_fops` 中的 `uring_cmd` 方法（在代码中为 `nvme_ns_chr_async_ioctl`）。
    2.  当 `io_uring` 调用此方法时，驱动程序从 `io_uring_cmd` 结构中提取出 NVMe 命令。
    3.  驱动将命令提交到硬件的提交队列（SQ），然后**立即返回 `-EIOCBQUEUED`**。这个返回值告诉 `io_uring` 框架：“我已经接受了这个请求，但它尚未完成，请不要阻塞”。
    4.  当硬件稍后完成该命令并发出中断时，NVMe驱动的中断处理程序会被触发。
    5.  为了将完成结果安全地传递回用户空间，驱动不能在中断上下文中直接修改用户内存。因此，这里利用了 `io_uring` 的 `task_work` 机制：驱动注册一个回调函数（`driver_cb`），`io_uring` 会在安全的用户上下文中执行这个回调。
    6.  回调函数执行时，可以将结果（如 I/O 状态、返回值）更新到用户提供的内存区域。
    7.  最后，通过 `io_uring_cmd_done()` 通知 `io_uring` 框架该命令已彻底完成，`io_uring` 随之将一个完成队列项（CQE）放入 CQ 环形缓冲区，用户空间便可得知命令已完成。

通过这个流程，提交和完成被完美解耦，实现了真正的异步 passthru。

### **第四章：利用 `io_uring` 高级特性进一步优化**

为了达到极致性能，该方案还利用了 `io_uring` 的两个高级特性：

1.  **Fixed Buffers（固定缓冲区）**
    *   **问题**：常规 I/O 需要在每次操作时动态地将用户空间的缓冲区（buffer）“钉”（pin）在物理内存中，并获取其 DMA 地址，I/O 结束后再“解钉”（unpin）。这个过程（`pin_user_pages`）有显著的性能开销。
    *   **解决方案**：`io_uring` 允许用户通过 `io_uring_register_buffers()` API 提前注册一组 I/O 缓冲区。内核只会 pin 这组缓冲区一次。之后提交 I/O 时，使用新的操作码 `IORING_OP_URING_CMD_FIXED`，并通过 `buf_index` 来指定使用哪个预注册的缓冲区。
    *   **实现**：NVMe 驱动的 `uring_cmd` 实现需要检查命令是否使用了固定缓冲区。如果是，它会从 `io_uring` 获取对应的 `bio_vec`（描述了预映射的内存），从而跳过 `pin_user_pages` 的开销。

2.  **Async Polling（异步轮询）**
    *   **问题**：基于中断的 I/O 完成通知本身有延迟。对于超低延迟存储，中断可能成为瓶颈。
    *   **解决方案**：`io_uring` 支持轮询模式（`IORING_SETUP_POLL`）。在这种模式下，内核线程会主动轮询硬件的完成队列，而不是等待中断。
    *   **实现**：
        *   **提交**：当用户提交一个 passthru 命令时，驱动选择一个支持轮询的硬件队列（hctx），并将一个“cookie”（包含 `io_uring_cmd` 的信息）与该队列关联，然后将命令发给硬件并禁用该队列的中断。
        *   **完成**：内核的 I/O 轮询线程（`kworker`）会定期检查硬件完成队列。一旦发现完成项，它会根据之前存储的 cookie 找到对应的 `io_uring_cmd`，然后触发 `io_uring` 的完成提交流程，最终用户从 `io_uring` 的 CQ 中收到完成事件。

### **第五章：性能评估**

性能测试（4KB 随机读）的结果令人印象深刻：
*   **ioctl**：性能最差，IOPS 始终在 100k 左右，无法随着队列深度的增加而提升，因为其同步阻塞的本质使其无法利用并发。
*   **uring-pt (uring-passthru)**：性能远超 `ioctl`。即使在队列深度为 1 时，性能已经很接近设备饱和点。
*   **spdk (Storage Performance Development Kit)**：这是一个知名的内核旁路方案，性能非常高。
*   **uring-pt-poll (开启轮询的 uring-passthru)**：性能与 `spdk` 基本持平。这表明，通过 `io_uring` 的异步机制和轮询优化，**内核内路径的性能已经可以媲美内核旁路方案**。

这个结果意义重大，因为它证明了我们可以在不放弃 Linux 内核丰富生态（如文件系统、安全性、管理工具）的前提下，获得接近裸金属的 I/O 性能。