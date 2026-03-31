# Hummingbird：面向微秒级SLO的GPU抢占式调度系统

---

## 第一部分：论文整体骨架

### 1. 论文基本信息
*   **英文标题**：Hummingbird: SLO-Oriented GPU Preemption at Microsecond-scale
*   **中文标题**：Hummingbird：面向微秒级SLO的GPU抢占式调度系统
*   **文件名**：2601.04071v1.pdf

### 2. 中文摘要
现有的GPU共享技术（包括空分共享和时分共享）旨在提高GPU利用率，但由于闭源GPU上缺乏细粒度的任务调度能力，这些技术在同时确保服务等级目标（SLO）遵守和最大化效率方面面临巨大挑战。本文提出了Hummingbird，这是一个面向SLO的GPU调度系统，它通过在闭源GPU上实现微秒级抢占，同时有效地利用空闲的GPU时间片，克服了上述挑战。在不同GPU架构上的综合评估表明，与最先进的空分和时分共享方法相比，Hummingbird将高优先级任务的SLO达成率分别提高了9.7倍和3.5倍。与独占执行相比，在Hummingbird上与低优先级任务共置时，高优先级任务的SLO达成率仅下降不到1%。同时，低优先级任务的吞吐量比最先进的时分共享方法高出2.4倍。Hummingbird在增强GPU利用率的同时，展示了在确保SLO方面的显著有效性。

### 3. 各章节主旨内容

#### 1. 引言 (Introduction)
近年来，以Transformer为基础的深度神经网络（如ChatGPT和Gemini）日益普及，给GPU资源带来了巨大压力。为了保证SLO，工业界通常以粗粒度方式分配GPU，导致利用率极低（仅25%-52%）。现有的GPU共享技术在共置不同任务时，难以在遵守SLO和提高利用率之间取得平衡。主要原因是NVIDIA等闭源GPU缺乏细粒度的任务调度和资源隔离能力。本文提出了Hummingbird，利用微秒级抢占和对“小气泡”（空闲时间片）的利用来解决这一问题。

#### 2. 背景与观察 (Background and Observations)
*   **DNN Kernel时间的特征化**：DNN任务的Kernel执行时间具有高度异构性（从几微秒到几毫秒），但绝大多数线程块（Thread Block）的执行时间非常短（<400μs），且具有可预测性。这表明通过控制线程块数量来调节Kernel执行时间是可行的。
*   **GPU硬件限制**：闭源GPU（特别是NVIDIA）不支持内核提交后的主动抢占。空分共享（如MIG, MPS）存在资源干扰问题；时分共享（如REEF）则面临抢占延迟高和吞吐量受限的问题。

#### 3. 动机 (Motivation)
通过实验量化分析发现，空分共享（如Orion）因内存带宽争用严重影响高优先级任务的SLO；时分共享（如REEF）虽能改善干扰，但因等待Kernel完成或通过各种手段驱逐Kernel会导致不可预测的抢占延迟，且严重限制了低优先级任务的吞吐量。研究发现，GPU执行中存在大量微秒级的“小气泡”（由内存操作、同步、通信等引起），有效利用这些气泡是提升利用率的关键。

#### 4. 设计 (Design)
Hummingbird的设计包含三个核心组件：
*   **Kernel Splitter（内核切分器）**：分析低优先级任务的特性和硬件能力，确定每个Kernel的最佳切分大小，生成切分日志。
*   **Runtime Scheduler（运行时调度器）**：利用切分日志将Kernel切分为更小的子Kernel（split-kernels），通过气泡检测、动态合并和Kernel-tick调度策略，在微秒级内抢占低优先级任务，并填充空闲气泡。
*   **Memory Management（内存管理）**：通过扩展NVLink的统一内存架构，支持层级化内存卸载，确保存储隔离并减少干扰。

#### 5. 实现 (Implementation)
Hummingbird主要由C++/CUDA实现（约8000行代码）。它通过Hook CUDA Driver API（如`cuLaunchKernel`）来拦截和管理任务，无需修改应用源码。利用PTX汇编代码转换技术实现Kernel切分，并利用`cudaEvent`进行轻量级Profiling。

#### 6. 评估 (Evaluation)
在多种GPU（L40s, A100, H100）和多种负载（LLM推理、训练，CNN等）下的评估显示：
*   **SLO达成率**：Hummingbird实现了接近独占执行的SLO达成率（99%），远超对比基线。
*   **吞吐量**：低优先级任务的吞吐量比REEF高出1.9倍到2.4倍。
*   **开销**：抢占延迟控制在微秒级（平均<140μs），系统开销极低。
*   **扩展性**：在多GPU分布式环境下依然保持高效。

#### 7. 相关工作 (Related Works)
综述了空分共享（Spatial Sharing）和时分共享（Temporal Sharing）的代表性工作，指出了它们在细粒度控制、资源隔离和通用性方面的不足。

#### 8. 结论 (Conclusion)
Hummingbird是一个面向SLO的GPU调度系统，通过微秒级抢占和气泡填充技术，成功在闭源GPU上实现了高优先级任务的SLO保障和低优先级任务的高吞吐量。

### 4. 中文结论
本文提出了Hummingbird，这是一个面向SLO的GPU调度系统，它允许高优先级任务在闭源GPU（即NVIDIA）上以微秒级进行抢占，同时最大化GPU利用率。我们令人鼓舞的结果表明，Hummingbird可以随时应用于当今的GPU集群中。

---

## 第二部分：指定章节详解

### 第4章：设计 (Design)

Hummingbird的核心设计目标是在闭源GPU上实现微秒级抢占，同时通过利用空闲时间片（气泡）最大化利用率。本章详细阐述了三个关键组件的设计。

#### 4.1 总体架构
Hummingbird包含三个组件：
1.  **Kernel Splitter**：静态或离线分析低优先级Kernel，计算最佳切分粒度。
2.  **Runtime Scheduler**：在线调度，动态切分和合并Kernel，平衡抢占延迟和GPU利用率。
3.  **Memory Management**：基于NVLink扩展的内存管理，支持层级化卸载。

#### 4.2 Kernel Splitter（内核切分器）
此组件解决“如何找到最佳切分大小”的问题。
*   **最佳切分大小 (Optimal Split-Kernel Size)**：
    *   切分太小会导致GPU利用率不足（Underutilization）和启动开销增加；切分太大会增加抢占延迟。
    *   **计算方法**：
        1.  首先计算一个Split-Kernel应包含的最大线程块数（$N_{block}$），公式考虑了SM数量（$N_{SM}$）、每个SM的最大线程数（$SM\_MAX\_THREADS$）和Kernel的占用率（Occupancy, $o$）。
            $$N_{block} = N_{SM} \cdot o \cdot \frac{SM\_MAX\_THREADS}{THREADS\_PER\_BLOCK}$$
            这个公式旨在让启动的块数刚好填满GPU的计算资源或带宽。
        2.  从$N_{block}$开始逐步减少块数，观察Kernel执行时间。当减少块数能线性减少执行时间时，说明资源利用饱和；当时间不再明显减少时，说明达到了硬件启动开销的瓶颈。Hummingbird记录这个拐点作为最佳切分大小。
*   **PTX Kernel变换**：
    *   切分后的子Kernel（Split-Kernel）需要重新映射线程块索引（`blockIdx`）以保证计算正确性。
    *   Hummingbird通过PTX（Parallel Thread Execution）汇编层面的代码注入来实现。它修改Kernel参数以接受偏移量（offset），并在代码开头注入指令，将逻辑`blockIdx`加上偏移量，从而恢复原始的寻址逻辑。选择PTX层是为了保证跨编译器和框架的通用性。

#### 4.3 Runtime Scheduler（运行时调度器）
此组件解决“如何最大化GPU利用率同时保证SLO”的问题。
*   **高优先级任务调度**：一旦高优先级任务到达，调度器立即停止发射新的低优先级Split-Kernel。由于Split-Kernel执行时间被限制在微秒级，高优先级任务可以几乎立即获得GPU资源。
*   **低优先级任务调度与气泡填充**：
    *   **气泡检测 (Bubble Detection)**：Hummingbird利用“小气泡”（Small Bubbles）来运行低优先级任务。这些气泡通常来源于Host-Device同步（如`cudaStreamSynchronize`）、内存拷贝或通信延迟。调度器通过Hook相关API或插入标记事件（Marker Events）来实时感知气泡的开始。
    *   **Kernel-Tick调度策略**：为了避免每发射一个Split-Kernel都进行同步（`cudaStreamSynchronize`）带来的巨大开销（约5-7μs），Hummingbird引入了“Kernel-Tick”策略。
        *   它不等待每个Split-Kernel完成，而是根据预测的执行时间，在CPU侧计算发射下一个Split-Kernel的时间点。
        *   这形成了一个CPU-GPU流水线，使得Split-Kernel可以紧密衔接执行，极大地减少了同步频率和空隙。
    *   **动态合并 (Dynamic Consolidation)**：当检测到“大气泡”（Large Bubbles，如网络延迟引起）时，切分Kernel会显得低效。调度器会检测队列状态，如果一段时间内没有高优先级任务，它会将剩余的Split-Kernel动态合并回较大的块或原始大小进行发射，以消除启动开销。

#### 4.4 Memory Management（内存管理）
此组件解决共置任务时的显存容量和带宽争用问题。
*   **NVLink扩展的统一内存**：受HUVM启发，Hummingbird构建了一个包含本地HBM、NVLink连接的远端HBM和DRAM的层级化内存池。
*   **优先级隔离**：利用CUDA Driver API的放置偏好（Placement Preferences），优先保障高优先级任务分配在本地HBM。当显存不足时，只驱逐低优先级任务的页面。
*   **干扰感知的驱逐策略 (Interference-Aware Eviction)**：
    *   NVLink共享HBM带宽，如果不加控制地进行内存页迁移（Eviction），会干扰正在执行的高优先级任务。
    *   **Ping-like带宽监测**：Hummingbird部署了一个轻量级的全局监控器，周期性发送小数据包（Ping）在GPU间测量实时延迟。
    *   如果发现某条链路延迟显著增加（意味着带宽拥塞），调度器会避免将内存页驱逐到该路径上的Peer GPU，或者降级驱逐到系统DRAM，从而保护高优先级任务的显存带宽不受干扰。

### 第5章：实现 (Implementation)

*   **技术栈**：基于NVIDIA GPU，使用C++和CUDA开发。
*   **透明性 (Transparency)**：通过**CUDA API Hook**技术（拦截`cuLaunchKernel`, `cuMemAlloc`等），Hummingbird对上层应用完全透明，用户无需修改代码即可使用。
*   **Kernel Profiler**：内置轻量级Profiler，利用`cudaEvent`记录Kernel执行时间，辅助切分决策。
*   **Kernel Transformation**：基于NEUTRINO框架的探测引擎，在运行时拦截GPU二进制代码（ELF/FatBinary），反汇编为PTX，注入偏移量计算指令，再通过`ptxas`重组为机器码。这种方法避免了对源码的依赖。
*   **限制与应对**：
    *   对于闭源库（如cuBLAS），由于无法获取PTX，Hummingbird选择用开源替代品（如CUTLASS）进行替换以实现切分。
    *   对于包含全局同步（如`grid_group::sync()`）的Kernel，切分可能会破坏语义，Hummingbird会自动禁用对此类Kernel的切分。

这一设计和实现确保了Hummingbird既能深入硬件底层进行精细控制，又能保持对现有软件生态的广泛兼容性。
