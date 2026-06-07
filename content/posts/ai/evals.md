---
title: Evals
subtitle:
date: 2026-05-30T11:38:42+08:00
slug: e6685fc
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
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

> 本文是对 [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) 的读书笔记，作者 Hamel Husain 是独立顾问，曾领导创建 GitHub CodeSearchNet（GitHub Copilot 的前身）。文中还整合了 Nuvi (Relari) 团队关于评估数据集策略和进阶实践的内容。

不成功的 AI 产品几乎都有一个共同的根源：**未能建立健壮的评估系统**。

AI 产品的成功取决于迭代速度，而迭代需要三个环节：

1. 评估质量（测试）
2. 调试问题（日志记录与数据检查）
3. 更改行为（Prompt 工程、微调、编写代码）

很多人只关注第 3 点，忽略前两点，导致产品永远停留在 demo 阶段。

SaaS 产品 Rechat 的 AI 助手 Lucy 就是一个典型：随着功能增加，性能遇到瓶颈——修一个问题导致另一个问题出现（打地鼠），无法系统评估 AI 的有效性，Prompt 变得冗长且难以维护。团队最终通过建立以评估为中心的系统化方法打破瓶颈。

RAG 部分见 [RAG]({{< relref "posts/ai/rag" >}})

## 评估数据集策略

> 来源：[How important is a Golden Dataset for LLM evaluation?](https://blog.nuvi.dev/how-important-is-a-golden-dataset-for-llm-pipeline-evaluation-4ef6deb14dc5) — Yi Zhang & Pasquale Antonante, Nuvi (formerly Relari)

评估 LLM 管道时，用什么数据集作为基准是一个前提性问题。三种策略各有取舍：

**1. 无参考评估（Reference-free）**

不需要任何基准答案，直接对输出做判断。上手极快（不到 5 分钟），适合生产环境的趋势监控（如事实准确性指标）。但洞察片面（无法测量 Context Recall 等），且在不同数据集上无法客观对比结果，优化时如同"射击移动目标"。

**2. 合成数据集（Synthetic Dataset）**

用 LLM 生成测试数据作为参考基准的"捷径"。拥有固定数据集后可以真实对比不同设计选择的影响（分块策略、检索算法、提示词等），上手相对容易。但 LLM 的概率特性会导致生成的查询质量参差不齐、可能不代表真实用户行为，仍需人工审核，且目前没有标准化的生成方法。

**3. 黄金数据集（Golden Dataset）**

最全面可靠的方式——离线评估表现应与生产环境用户满意度高度相关。能提供最一致、最完整的评估洞察，但构建成本高，需要工程师、产品经理、领域专家协作，且必须持续维护、不断进化以覆盖新场景。

**建议路径**：从"无参考评估 + 合成数据集"组合起步，快速获得管道的初步视图；同时逐步积累高质量的、经人工验证的黄金数据集，这是不可替代的。

### 合成数据生成实践

> 来源：[Generate Synthetic Data to test LLM Applications](https://blog.nuvi.dev/generate-synthetic-data-to-test-llm-applications-4bffeb51b80e) — Yi Zhang & Pasquale Antonante, Nuvi (formerly Relari)

上文提到合成数据集是快速建立评估基准的"捷径"，这篇文章进一步回答了**如何实际生成高质量的合成数据**。

合成数据不是完全取代人工标注，而是其高效补充——如果团队能手动标注 100 个样本，通过合成数据管道可以轻松扩大 10 倍以上，同时覆盖人类难以想到的边缘情况。

整个工作流分为三步：

**1. 生成应用特定的合成数据**，需要三个输入维度：

- 应用逻辑（如 RAG 的检索+生成流程、Agent 工具调用等），确保生成的输入输出格式符合测试需求
- 环境数据（文档、语料库、向量数据库等 LLM 运行的真实上下文）
- 种子示例数据（少量真实历史数据或人工标注样本，指导生成器创建更逼真的数据）

以维基百科 RAG 为例：应用逻辑是"用户问题 → 检索维基百科 → LLM 生成回答"，环境数据是维基百科语料库，种子数据是示例问答。生成器产出合成问题（Input）、合成的来源 URL 与上下文（Intermediate Steps）、参考答案（Output）。

**2. 执行测试**：将合成输入喂给应用，对比应用的中间步骤和最终输出与合成数据中的预期结果。好的生成器应结合确定性规则、传统 ML 模型和 LLM 来保证数据保真度和多样性。

**3. 持续改进**：生产环境存在数据漂移，需要将真实数据分布变化反馈给合成管道以保持测试有效性。

对于 LLM Agent，合成数据可以创建两类测试。功能测试以编程 Agent 为例：输入是修改指令和目标代码库，输出是预期的代码 diff。端到端测试以 SEC 财务分析 Agent 为例：输入是财务问题，中间输出包括问题类型、SEC 文件链接、使用的数据和计算过程，最终输出是格式化答案。

最大的挑战是 **Sim-to-Real Gap（虚实鸿沟）**——合成数据可能不够真实。应对方法包括：初期必须人工抽查质量、迭代反馈（将高质量样本作为新种子）、系统化测量合成数据与真实数据的分布差异。

## 评估的三个层次

### Level 1：单元测试

类似传统开发中的 `pytest`，对 LLM 输出做断言。必须运行快、成本低，以便每次代码变更时都能运行。

步骤：

1. **编写范围测试**：将 LLM 功能拆解为具体的场景，为每个场景编写断言（如验证结果数量、正则检查不泄露 UUID 等）。不需要 100% 通过率，容忍度是商业决策。
2. **创建测试用例**：利用 LLM 合成生成大量测试输入数据，好的测试应有一定挑战性，能暴露失败模式。
3. **运行并持续跟踪**：利用 CI（如 GitHub Actions）运行测试，用分析工具构建仪表盘，长期跟踪测试结果和错误率。

### Level 2：人类与模型评估

Level 1 稳定后，进行无法用简单断言完成的验证。

**记录追踪**：记录用户与 LLM 交互的完整会话流，推荐使用 LangSmith 等工具查看、搜索和过滤日志。

**亲自查看数据**：

- 消除查看数据的所有阻力：根据业务定制数据查看和标注工具（可用 Gradio、Streamlit 等快速构建），将所有上下文集中在一个屏幕上
- 数据标注：建议初期使用简单的二元评分（好/坏），比复杂打分更容易管理
- 看多少数据：刚开始尽可能多看（至少看完所有测试用例和真实用户追踪）。永远不要停止看数据

**使用 LLM 进行自动化评估**：

- 不要完全依赖自动化评估，必须定期让人类抽样检查
- 对齐人类与模型：通过对比"人类评判"与"模型评判"，调整评估模型的 Prompt 或微调，让模型越来越接近人类判断
- 指标建议：当数据集不平衡时，不要用"一致性"，应使用精确率和召回率
- 模型选择：评估模型应尽量用能负担得起的最强模型，因为批判需要高级推理能力

### Level 3：A/B 测试

产品足够成熟时，通过 A/B 测试验证是否真正推动了用户期望的行为或结果。

## 进阶实践

### LLM-as-a-Judge 深入实践

> 本节是对 [Your AI Product Needs Evals: LLM-as-a-Judge](https://hamel.dev/blog/posts/llm-judge/) 的补充，同一作者基于 30+ 公司实战经验的深入展开。

多数团队在 LLM 评估上犯同样的错误：指标过多、使用未校准的 1-5 分评分、忽视领域专家、追踪的指标不反映真实需求。作者提出 **Critique Shadowing（批评影子法）**，一套以二元判断和详细批评为核心的迭代流程。

**第一步：找到主要领域专家**

组织中真正懂业务的一两人，而非开发者自己。他们设定标准、捕捉未明说的期望、确保评估一致性。

**第二步：从三个维度构建数据集**

特征（AI 能做什么）、场景（AI 需要处理什么情况）、用户画像（典型用户是谁）。可用 LLM 生成合成数据，只要输入多样化效果就很好。

**第三步：让专家做二元判断并写详细批评**

拒绝 1-5 分评分，只用 Pass/Fail——"如果有人说要考核 8 个维度的 1-5 分，说明他不知道自己在找什么"。批评必须足够详细，详细到可以直接用作 few-shot 提示词。

**第四步：先修复错误**

在构建 LLM Judge 之前，先修复审查中发现的明显问题，然后循环回第三步直到系统稳定。

**第五步：迭代构建 LLM Judge**

将专家的判断和批评作为 few-shot 示例构建提示词，反复调整直到与专家一致性 >90%。常见错误包括：提示词中没有包含专家批评、批评过于简短、没有提供系统外部上下文、示例不够多样化。

**第六步：错误分析**

用构建好的 LLM Judge 批量评估，手工分类错误（如 40% 缺少引导、30% 权限问题），定位具体弱点后修复，再回到第三步循环。

**第七步：建立专门的评判者**

针对已知的系统弱点（如总是引用来源出错），建立专门的评判者或硬编码断言。

关键洞见：评估的真正价值在于**强迫你和专家仔细看数据**，LLM Judge 只是达成这个目的的手段。不建议微调 Judge 模型，精力应花在微调业务主模型上或编写更好的断言和提示词。

### 案例实证：无参考 vs 基于参考评估的差异

> 来源：[Case Study: Reference-free vs Reference-based evaluation of RAG pipeline](https://blog.nuvi.dev/case-study-reference-free-vs-reference-based-evaluation-of-rag-pipeline-9a49ef49866c) — Yi Zhang & Pasquale Antonante, Nuvi (formerly Relari)

文章用一个企业 Q&A 案例直观展示了两种评估方法的差距。场景是"温莎公司碳排放可持续发展举措"，检索器只召回了部分上下文（车队改电动 + 海上风电场），遗漏了另一条关键信息（收购 Treeplanter 公司及种树计划）。

**无参考评估阶段**：Answer Relevance、Context Precision、Faithfulness 等指标全部"看起来很好"——答案相关、无幻觉、上下文精准。但实际答案是不完整且不正确的，无参考指标无法发现这一问题。

**基于参考评估阶段**：引入 Ground Truth 后，结果暴露了真正的问题：Context Recall 仅 50%（只检索到一半信息）、Answer Correctness 仅 0.75、Deberta Entailment 极低（0.013）、Style Consistency 仅 0.333。

核心结论与上面的策略分析一致：无参考指标提供重要但不完整的洞察，某些关键缺陷（如检索遗漏导致答案不完整）只能通过基于参考的指标揭示。最佳实践是在离线评估中结合两者。

### 复杂管道的细粒度评估

> 来源：[How to evaluate complex GenAI Apps: a granular approach](https://blog.nuvi.dev/how-to-evaluate-complex-genai-apps-a-granular-approach-0ab929d5b3e2) — Yi Zhang, Nuvi (formerly Relari)

前面讨论的评估方法仍隐含一个假设：管道只有"检索"和"生成"两步。但生产环境中的 GenAI 管道往往包含 10+ 个模块（分类器、多种检索器、重排序器、Agent 等），仅评估最终输出无法定位瓶颈。

以一个典型复杂 RAG 管道为例：查询分类器 → 向量检索器 + BM25 检索器 + HyDE 生成/检索器 → Cohere 重排序器 → LLM 生成器。该管道的最终答案正确率约 70%，但通过细粒度评估回溯发现，问题出在某个检索器的召回率不足，而非生成器的问题。

细粒度评估需要三个能力：**记录中间步骤的输入输出**、**模块级别的评估指标**（如可读性 FleschKincaid、DeBERTa 答案评分、LLM 忠实度评估等）、**定制黄金数据集**。开源工具 `continuous-eval` 提供了相应的框架支持：用 `Module` 和 `Pipeline` 类定义管道拓扑，为每个模块绑定独立的指标和阈值测试，运行后可逐模块分析结果。

核心原则：评估复杂 GenAI 应用不能只看端到端结果，必须对每个模块单独评估，才能精准定位问题并持续优化。

## 评估系统带来的收益

### 微调的数据合成与策展

微调最适合让模型学习语法、风格和规则（RAG 更适合提供最新事实）。评估系统本身就是强大的数据生成引擎：用 LLM 生成海量合成数据，然后用 Level 1 和 Level 2 测试过滤不合格数据，甚至直接利用人工评估工具整理高质量的微调数据集。

### 快速调试

收到错误报告时，评估系统提供：可搜索的追踪数据库、能标记错误的断言机制、快速定位问题（是 RAG 召回错误？代码 Bug？还是模型不行？）的日志工具。评估和调试的基础设施高度重合。

## 核心建议

- 消除查看数据的所有阻力
- 保持简单，不要急于购买花哨的 LLM 评估工具，先用好现有工具
- 如果你没有看大量的数据，那你的做法就是错的
- 不要依赖通用的评估框架，必须为具体问题建立专属的评估系统
- 编写大量的测试，并随产品演进不断更新
- 用魔法打败魔法：充分利用 LLM 辅助建立评估系统（生成测试用例、写断言、生成合成数据、做数据评判等）
- 复用评估基础设施，将其用于调试和微调

## 参考资源

- [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) — Hamel Husain
- [Your AI Product Needs Evals: LLM-as-a-Judge](https://hamel.dev/blog/posts/llm-judge/) — Hamel Husain
- [How important is a Golden Dataset for LLM evaluation?](https://blog.nuvi.dev/how-important-is-a-golden-dataset-for-llm-pipeline-evaluation-4ef6deb14dc5) — Yi Zhang & Pasquale Antonante, Nuvi
- [Generate Synthetic Data to test LLM Applications](https://blog.nuvi.dev/generate-synthetic-data-to-test-llm-applications-4bffeb51b80e) — Yi Zhang & Pasquale Antonante, Nuvi
- [Case Study: Reference-free vs Reference-based evaluation of RAG pipeline](https://blog.nuvi.dev/case-study-reference-free-vs-reference-based-evaluation-of-rag-pipeline-9a49ef49866c) — Yi Zhang & Pasquale Antonante, Nuvi
- [How to evaluate complex GenAI Apps: a granular approach](https://blog.nuvi.dev/how-to-evaluate-complex-genai-apps-a-granular-approach-0ab929d5b3e2) — Yi Zhang, Nuvi