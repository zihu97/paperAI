# 2603.00084v2.pdf - DeepXiv-SDK: An Agentic Data Interface for Scientific Literature

## 中文标题：DeepXiv-SDK：面向科学文献的智能体数据接口

### 中文摘要
本文介绍了 DeepXiv-SDK，这是一个专为大语言模型（LLM）智能体设计的科学文献数据接口。传统的智能体在访问学术
论文时面临数据非结构化、解析开销大、证据检索脆弱等瓶颈。DeepXiv-SDK 通过三层架构解决了这些问题：1) 数据层：将
非结构化的 HTML/PDF 转换为 JSON 格式的标准化、结构化表示；2) 服务层：提供可重用的工具，支持混合检索和渐进式数
据访问（如仅读取摘要或特定章节以节省 token）；3) 应用层：内置智能体，支持深层搜索和研究工作流。该系统目前涵盖了
整个 ArXiv 库并每日同步，支持 RESTful API、Python SDK 和 MCP 协议，旨在为 AI4Science 提供高效、低成本的数据基础设施。

### 各章节主旨内容
- **Chapter 1: Introduction** - 分析了 LLM 智能体在获取科学数据时的瓶颈，提出了“智能体友好”接口的三个核心原则：
  结构化规范、渐进式披露（按需读取）以及检索导向。
- **Chapter 2: Related Work** - 评述了现有的文献辅助工具（如 ar5iv, AlphaXiv），指出它们多为人类阅读设计，缺乏
  面向智能体的自动化接口 API。
- **Chapter 3: System: DeepXiv-SDK** - 详细阐述了系统的三层架构。数据层负责语料库级解析与信号丰富化；服务层提供
  统一协议支持成本可控的访问；应用层提供开发工具和内置智能体工作流。
- **Chapter 4: License** - 明确了版权立场：重用 ArXiv 的描述性元数据（CC0 1.0），而对论文正文则采用重定向至原
  始链接的策略，不进行全文分发以尊重版权。
- **Chapter 5: Evaluation** - 在“智能体搜索”和“深度研究 QA”两个任务上验证了系统。结果显示 DeepXiv 显著降低了
  Token 消耗和响应延迟，同时提升了答案质量。
- **Chapter 6: Conclusion** - 总结了 DeepXiv-SDK 如何将论文转化为“可调用对象”，并展望了其作为稳定服务对未来科学
  智能体生态的贡献。
- **Appendix A & B** - 提供了实现细节（如使用 MinerU 解析 PDF、Qwen3 提取信号）以及系统运行的统计数据（索引近
  300 万篇论文）。

### 中文结论
DeepXiv-SDK 成功地将大规模科学文献语料库转化为“智能体原生”的数据接口。通过将论文对象化、结构化，并提供按需
读取的渐进式访问机制，它有效地解决了智能体在处理长文档时的 Token 浪费和解析鲁棒性问题。实验表明，该系统能显著
提升研究智能体在复杂查询下的准确性与效率，是构建下一代科学研究自动化工具的关键基础设施。

---

# 章节详细解读

## 1. 系统架构：三层设计思想 (System Architecture)

### 1.1 设计思想 (Design Philosophy)
- **痛点分析**：现有的智能体工作流通常是“搜索 -> 下载 PDF -> 解析 -> 切片喂入 LLM”。这种流程在处理长篇学术论文时
  极其低效：重复解析产生大量计算开销；PDF 布局复杂导致提取文本质量差；全文本注入导致严重的 Token 浪费。
- **核心动机**：作者认为应将论文视为“可调用对象 (Tool-callable Objects)”。通过预先处理好的结构化视图，智能体可以
  先读摘要（低成本），再决定读哪个章节（中成本），最后才获取原文核实细节（高成本）。
- **设计目标**：实现“协议可重用 (Protocol Reusable)”和“成本可控 (Cost Controllable)”。

### 1.2 关键技术 (Key Technologies)
- **三层架构拆解**：
    - **Data Layer (数据层)**：负责大规模语料的标准化。利用 MinerU 处理 PDF 布局恢复，将非结构化文本转为 Canonical JSON。
    - **Service Layer (服务层)**：通过 REST API 暴露能力。核心创新是“渐进式访问视图”，包括 Overview（元数据）、
      Section（特定章节文本）和 Evidence（全文本）。
    - **Application Layer (应用层)**：封装了 Python SDK、MCP 连接器和内置智能体，支持复杂的“Deep Research”逻辑。

## 2. 数据层：语料库级解析与信号丰富化 (Data Layer)

### 2.1 关键技术 (Key Technologies)
- **解析管线 (Processing Pipeline)**：
    - **MinerU 布局恢复**：针对 ArXiv PDF，使用专用模型识别标题、段落、图表，保证 Markdown 转换的语义准确。
    - **结构化映射**：将 Markdown 进一步解析为带有层级结构的 section map，支持按 index 直接访问特定章节。
- **信号丰富化 (Enrichment)**：
    - **语义信号**：使用 Qwen3-4B 提取论文的一句话 TL;DR 和 5 个核心关键词。
    - **外部链接提取**：通过正则和 LLM 验证，从全文中提取 GitHub 仓库链接，并过滤掉通用的非代码库链接。
    - **社交热度**：对接 X.com API，统计论文在社交媒体上的浏览量、点赞数和转发数，作为排序权重。

### 2.2 实现细节 (Implementation Details)
- **基础设施**：PDF 转 Markdown 阶段在 8 个节点的 8xH100 集群上运行，全量 ArXiv 约需 72 小时。
- **数据库选型**：元数据存储在 PostgreSQL，全文和嵌入向量存储在 Elasticsearch（结合 BGE-m3 模型）。

## 3. 服务层：统一协议与混合检索 (Service Layer)

### 3.1 关键技术 (Key Technologies)
- **混合检索 (Hybrid Retrieval)**：结合了 BM25（关键词匹配）和基于 BGE-m3 的向量搜索，支持对标题+摘要+章节
  TL;DR 的联合索引，从而在不读取全文的情况下实现高精度的初始定位。
- **渐进式读取协议**：定义了不同粒度的 API 返回值。
    - `/arxiv?type=head`：返回目录和预算提示（Token 数预估）。
    - `/arxiv?type=section`：仅返回请求的特定章节内容，极大节省了 Context Window 占用。

### 3.2 权衡与对比 (Trade-offs & Comparison)
- **成本权衡**：传统的 RAG 系统需要对全文进行 chunking，而 DeepXiv 选择对“章节摘要”进行索引。这种方式虽损失了
  极细粒度的语义匹配，但大幅提升了系统在大规模语料下的检索速度和准确率，且能保持文档结构的完整性。

## 4. 实验评估：智能体效率验证 (Evaluation)

### 4.1 设计思想 (Design Philosophy)
- **任务设置**：设置了两个挑战性任务。
    - **Agentic Search**：50 个带约束的查询（如“过去一月内某领域表现最好的论文”），考察智能体的发现能力。
    - **Deep Research**：对 47 个复杂 QA 进行证据提取，考察智能体的分析深度。

### 4.2 实验结果 (Results)
- **Token 节省**：在 Deep Research 任务中，相比传统的“Search & Read (全文注入)”流程，DeepXiv 模式减少了约
  50% 的 Token 开销，且由于减少了无关背景干扰，答案的准确性反而有所提升。
- **延迟表现**：本地热缓存下的响应时间在 100ms 级别，即使是冷启动加载，首屏元数据返回也在 300ms 以内，满足
  交互式智能体实时调用的需求。

---
[解析完成：DeepXiv-SDK 通过将论文转化为结构化、可调用的数据服务，为科学智能体的规模化应用打下了坚实基础。]
