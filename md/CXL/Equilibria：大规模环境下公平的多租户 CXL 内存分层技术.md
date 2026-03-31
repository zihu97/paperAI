# 2602.08800v1.pdf - Equilibria: Fair Multi-Tenant CXL Memory Tiering At Scale

## 中文标题：Equilibria：大规模环境下公平的多租户 CXL 内存分层技术

### 中文摘要
随着数据中心内存需求不断增长，其成本和功耗已成为严峻挑战。基于 CXL 的内存扩展提供了一种低成本、低功耗的
解决方案。然而，在超大规模数据中心（Hyperscaler）部署 CXL 时，现有的内存分层方案在多租户环境下表现出
明显局限性：缺乏租户间的公平性保障、控制面灵活性不足以及可观测性匮乏。本文提出了 Equilibria，这是首个
直接解决上述挑战的操作系统框架，旨在实现大规模数据中心下公平的多租户 CXL 内存分层。Equilibria 提供
了基于容器的公平份额分配、精细的可观测性、受控的晋升（Promotion）与降级（Demotion）策略，并能有效
缓解内存抖动带来的负面影响。在大型数据中心的生产负载和标准基准测试中，Equilibria 相比 Linux 现有的
分层方案（TPP），在生产负载上性能提升达 52%，在基准测试中提升达 1.7 倍，并已将相关补丁发布至 Linux 社区。

### 各章节主旨内容
- **Chapter I: Introduction** - 阐述了内存占数据中心 TCO 和功耗比例过高的问题，引出 CXL 内存分层技术的
  必要性，并指出多租户场景下现有分层机制在公平性、灵活性和可观测性上的缺失。
- **Chapter II: Background** - 介绍了数据中心内存开销背景、CXL 技术特性，以及当前 Linux 系统对内存分层
  和容器多租户的支持现状。
- **Chapter III: Learnings from Deploying CXL in Production** - 分享了在 Meta（corpX）真实生产环境中
  部署 CXL 的经验教训，包括真实 CXL 与模拟硬件的性能差异、带宽瓶颈并非主要限制、以及多租户下显著的
  性能干扰和公平性问题。
- **Chapter IV: Equilibria Design** - 详细介绍了 Equilibria 的设计，包括其公平性定义、基于 cgroup 的
  配置接口、精细的可观测性计数器，以及核心的降级调制、晋升调节和抖动缓解机制。
- **Chapter V: Evaluation** - 在小型和大型服务器上对 Equilibria 进行验证，评估了其在微基准测试、
  SparkBench、TaoBench 以及真实生产负载（Cache, Web, CI）下的性能表现。
- **Chapter VI: Related Work and Discussions** - 比较了 Equilibria 与 TPP、Memtis、Pond 等现有
  内存分层方案的异同，强调了 Equilibria 在多租户公平性和上游一致性方面的优势。
- **Chapter VII: Conclusion** - 总结了 Equilibria 的贡献，强调其在提升多租户环境内存效率和保障 SLO 方面
  的显著效果。

### 中文结论
Equilibria 为 CXL 内存分层提供了一种多租户公平性解决方案。它通过精细的每容器（Per-container）控制和
可观测性，结合灵活的公平策略及抖动抑制机制，有效解决了多租户环境下的性能不确定性和干扰问题。实验表明，
Equilibria 能够帮助负载在共享 CXL 内存时满足其业务水平目标（SLO），并显著优于现有的 TPP 方案，目前正
在 Meta 生产集群大规模部署并回馈 Linux 社区。

---

# 章节详细解读

## 第一章：引言与背景 (Introduction & Background)

### 1. 设计思想 (Design Philosophy)
- **痛点分析**：内存占数据中心成本高达 44%，CXL 虽能提供扩展，但硬件本身引入了更高的访问延迟（相比本地
  DRAM 慢）。现有 OS（如 Linux 及其 TPP 扩展）主要关注系统全局性能最大化，即尝试将系统最热的页面放
  入本地内存。在多租户场景下，这会导致“先到先得”或“热者恒强”的不公平现象，新启动的或访问稍冷但仍有
  SLO 要求的租户会被挤占到 CXL 内存，导致性能受损。
- **核心动机**：在超大规模数据中心，多租户（Stacking）是提高硬件利用率的关键。需要一种能够感知租户需求、
  保障租户“公平份额”本地内存的机制，使得即使在资源竞争激烈时，每个容器也能维持预期的性能。
- **设计目标**：实现透明的、高性能的、且具备多租户公平保障的内存分层框架。

### 2. 关键技术 (Key Technologies)
- **CXL 拓扑抽象**：将 CXL 内存呈现为无 CPU 的远端 NUMA 节点，利用 OS 的分层机制进行页面迁移。
- **透明内存分层**：通过 OS 自动监测页面热度并在本地（Tier 0）和 CXL（Tier 1）之间迁移，无需应用修改代码。

## 第三章：生产环境中的经验教训 (Learnings from Production)

### 1. 核心发现
- **真实 CXL vs. 模拟**：作者发现真实的 CXL 硬件延迟远高于简单的 CPU 核禁用模拟方案。真实 CXL 闲置延迟
  高出 27ns，负载下高出 37ns。这意味着真实环境对分层策略的效率更为敏感。
- **容量绑定而非带宽绑定**：对于大多数互联网业务（如 Cache, Web），其主要痛点是内存容量不足。实验显示，
  即使应用占用了大量 CXL 容量，其驱动的 CXL 总线带宽也非常低（仅几 GBps）。因此，设计重点应放在
  容量管理和透明迁移，而非追求极致的 CXL 带宽利用。
- **公平性缺失的恶果**：在多租户环境下，由于缺乏隔离，一个产生大量热页面的“吵闹邻居”（Noisy Neighbor）
  可以夺走几乎所有本地内存，导致同机其他容器的性能暴跌（性能损失高达 65%）。

## 第四章：Equilibria 设计 (Equilibria Design)

### 1. 设计思想 (Design Philosophy)
- **公平性定义**：确保每个租户无论何时启动、与谁共存，都能获得足够的本地内存份额以满足其 SLO。
- **核心逻辑**：引入“本地内存保障”（Lower Protection）和“本地内存上限”（Upper Bound）两个关键维度。

### 2. 关键技术 (Key Technologies)
- **每容器可观测性**：在 cgroup 层面增加了对本地内存和 CXL 内存使用量的实时统计计数器，以及降级和晋升
  活动的次数记录。这为自动化运维和故障诊断提供了基础。
- **调制降级 (Modulating Demotions)**：
    - 引入 $d_{scan}$ 公式：$d_{scan} = N_{lru} \cdot \frac{N_{cgroup} - N_{protection}}{N_{cgroup}}$。
    - 逻辑：当系统需要腾出本地内存空间时，Equilibria 会根据容器当前本地内存量相对于其保障值（Protection）
      的超出比例来决定其被扫描降级的频率。低于保障值的容器会被豁免，确保“弱势”租户的性能。
- **调节晋升 (Regulating Promotion)**：
    - 引入晋升抑制：当容器的本地内存使用量已超过保障值且系统资源紧张，或者超过了其上限值时，Equilibria
      会按比例降低其页面晋升速度。
    - 公式：$P_{scan} = P_{base} \cdot \min((\frac{N_{protection}}{N_{cgroup}})^4, \frac{1}{16})$。
    - 这种四次方的衰减确保了超出份额越多，晋升越困难，防止其无节制抢占资源。
- **抖动缓解 (Thrashing Mitigation)**：
    - 通过采样技术（Hash Table 记录 PPN 和 Timestamp）检测页面是否在短时间内（如 200ms）在两层间反复搬运。
    - 一旦检测到抖动，会强制减半该容器的晋升速率，防止无效的迁移操作耗尽系统带宽和 CPU 周期。

## 第五章：实验评估 (Evaluation)

### 1. 对比分析
- **与 TPP (Linux Baseline) 对比**：
    - 在 Web 负载下，Equilibria 能维持稳定的本地内存分配，使得受影响的分区（Partition B）吞吐量提升 8%。
    - 在 Cache 负载（延迟敏感型）下，Equilibria 通过设定 70% 的本地内存保障，将 P99 延迟降低了约 50%，
      并将由于内存竞争导致的请求失败率从 83K/min 降回正常水平。
- **抖动抑制效果**：在故意构造的抖动场景下，开启 Equilibria 后，正常工作的邻居容器吞吐量提升了 7%，
  且系统全局的内存管理开销显著下降。

### 2. 实现细节
- 基于 Linux Kernel 6.11 开发，仅需约 1000 行代码修改，具有极高的可维护性和可集成性。
- 已将代码贡献至 Linux 上游（部分已合入），证明了其工程设计的合理性。

---
Equilibria 的成功在于其深刻理解了工业界在 CXL 演进过程中从“如何连上硬件”到“如何大规模公平分享硬件”的
需求转变。其提出的基于比例的降级和晋升控制逻辑，为未来异构内存管理提供了标准的 OS 范式。
