---
title: 高级rag
subtitle:
date: 2026-05-30T13:15:24+08:00
slug: 40be33d
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  -
categories:
  - ai
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

朴素 RAG（索引→检索→生成）的核心问题是"盲目检索"——无论问题是否需要外部知识都检索，无论检索结果质量如何都喂给生成器，无论回答是否可靠都输出。高级 RAG 模式从三个方向突破这一限制：让系统"会判断"（Self-RAG、CRAG、Adaptive RAG）、让系统"会行动"（Agentic RAG）、让知识"有结构"（Graph RAG），而 Modular RAG 提供了统一这些模式的理论框架和工程实现基础。

这些模式并非互斥的竞争关系，而是从不同维度补强 RAG 流水线，实践中组合使用是常态。

### Self-RAG（自反思 RAG）

> 论文：Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection (Asai et al., ICLR 2024 Oral, arXiv:2310.11511)

Self-RAG 的核心创新是在 LLM 的输出中**嵌入四类反思 token**，让模型自主决策检索行为并评估生成质量：

| Token 类型 | 输入 | 输出 | 作用 |
|-----------|------|------|------|
| Retrieve | 问题 x | `{yes, no}` | 模型自主决定是否需要检索 |
| IsRel | x, 文档 d | `{relevant, irrelevant}` | 检索文档是否与问题相关 |
| IsSup | x, d, 回答 y | `{fully/partially/no support}` | 回答是否被文档支持（忠实度） |
| IsUse | x, y | `{5,4,3,2,1}` | 回答对用户的有用程度 |

推理流程：对每个生成步骤，模型先预测 `[Retrieve=Yes/No]`。若 Yes，检索 Top-K 文档，逐个评估 IsRel 过滤无关文档，基于通过筛选的文档生成回答，再评估 IsSup（忠实度）和 IsUse（效用）。段级评分公式对多个候选回答打分选最优：

```
score = p(y|x,d) + w_relevance × s(IsRel) + w_support × s(IsSup) + w_utility × s(IsUse)
```

默认权重：IsRel=1.0, IsSup=1.0, IsUse=0.5，推理时可调无需重训。

训练方法分两阶段：先用 GPT-4 生成反思 token 标注数据（每类 4k-20k 样本），在 Llama2-7B 上微调训练 Critic 模型（与 GPT-4 一致性 >90%）；再用 Critic + Retriever 为 150k 指令数据插入反思 token，在 Llama2-7B/13B 上微调 Generator。

关键实验数据：Self-RAG-7B 在 PopQA 上 54.9%（ChatGPT 仅 29.3%），在 PubHealth 上 72.4%。移除 Critic 模块后 ASQA 的 em 下降 14 个点，测试时不检索 PopQA 下降 20.8 个点。

设计哲学是**模型中心**——将反思能力内置到模型本身，从生成端改进。

局限：需要专门训练注入反思 token，难以应用于闭源 API；树状解码多候选增加推理延迟；迁移性差。

### Corrective RAG（CRAG，纠错 RAG）

> 论文：Corrective Retrieval Augmented Generation (Yan et al., arXiv:2401.15884)

CRAG 的核心问题是"What if the retrieval goes wrong?"——传统 RAG 盲目将检索结果喂给生成器，不评估检索质量。

核心机制是一个轻量级**检索评估器**（基于 T5-large，仅 0.77B 参数），为每个检索文档输出 [-1, 1] 的相关性分数，准确率 84.3%（显著优于 ChatGPT 的 58-64%）。通过上下阈值将评估结果分为三路：

| 路径 | 触发条件 | 操作 |
|------|---------|------|
| Correct | 至少一个文档 > 上阈值 | 保留文档 + 知识精炼 |
| Ambiguous | 介于上下阈值之间 | 知识精炼 + 网络搜索 |
| Incorrect | 所有文档 < 下阈值 | 丢弃全部 + 网络搜索 |

```
用户查询 → 检索 → 检索评估器评分
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
    Correct        Ambiguous     Incorrect
  知识精炼       精炼+搜索      丢弃+搜索
        │             │             │
        └─────────────┼─────────────┘
                      ▼
                   生成回答
```

**知识精炼**（Decompose-then-Recompose）：将文档按句子分割为知识片段，用评估器计算每个片段的相关性分数（阈值 -0.5），过滤低分片段后按原始顺序重组。

**网络搜索回退**：用 LLM 将问题重写为关键词查询 → Google Search API 获取 Top-5 URL → 提取内容 → 复用评估器精炼。关键发现：条件触发机制显著优于始终补充网络搜索（PopQA: 52.2→59.8）。

实验数据：CRAG 相比朴素 RAG 提升幅度 4.4%-36.6%，计算开销极小（TFLOPs 从 26.5 增至 27.2）。Self-CRAG（组合 Self-RAG + CRAG）在 Biography 上达到 86.2%。

设计哲学是**系统中心**——在 RAG 流程中引入外部纠错机制，从检索端改进。核心优势：**即插即用**，无需微调基座模型，可与任意 LLM 搭配。

局限：仅关注检索质量未改进生成端；依赖评估器判断准确性；微调评估器不可避免。

### Adaptive RAG（自适应 RAG）

> 论文：Adaptive-RAG: Learning to Adapt Retrieval-Augmented LLMs through Question Complexity Classification (Jeong et al., NAACL 2024)

Adaptive RAG 的核心洞察是并非所有问题都需要检索，也并非所有问题检索一次就够了。它引入一个轻量级**查询复杂度分类器**（基于 T5-large），在检索前预判问题复杂度，路由到不同策略：

| 级别 | 策略 | 适用场景 |
|------|------|---------|
| 简单(A) | 无检索，直接用 LLM 参数知识回答 | 常识问题（"法国首都是哪"） |
| 中等(B) | 单次检索 + RAG 生成 | 需要外部知识但逻辑简单 |
| 复杂(C) | 多步迭代检索 + 推理循环（IRCOT 风格） | 多跳推理，需综合多源信息 |

```
用户查询 → [复杂度分类器] → 复杂度判断
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
  简单(A)     中等(B)     复杂(C)
    │           │           │
 无检索       单次检索     多步迭代检索
 直接LLM      + RAG生成    + 推理循环
```

分类器训练无需人工标注——通过"首次成功策略"自动生成标签：让 LLM 分别用三种策略回答，哪种首次答对就标记为对应级别。结合数据集归纳偏置（如 SQuAD 多为单跳 → A/B，MuSiQue 多为多跳 → C）辅助标注。

Adaptive RAG 可看作**更上层的路由框架**：A 层借鉴 Self-RAG 的"按需检索"思想，B 层类似 CRAG 的单次检索+评估，C 层是多步 RAG 的迭代推理。与 Self-RAG/CRAG 的关键区别在于**决策时机**——Adaptive RAG 在检索前预判，Self-RAG 在生成中逐步决策，CRAG 在检索后评估。

实验表明 Adaptive RAG 在效率-性能权衡上处于帕累托最优：在多跳任务上与 Self-RAG/CRAG 持平或更优，在简单问题上通过跳过检索减少噪声和开销。

### Agentic RAG（基于 Agent 的 RAG）

Agentic RAG 是从固定流水线到自主 Agent 的范式转变。传统 RAG 是"检索→生成"的线性管道，Agentic RAG 让 LLM 作为 Agent 自主决定"用什么工具、检索什么、检索几次、结果够不够"。

核心架构基于 **ReAct 模式**（Reasoning + Acting）：Agent 在每一步先推理（Thought）→ 选择并执行动作（Action，如搜索、查数据库、计算）→ 观察结果（Observation）→ 决定是否继续。循环直到 Agent 判断信息足够，生成最终回答。

```
用户查询 → Agent
            │
            ├─ Thought: "需要查找 X 的相关信息"
            ├─ Action: search("X 技术文档")
            ├─ Observation: 找到 3 篇相关文档
            ├─ Thought: "还需要 Y 的数据来验证"
            ├─ Action: query_database("SELECT ...")
            ├─ Observation: 查询结果 ...
            ├─ Thought: "信息已足够"
            └─ 最终回答
```

多 Agent 协作 RAG 将系统拆分为多个专职 Agent：路由 Agent 判断查询意图并分发到合适的知识源；检索 Agent 负责具体的检索操作（向量搜索、关键词搜索、图遍历等）；验证 Agent 检查检索结果和生成回答的质量；还有专门的工具 Agent（计算器、代码执行器、API 调用器等）。各 Agent 通过消息传递协作，比单 Agent 更灵活但协调复杂度更高。

实现框架方面，**LangGraph** 采用状态机图结构实现 Agent 工作流，支持条件边和循环，是当前最主流的实现方式。**LlamaIndex** 提供模块化的 Agent RAG 抽象。**CrewAI / AutoGen** 适合多 Agent 协作场景。

适用场景判断：简单事实问答不需要 Agent，线性流水线足够；需要多数据源、多轮检索、工具调用的复杂场景才需要 Agentic RAG。代价是延迟更高、成本更大、可控性更难保证。

### Graph RAG（知识图谱增强 RAG）

> 微软 GraphRAG 论文：From Local to Global: A Graph RAG Approach to Query-Focused Summarization (2024, arXiv:2404.16130)

Graph RAG 的核心区别在于用**结构化知识图谱**替代扁平的文本块向量索引。微软 GraphRAG 是最有代表性的实现。

索引阶段分五步：文本分块（默认 600 token）→ LLM 实体/关系抽取（含自反思补充提取：先提取一次，再提示"很多实体被遗漏了"迭代补充）→ 构建知识图谱（实体=节点，关系=边）→ Leiden 算法层次化社区检测 → 为每个社区生成摘要报告。

```
源文档 → 文本分块 → 实体/关系抽取 → 知识图谱 → 社区检测(Leiden) → 社区摘要
                                                    │
                                              层次化社区结构
                                              C0(根级) → C1 → C2 → C3(叶级)
```

查询阶段提供两种模式：**全局搜索**采用 Map-Reduce 范式——社区摘要分块 → 并行生成部分回答（附 0-100 有用性评分）→ 过滤 0 分答案 → 按评分降序聚合生成最终回答。适用于"数据集的主要主题是什么"这类全局性问题。**局部搜索**从查询中识别种子实体 → 遍历关联关系和文本块 → 生成局部性回答。适用于具体实体查询。

论文在两个约 1M token 数据集上的关键数据：全局方法在全面性上胜率 72-83%（p<.001），在多样性上胜率 62-82%。根级社区查询仅需全文本摘要 2.3-2.6% 的 token 消耗即保持 72% 的全面性胜率。

变体演进：**LightRAG**（港大）引入双层级检索和增量更新算法，索引成本约为微软 GraphRAG 的 1/1000，解决了全量重建的核心痛点。**LazyGraphRAG**（微软自身演进）将数据索引成本降至 GraphRAG 的 0.1%。**nano-graphrag** 用约 800 行代码实现核心功能，适合研究和快速集成。

工程考量：建图阶段 LLM 调用量巨大（Podcast 数据集索引耗时 281 分钟），适合知识密集且需要复杂推理的场景（多跳推理、全局摘要、关系查询），不适合频繁更新的数据和简单的精确事实检索。

### Modular RAG（模块化 RAG）

> 论文：Modular RAG: Transforming RAG Systems into LEGO-like Reconfigurable Frameworks (Gao et al., 2024, arXiv:2407.21059)

Modular RAG 不是一种具体的 RAG 方法，而是一个**统一的理论框架**，将 RAG 系统视为可插拔模块的组合。它定义了 RAG 的三阶段演进：Naive RAG（线性管道）→ Advanced RAG（增强的线性管道）→ Modular RAG（可组合的拓扑结构），后者的核心论点是 Advanced RAG 是 Modular RAG 的特例，Naive RAG 是 Advanced RAG 的特例。

框架定义了六大核心模块：**索引模块**（块优化、结构组织）、**预检索模块**（查询扩展/变换/构造）、**检索模块**（检索器选择/微调）、**后检索模块**（重排序/压缩/选择）、**生成模块**（微调/验证）、**编排模块**（路由/调度/融合）——其中编排模块是 Modular RAG 区别于 Advanced RAG 的标志。

| 模块 | 核心子模块 | 代表算子 |
|------|-----------|---------|
| 索引 | 块优化 | Small-to-Big、元数据附加 |
| 索引 | 结构组织 | 层次索引、知识图谱索引 |
| 预检索 | 查询扩展/变换 | Multi-Query、Rewrite、HyDE、Step-back |
| 检索 | 检索器选择 | BM25、Dense、Hybrid |
| 后检索 | 重排序/压缩 | Cross-Encoder、LLMLingua |
| 生成 | 微调/验证 | SFT、RL、知识库验证 |
| 编排 | 路由/调度/融合 | 语义路由、LLM 判断、RRF |

模块通过四种 Flow 模式组合：**线性模式**（Naive/Advanced RAG）、**条件模式**（按查询性质路由）、**分支模式**（多路并行检索/生成）、**循环模式**（迭代/递归/自适应检索）。Self-RAG 是"循环模式 + 自适应调度"的特定组合，CRAG 是"条件模式 + 检索后路由"的特定组合，FLARE 是"循环模式 + 置信度调度"的特定组合。

工程实现：**LangGraph** 用状态机图结构实现 Modular RAG 的编排逻辑，支持条件边和循环。**LlamaIndex** 通过 `QueryEngine` 接口组合不同模块。这些框架的本质都是提供"模块 + 编排"的基础设施，让开发者像搭乐高一样组合出所需的 RAG 变体。

### 模式关系总览

这六种高级 RAG 模式解决的是不同层面的问题：

Self-RAG 和 CRAG 是**互补的**——Self-RAG 从生成端让模型自己判断检索需求和回答质量，CRAG 从检索端评估检索结果并在失败时纠正。两者可以组合使用（Self-CRAG），Self-RAG 的检索决策 + CRAG 的检索纠错 + Self-RAG 的生成反思。

Adaptive RAG 是一个**上层路由框架**，在检索前做复杂度分类，其三个分支内部可以分别使用 Self-RAG（简单问题，按需检索）、CRAG 风格（中等问题，单次检索+评估）、多步 RAG（复杂问题，迭代推理）。

Agentic RAG 是**范式的跃迁**——从预定义流水线到 Agent 自主决策，理论上可以包含上述所有模式作为 Agent 的工具或策略。

Graph RAG 解决的是**知识表示**层面的问题——用结构化图谱替代扁平文本块，可以与上述任何模式组合（如 Agentic Graph RAG）。

Modular RAG 是**统一框架**——上述所有模式都是 Modular RAG 的特定模块组合，它提供了理解和实现这些模式的理论基础。

| 模式 | 改进维度 | 决策时机 | 核心机制 |
|------|---------|---------|---------|
| Self-RAG | 生成端反思 | 生成中逐步决策 | 反思 token（Retrieve/IsRel/IsSup/IsUse） |
| CRAG | 检索端纠错 | 检索后评估 | T5 评估器三路分支 + 网络搜索回退 |
| Adaptive RAG | 检索前路由 | 检索前预判 | 复杂度分类器三级策略 |
| Agentic RAG | 全流程自主决策 | 全程 | ReAct 循环 + 多工具/多 Agent |
| Graph RAG | 知识表示结构化 | 索引阶段 | 知识图谱 + 社区检测 + Map-Reduce |
| Modular RAG | 统一框架 | — | 六大模块 + 四种 Flow 模式 |

### 实践选型参考

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 快速落地 / 已有 RAG 系统 | CRAG | 即插即用，无需微调模型 |
| 高精度要求（医疗、法律） | Self-RAG | 多层自校验保证事实性 |
| 查询复杂度差异大 | Adaptive RAG | 按复杂度路由，效率最优 |
| 多数据源 + 工具调用 | Agentic RAG | Agent 自主决策，灵活组合 |
| 多跳推理 + 全局摘要 | Graph RAG | 结构化知识表示，关系推理 |
| 研究和实验 | Modular RAG | 模块化组合，快速迭代 |

> 参考：[Self-RAG (arXiv)](https://arxiv.org/abs/2310.11511), [CRAG (arXiv)](https://arxiv.org/abs/2401.15884), [Adaptive-RAG (arXiv)](https://arxiv.org/abs/2403.14403), [GraphRAG (arXiv)](https://arxiv.org/abs/2404.16130), [LightRAG (arXiv)](https://arxiv.org/abs/2410.05779), [Modular RAG (arXiv)](https://arxiv.org/abs/2407.21059), [LangGraph CRAG 教程](https://github.langchain.ac.cn/langgraph/tutorials/rag/langgraph_crag/), [LangChain - Agentic RAG](https://blog.langchain.dev/agentic-rag-with-langgraph/), [Microsoft GraphRAG](https://microsoft.github.io/graphrag/), [nano-graphrag](https://github.com/gusye1234/nano-graphrag)
