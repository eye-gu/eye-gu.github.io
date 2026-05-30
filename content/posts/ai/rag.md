---
title: Rag
subtitle:
date: 2026-05-28T21:25:02+08:00
slug: 9509132
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

## RAG

### 查询转换策略

朴素 RAG 的典型流程是将文档分块、嵌入，然后检索与用户问题语义相似度高的文档块。但这存在几个问题：文档块可能包含无关内容、用户问题的措辞可能不适合检索、可能需要从用户问题生成结构化查询。

查询转换的核心思想是：**在将用户问题传递给嵌入模型之前，利用 LLM 对其进行变换操作**，以改善检索效果。以下几种方法的共同点是使用 LLM 生成新的（或多个）查询，主要区别在于生成时使用的提示词不同。

> 参考：[LangChain - Query Transformations](https://www.langchain.com/blog/query-transformations)

#### Rewrite-Retrieve-Read（重写-检索-阅读）

利用 LLM **重写用户查询**，而不是直接使用原始查询进行检索。流程变为：重写 → 检索 → 阅读。

原始查询并不总是最适合检索的，尤其在真实场景中，用户的提问往往含糊或措辞不当。通过 LLM 重写为更清晰、更适合检索的版本后再执行检索。

```
原始问题: "那个怎么用"
重写后: "LangChain 中 Multi Vector Retriever 如何使用"
```

#### Step Back Prompting（后退提示）

利用 LLM 生成一个更抽象、更高层次的"后退"问题，然后**同时使用后退问题和原始问题进行检索**，将两个检索结果都用于支撑 LLM 的响应。

后退问题可以帮助获取更广泛的背景知识，弥补原始问题过于具体而检索不到相关文档的不足。

```
原始问题: "Python 3.11 有哪些新特性"
后退问题: "Python 版本更新的历史和新特性概述"
→ 同时用两个问题检索，合并结果
```

#### Follow Up Questions（后续问题处理/上下文压缩）

在对话链中处理后续问题时，存在三种选项：

1. **仅嵌入后续问题** — 会丢失上下文。例如先问"意大利有什么好玩的"，再问"那里有什么食物"，仅嵌入后者将丢失"那里"指代意大利的上下文
2. **嵌入整个对话** — 后续问题若与之前无关，会引入不相关结果
3. **LLM 查询转换（推荐）** — 将整个对话（含后续问题）传给 LLM，生成一个独立的搜索查询

```
对话历史:
  用户: "意大利有什么好玩的"
  助手: "罗马斗兽场、威尼斯..."
  用户: "那里有什么食物"
LLM 生成独立查询: "意大利有哪些特色美食"
```

#### Multi Query Retrieval（多查询检索）

利用 LLM **生成多个搜索查询**，并行执行检索，然后将所有检索结果合并传入。

适用于复合问题，一个问题可能依赖多个子问题：

```
原始问题: "红袜队和爱国者队谁最近赢得了冠军"
生成子查询:
  - "红袜队上一次赢得冠军是什么时候"
  - "爱国者队上一次赢得冠军是什么时候"
→ 并行检索，合并结果
```

#### RAG-Fusion（RAG 融合）

建立在多查询检索思想之上，但**不再将所有文档直接传入**，而是使用**互惠排名融合（Reciprocal Rank Fusion, RRF）**对多路检索结果进行重新排序，提升检索结果质量。

RRF 的核心公式：每个文档在每路查询中的排名取倒数（`1 / (k + rank)`），然后对所有查询的得分求和，得到最终排序分数。

```
原始问题: "如何优化 RAG 系统性能"
生成多个查询:
  - "RAG 检索优化方法"
  - "提升 RAG 响应质量的技巧"
  - "RAG 系统最佳实践"
→ 并行检索 → RRF 重排序 → 合并结果
```

#### 方法对比

| 方法 | 核心机制 | 适用场景 |
|------|---------|---------|
| Rewrite-Retrieve-Read | LLM 重写查询 | 用户提问含糊或措辞不当 |
| Step Back Prompting | 生成后退问题 + 双重检索 | 需要更广泛的背景知识 |
| Follow Up Questions | 对话压缩为独立查询 | 多轮对话中的上下文保持 |
| Multi Query Retrieval | 生成多个子查询并行检索 | 复合问题需拆分 |
| RAG-Fusion | 多查询 + RRF 重排序 | 追求更高检索质量 |


### RAG 路由

当系统存在多个不同的知识源（如 Python 文档库、JS 文档库、内部 Wiki）时，需要通过路由机制将查询智能分发到最相关的数据源。路由结果可能是 0 个、1 个或多个数据源，多路结果再通过融合策略合并。

#### 逻辑路由

通过 LLM 的结构化输出，将用户查询分类到预定义的数据源。LLM 分析查询意图后，从候选数据源中选择最匹配的一个或多个。

```
用户问题: "Python 的装饰器怎么用"
LLM 路由决策 → datasource: "python_docs"
→ 仅在 Python 文档库中检索

用户问题: "React 和 Vue 哪个更适合大型项目"
LLM 路由决策 → datasource: ["js_docs", "js_docs"]
→ 在两个文档库中分别检索，融合结果
```

优势是可预测、易调试；劣势是灵活性有限，需要预先定义所有候选数据源。

#### 语义路由

基于嵌入向量计算用户查询与各路由描述之间的余弦相似度，选择相似度最高的路由。不依赖 LLM 调用，仅使用嵌入模型，速度更快、成本更低。

```
路由描述:
  - "与物理学、力学、运动相关的问题"
  - "与数学、计算、公式相关的问题"

用户问题: "什么是牛顿第二定律"
→ 计算查询嵌入与各路由描述的余弦相似度
→ 匹配到物理学路由 → 在物理知识库中检索
```

#### 元数据路由

不依赖查询内容本身，而是根据请求附带的上下文元数据（用户角色、商品属性、地域、时间等）决定路由目标。这些信息通常来自用户画像或业务系统，而非 LLM 推理。

```
用户 A (role: "客服") 询问: "退货流程是什么"
→ 路由到客服知识库

用户 B (role: "买家") 询问: "退货流程是什么"
→ 路由到买家帮助文档

用户 C (product_category: "电子产品") 询问: "保修政策"
→ 路由到电子产品售后文档库

用户 D (product_category: "服装") 询问: "保修政策"
→ 路由到服装售后文档库
```

同一个问题因元数据不同被路由到不同知识源，返回与用户身份或业务上下文匹配的答案。

#### 混合路由

结合多种策略：优先使用语义路由（快速、低成本），当相似度低于阈值时回退到逻辑路由（LLM 判断更准确），同时叠加元数据路由作为硬性约束，兼顾效率与准确性。

#### 路由方式对比

| 方式 | 决策依据 | 速度 | 成本 | 可解释性 |
|------|---------|------|------|---------|
| 逻辑路由 | LLM 结构化输出 | 快 | 较高（需 LLM 调用） | 高 |
| 语义路由 | 嵌入相似度 | 很快 | 低（仅需嵌入） | 中 |
| 元数据路由 | 请求上下文（角色/属性等） | 极快 | 无 | 极高 |
| 混合路由 | 多策略组合 | 中 | 中 | 中 |

> 参考：[LangChain - 路由](https://python.langchain.ac.cn/docs/how_to/routing/), [Semantic Router](https://github.com/aurelio-labs/semantic-router)

### 切分策略

#### 元数据注入/过滤

在文档切分后，为每个分块注入元数据（如文档名称、所属分类、标签、作者、时间等），查询时可根据元数据进行过滤，缩小检索范围，提升检索精度。

```
文档: "Python 3.12 新特性.md"
切分后为每个 chunk 注入:
  metadata:
    doc_name: "Python 3.12 新特性"
    category: "python"
    tags: ["python", "release", "changelog"]

用户查询: "Python 最新版本的改进"
→ 先按 category=python 过滤，再执行语义检索
```