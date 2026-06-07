---
title: N-gram 索引
subtitle: 搜索引擎中的子串匹配利器
date: 2026-05-30T09:57:51+08:00
slug: 480fa41
draft: false
author:
  name:
  link:
  email:
  avatar:
description: N-gram 索引原理及其在搜索引擎中的应用，涵盖字符级/词级 N-gram、索引构建、查询匹配，以及 Skip-gram、Positional N-gram 等变体
keywords:
  - n-gram
  - 全文检索
  - 搜索引擎
  - 倒排索引
license:
weight: 0
tags:
  - 
categories:
  - index
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary: N-gram 索引原理及其在搜索引擎中的应用，涵盖字符级/词级 N-gram、索引构建、查询匹配，以及 Skip-gram、Positional N-gram 等变体
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

## 什么是 N-gram

N-gram 是从文本中提取的连续 N 个元素的子序列。这个"元素"可以是字符，也可以是词。搜索引擎利用 N-gram 将文本拆成固定长度的片段，为每个片段建立倒排索引，从而支持子串匹配、模糊搜索等能力。

以 `"hello"` 为例，它的 bigram（2-gram）拆分结果为：`he`、`el`、`ll`、`lo`。

## 为什么需要 N-gram 索引

传统的倒排索引以**完整词项**为索引单位，依赖分词器将文本切分为独立词语。这在英文等以空格自然分词的语言中工作良好，但在以下场景中存在局限：

- **中文/日文等 CJK 语言**：词与词之间没有空格分隔，分词本身就是一个难题
- **子串搜索**：用户输入 `"ell"` 想匹配 `"hello"`，词项索引无法直接支持
- **模糊搜索/拼写纠错**：需要度量两个字符串的相似度

N-gram 索引通过将文本拆成更小的粒度来解决这些问题。它不依赖分词，直接在字符层面建立索引，天然支持任意子串的检索。

## 字符级 N-gram

字符级 N-gram 以**单个字符**为基本单位，每次取连续 N 个字符构成一个 gram。

### 拆分规则

给定文本 `T`，从第 1 个字符开始，每次取连续 N 个字符，步长为 1，直到取完。

以 `"搜索引"` 的 bigram 为例：`搜索`、`索引`

以 `"搜索引擎"` 的 trigram（3-gram）为例：`搜索引`、`索引擎`

### 边界处理

实际工程中通常引入特殊边界符（如 `\0` 或 `$`），在文本前后各填充 N-1 个边界符。这样可以让文本开头的字符也参与完整的 N-gram 拆分。

以 `"hi"` 的 bigram 为例，填充后为 `$hi$`，拆分结果为：`$h`、`hi`、`i$`。

### 索引构建

构建流程：

1. 对文档集合中每篇文档的文本字段进行 N-gram 拆分
2. 将每个 gram 作为 key，文档 ID 作为 value，写入倒排索引

```
gram      → [docId1, docId2, ...]
─────────────────────────────────
"搜索引"  → [1, 5, 12]
"索引擎"  → [1, 8]
```

### 查询匹配

查询时，将用户输入的查询词同样拆分为 N-gram，然后对每个 gram 的倒排列表求交集，得到候选文档集合。

例如查询 `"搜索引"`，拆分为 `搜索引`，只有一个 gram，直接返回其倒排列表 `[1, 5, 12]`。

查询 `"搜索引擎"`，拆分为 `搜索引`、`索引擎`，两个 gram 的倒排列表求交集，得到同时包含这两个 gram 的文档。

注意：N-gram 索引的交集结果可能存在误匹配（false positive），因为 gram 的相对位置信息在简单倒排索引中丢失了。最终需要回原文验证。

## 词级 N-gram

词级 N-gram 以**词（token）**为基本单位，适用于已经过良好分词的文本。

以 `"我 喜欢 搜索引擎"` 的词级 bigram 为例：`我喜欢`、`喜欢搜索引擎`

词级 N-gram 在搜索引擎中常用于**短语查询**和**邻近度查询**的场景，用于捕获词与词之间的搭配关系和语序信息。

字符级与词级 N-gram 的核心区别：

| 维度 | 字符级 N-gram | 词级 N-gram |
|------|--------------|-------------|
| 索引粒度 | 字符 | 词 |
| 适用语言 | CJK、无空格语言 | 任何已分词文本 |
| 典型用途 | 子串匹配、模糊搜索 | 短语匹配、语言模型 |
| 索引大小 | 较大（gram 数量多） | 较小 |
| 分词依赖 | 无 | 需要分词器 |

## N 的选择

N 的取值直接影响索引的效果和开销：

- **N 太小（如 unigram）**：区分度低，倒排列表过长，查询效率差，误匹配率高
- **N 太大**：粒度太粗，漏匹配增多，且索引体积膨胀
- **实践中常用 bigram 或 trigram**：在区分度和灵活性之间取得平衡。搜索引擎中 trigram 是最常见的选择

对于 CJK 文本，trigram 通常足够；对于英文等拉丁语系，bigram 到 4-gram 都有应用。

## N-gram 变体

### Edge N-gram

Edge N-gram（边缘 N-gram）只从文本的**一端**开始生成 gram，通常用于**前缀匹配/搜索建议（autocomplete）** 场景。

以 `"hello"` 的前缀 edge n-gram 为例（从长度 1 递增到完整长度）：`h`、`he`、`hel`、`hell`、`hello`

由于只保留前缀 gram，索引体积远小于完整 N-gram 索引，非常适合搜索框补全。

### Skip-gram

Skip-gram 在标准 N-gram 的基础上允许**跳过**中间的若干个元素。核心思想是：不仅捕获相邻元素的关系，还能捕获非相邻元素之间的关联。

以 `"搜索引擎"` 的 skip-1 bigram（允许跳过 1 个字符）为例：在标准 bigram `搜索`、`索引`、`引擎` 之外，还生成 `索引`（跳过 `索`）和 `搜引`（跳过 `索`，取决于具体定义）。

Skip-gram 的价值：

- 提高召回率，容忍查询中的少量偏差
- 在 NLP 领域是 Word2Vec 的核心训练方式（Word2Vec 的 Skip-gram 模型）
- 搜索引擎中可用于模糊匹配和容错检索

### Positional N-gram

Positional N-gram 在标准 N-gram 的基础上额外记录每个 gram 在原文中的**位置信息**。

```
gram      → [(docId, position), ...]
─────────────────────────────────────
"搜索引"  → [(1, 0), (5, 3), (12, 0)]
"索引擎"  → [(1, 1), (8, 7)]
```

查询匹配时，不仅检查文档是否包含所有查询 gram，还验证 gram 之间的**相对位置**是否连续。这解决了普通 N-gram 索引的误匹配问题，无需回原文验证即可确保精确匹配。

代价是索引体积增大（需要存储位置信息），以及查询时额外的位置计算开销。

### Q-gram / Overlapping N-gram

Q-gram 是 N-gram 的另一种命名方式（Q 即为 N），多见于数据库领域。当文献中提到 q-gram 时，本质上与 N-gram 相同，只是参数符号不同。它常用于近似字符串连接（approximate string join）等场景。

## N-gram 索引 vs 其他索引方式

| 对比维度 | 普通全文索引（倒排索引） | N-gram 索引 | 前缀树（Trie） |
|---------|----------------------|------------|--------------|
| 分词依赖 | 强依赖（内建分词器） | 无 | 无 |
| LIKE '%xx%' | 全表扫描 | 走索引 | 需遍历 |
| 精确查询性能 | 快 | 较快（需去重验证） | 快 |
| 相关性排序 | 原生支持（TF-IDF/BM25） | 需额外实现 | 不支持 |
| 索引体积 | 小 | 较大 | 中等 |
| 模糊搜索 | 需额外算法 | 天然支持 | 需额外算法 |
| 典型应用 | 通用全文检索 | CJK 搜索、模糊匹配 | 自动补全、词典 |
| 代表实现 | MySQL FULLTEXT、Lucene | ES ngram tokenizer | Marisa-trie、DAWG |

## 实际应用

- **Elasticsearch / Lucene**：提供了 `ngram` tokenizer 和 `edge_ngram` tokenizer，可在 mapping 中配置 min_gram / max_gram 参数。广泛用于中文搜索和自动补全
- **SQLite FTS**：支持 fts5 的 ngram tokenizer，用于 CJK 全文搜索
- **PostgreSQL**：`pg_trgm` 扩展基于 trigram 提供相似度搜索和模糊匹配，支持 `%` 运算符和 `similarity()` 函数
- **Google / Bing 拼写纠错**：底层利用 N-gram 统计模型计算编辑距离和相似度
