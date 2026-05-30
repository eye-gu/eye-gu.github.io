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

## 完整列表

[评估](https://langchain4j.cn/tutorials/testing-and-evaluation.html)

> 本文是对 [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) 的读书笔记，作者 Hamel Husain 是独立顾问，曾领导创建 GitHub CodeSearchNet（GitHub Copilot 的前身）。

## 核心观点

不成功的 AI 产品几乎都有一个共同的根源：**未能建立健壮的评估系统**。

AI 产品的成功取决于迭代速度，而迭代需要三个环节：

1. 评估质量（测试）
2. 调试问题（日志记录与数据检查）
3. 更改行为（Prompt 工程、微调、编写代码）

很多人只关注第 3 点，忽略前两点，导致产品永远停留在 demo 阶段。

## 案例：Lucy（房地产 AI 助手）

SaaS 产品 Rechat 的 AI 助手 Lucy，随着功能增加，性能遇到瓶颈：修一个问题导致另一个问题出现（打地鼠），无法系统评估 AI 的有效性，Prompt 变得冗长且难以维护。团队最终通过建立以评估为中心的系统化方法打破瓶颈。

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

#### LLM-as-a-Judge 深入实践

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

### Level 3：A/B 测试

产品足够成熟时，通过 A/B 测试验证是否真正推动了用户期望的行为或结果。

## 评估系统解锁的超能力

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