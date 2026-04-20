# 2604.13743v1.pdf - OffloadFS: Leveraging Disaggregated Storage for Computation Offloading

## 中文标题：OffloadFS：利用解耦存储实现计算卸载

### 中文摘要
本文针对现代数据中心解耦存储（Disaggregated Storage）架构中，NVMe over Fabrics (NVMeoF) 存储节点的计算与内存
资源利用不足的问题，提出了 OffloadFS。OffloadFS 是一个用户级文件系统，能够将 IO 密集型任务卸载到存储节点进
行近数据处理（Near-Data Processing），同时也支持卸载到对等计算节点。该系统采用以启动端为中心的元数据管理
方案，无需复杂的分布式锁管理即可确保一致性。基于 OffloadFS，本文进一步开发了支持压缩卸载的键值数据库 
OffloadDB 和用于机器学习图像预处理的 OffloadPrep 库。实验结果显示，OffloadFS 显著提升了系统性能，
RocksDB 的吞吐量提升了 3.36 倍，机器学习预处理速度提升了 1.85 倍。

### 各章节主旨内容
- **Chapter I: Introduction** - 阐述了解耦存储中存储节点资源被浪费的现状，提出了利用 NVMeoF 存储节点多核 CPU 
  和富余内存进行计算卸载的动机，并指出了选择性卸载与缓存一致性管理的挑战。
- **Chapter II: Background** - 介绍了 NVMeoF 协议、共享磁盘文件系统、LSM 树（以 RocksDB 为例）的压缩机制以
  及机器学习流水线中的数据预处理瓶颈。
- **Chapter III: Design of OffloadFS** - 详细描述了 OffloadFS 的架构，包括启动端（Initiator）的元数据管理、
  任务卸载器（Task Offloader）以及目标端（Target）的卸载引擎（Offload Engine）和缓存。
- **Chapter IV: Design of OffloadDB** - 介绍了如何基于 OffloadFS 改造 RocksDB，通过“日志回收”（Log Recycling）
  技术将 MemTable 刷新和后台压缩任务卸载到存储节点，减少网络流量。
- **Chapter V: OffloadPrep** - 描述了一个轻量级库，用于将图像预处理任务（如缩放、裁剪）卸载到存储节点，
  解决 DNN 训练中的数据停顿（Data Stall）问题。
- **Chapter VI: Performance Evaluation** - 通过多节点集群实验，对比了 OffloadFS 与 OCFS2、GFS2 的性能，
  验证了其在不同工作负载下的可扩展性和卸载策略的有效性。
- **Chapter VII: Conclusion** - 总结了 OffloadFS 在简化缓存一致性、提升解耦存储资源利用率方面的贡献。

### 中文结论
OffloadFS 证明了在解耦存储架构中，通过简单的以启动端为中心的一致性协议，可以高效地利用存储节点闲置资源进行
计算卸载。该系统有效解决了传统共享文件系统在 NVMeoF 环境下锁开销大的问题，并为 LSM 树数据库和机器学习流水
线提供了显著的加速效果。

---

# 章节详细解读

## III. OffloadFS 的设计 (Design of OffloadFS)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：传统的共享磁盘文件系统（如 OCFS2）为了维护多节点并发访问的一致性，使用了复杂的分布式锁管理器
  （DLM）和 RPC 协议。这在 NVMeoF 这种低延迟环境中产生了巨大的软件栈开销，导致 IO 性能大幅下降，且存储节点的 
  CPU 和内存资源无法得到有效利用。
- **核心动机**：作者认为，在许多应用场景中（如单节点 DBMS），虽然需要卸载计算到远程，但并不需要多节点并行的
  无序访问。因此可以简化一致性模型，将元数据管理的权利完全交给“启动端”（Initiator），从而消除分布式锁。
- **设计目标**：构建一个轻量级、以启动端为中心的用户级文件系统，支持透明的任务卸载和高效的近数据缓存。

### 2. 关键技术 (Key Technologies)
- **架构拆解**：系统分为 Initiator（启动端）和 Target（存储节点）。Initiator 运行 Inode 表和范围管理器 
  (Extent Manager)；Target 运行卸载引擎 (Offload Engine) 和卸载缓存 (Offload Cache)。
- **任务卸载流程**：启动端通过 gRPC 将 IO 密集型任务（如数据的计算操作）发送至 Target 端的 RPC 骨架代码。
  Target 端使用 SPDK API 直接操作裸块设备，绕过内核文件系统栈。
- **以启动端为中心的缓存一致性**：Initiator 拥有分配块地址的绝对权力。在卸载任务期间，Initiator 跟踪任务
  访问的块地址，并禁止自身或其他节点访问这些冲突块，直到任务完成。这种方案消除了节点间频繁的缓存一致性心跳。

### 3. 实现细节 (Implementation Details)
- **API 接口**：提供了 `offload_read()` 和 `offload_write()` 供卸载任务调用。
- **多租户支持**：提供了两种策略：
    - **反应式策略**：基于阈值。如果 Target CPU 利用率过高，则拒绝卸载请求，任务退回到 Initiator 执行。
    - **主动式策略**：基于令牌（Token）。Target 端分发限量的令牌，持有令牌的 Initiator 才能发起卸载任务，
      确保了多 Initiator 间的公平性。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **优势**：极大地降低了元数据操作的延迟；利用了存储节点的空闲内存作为二层缓存（Offload Cache）。
- **劣势**：不适用于真正的分布式并行应用（如多个节点同时写入同一个文件的随机位置且无外部协调）。

## IV. OffloadDB 的设计 (Design of OffloadDB)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：RocksDB 在进行压缩（Compaction）时，需要从存储读取大量 SSTable 并在内存中排序再写回。这在 
  NVMeoF 环境下产生了巨大的网络流量，并造成计算节点的 CPU 抖动。
- **核心改进**：将 SSTable 的归并排序过程完全卸载到存储节点执行。
- **关键直觉**：由于 WAL 日志已经包含了最新的键值对，如果在存储节点直接从 WAL 重建 SSTable，可以避免从计算
  节点再次发送相同的数据。

### 2. 关键技术 (Key Technologies)
- **日志回收 (Log Recycling)**：这是 OffloadDB 的核心创新。通常刷写 L0 SSTable 需要从内存发送数据。
  OffloadDB 则向存储节点发送 WAL 的块地址，由存储节点的“日志回收器”直接读取 WAL 块，在存储节点内存中重构出
  排序好的 L0 SSTable，从而将网络流量从“发送数据”降级为“发送偏移量”。
- **L0 缓存**：在 Initiator 端保留一份不可变的 MemTable 缓存（L0 Cache），使得前台查询无需去远程读取刚刷回
  存储的 L0 文件。

### 3. 实现细节 (Implementation Details)
- **压缩流程**：Initiator 发起请求 -> Extent Manager 在存储上预分配空间 -> 发送 Victim SSTable 的块地址
  给 Target -> Target 执行归并 -> Target 返回新生成的元数据 -> Initiator 更新 MANIFEST 文件。

### 4. 权衡与对比 (Trade-offs & Comparison)
- **对比**：相比于直接在计算节点做压缩，OffloadDB 减少了约 50% 的网络写流量（因为省去了刷写 L0 的传输）。
- **权衡**：虽然减少了网络负载，但增加了存储节点的 CPU 负担。作者通过多租户策略（令牌制）来平滑这种开销。

---

## VI. 性能评估 (Performance Evaluation)

### 1. 实验配置
- **硬件**：9 节点集群（8 计算 + 1 存储）。InfiniBand 40/56GbE 网络。存储节点配备 24 个 NVMe SSD。
- **基准**：对比 OCFS2（Linux 官方集群文件系统）和 GFS2。

### 2. 核心实验结果
- **RocksDB 性能**：OffloadFS 比 OCFS2 快 2.2 倍。当开启压缩卸载（OffloadDB）并增加卸载深度时，
  吞吐量提升高达 3.36 倍。
- **ML 预处理**：在 ImageNet 训练任务中，OffloadPrep 将图像预处理（旋转、缩放）卸载到存储节点，
  使得计算节点的 CPU 可以完全专注于训练逻辑，整体处理速度提升 1.85 倍。
- **可扩展性**：实验证明，即使有 8 个 Initiator 同时卸载，通过 Token 机制，存储节点的利用率被控制在 80% 
  以下，且整体系统的吞吐量随节点数线性增长，直到 IO 带宽饱和。
- **缓存有效性**：通过对比有无 Offload Cache 的情况，证明了在存储节点缓存热块能显著降低长尾延迟。
