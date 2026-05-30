---
title: Binary 指纹索引
subtitle: 二进制指纹与相似度度量
date: 2026-05-30T10:13:46+08:00
slug: 9fc21a5
draft: false
author:
  name:
  link:
  email:
  avatar:
description: 二进制指纹索引原理，涵盖 Hamming 距离、Jaccard 相似度、Tanimoto 系数、Superstructure 和 Substructure 筛选，以及它们在化学信息学和信息检索中的应用
keywords:
  - binary fingerprint
  - hamming
  - jaccard
  - tanimoto
  - substructure
  - superstructure
  - 分子指纹
  - 相似度搜索
license:
comment: false
weight: 0
tags:
  -
categories:
  - index
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary: 二进制指纹索引原理，涵盖 Hamming 距离、Jaccard 相似度、Tanimoto 系数、Superstructure 和 Substructure 筛选，以及它们在化学信息学和信息检索中的应用
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: false
password:
message:
repost:
  enable: false
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

## 什么是二进制指纹

二进制指纹（Binary Fingerprint），也称为位向量（Bit Vector），是一种将对象编码为固定长度二进制向量的技术。向量中的每一位（0 或 1）表示该对象是否具有某个特征。二进制指纹将复杂的对象——分子结构、文本文档、图像特征——压缩为等长的二进制序列，使相似度计算转化为高效的位运算。

在化学信息学中，分子指纹将分子的结构特征编码为二进制向量。例如 MACCS keys（166 位）中每一位对应一个预定义的子结构查询，Morgan/ECFP 指纹（通常 1024 或 2048 位）则通过迭代聚合原子邻域信息来编码局部环境。

在信息检索中，文档指纹将文档的内容特征编码为二进制向量。SimHash 将文档的词频特征压缩为固定位数的签名，Bloom Filter 用多个哈希函数将特征映射到位向量上，支持集合关系的快速判断。

## 为什么需要二进制指纹索引

在大规模数据集中进行相似度搜索时，直接比较原始对象的代价极高。分子结构比对（子图同构）是 NP-hard 问题，文档全文比较的空间和时间开销同样巨大。

二进制指纹将比较问题转化为位运算，带来几个关键优势：

- **计算高效**：相似度计算转化为按位 AND、OR、XOR 和 popcount 操作，CPU 原生支持，单次比较仅需纳秒级
- **空间紧凑**：一个 2048 位的指纹仅需 256 字节，适合内存缓存和批量加载
- **可索引**：基于指纹可以构建倒排索引、树形索引或局部敏感哈希（LSH），实现亚线性检索
- **通用性**：相同的相似度度量框架可应用于不同领域的对象

## 指纹生成

### 分子指纹

分子指纹的生成方式主要有两类：

**子结构键控指纹（Keyed Fingerprint）**：预定义一组结构模式（子结构、官能团），对分子逐一检测每个模式是否存在。MACCS keys 是典型代表，166 个键中每个对应一个明确定义的子结构查询，如"是否存在芳环""是否有羟基"。

**拓扑指纹（Topological Fingerprint）**：基于分子图的拓扑结构生成指纹。以 Morgan 算法（ECFP，Extended-Connectivity Fingerprint）为例：从每个原子出发，迭代式地将邻近原子和键的信息聚合为特征标识，再哈希到指定位上。Morgan 指纹能捕获分子的局部环境特征，是虚拟筛选中最常用的指纹类型。

两类指纹各有侧重：键控指纹可解释性强（每位对应已知子结构），拓扑指纹覆盖面广（能编码未预定义的结构模式）。

### 文档指纹

文档指纹的生成方式包括：

**词袋/特征映射**：将文档中的词（或 N-gram）通过哈希函数映射到二进制向量的特定位上。每个特征对应一个或多个位，存在则为 1。

**SimHash**：一种局部敏感哈希算法。对文档的每个特征计算 n 位哈希值，然后根据哈希的每一位对 n 维向量做加权累加（哈希位为 1 则加，为 0 则减），最终取符号得到 n 位二进制指纹。SimHash 的核心特性是：相似文档的指纹 Hamming 距离小，这使得它可以用于快速近似最近邻搜索。

**Bloom Filter**：用 k 个独立的哈希函数将文档的每个特征映射到位向量的 k 个位置上。Bloom Filter 能高效判断某个特征是否"可能存在"（可能有假阳性，但没有假阴性），适用于集合包含关系的快速筛选。

## 基本统计量

在介绍各相似度度量之前，先定义两个二进制向量之间的基本统计量。

给定两个等长的二进制向量 A 和 B：

- **a**：仅在 A 中置 1 的位数
- **b**：仅在 B 中置 1 的位数
- **c**：在 A 和 B 中同时置 1 的位数（即 `A AND B` 的 popcount）
- **d**：在 A 和 B 中同时为 0 的位数

其中 $a + b + c + d = n$（n 为指纹长度）。

在集合语义下，c 是交集大小，$a + b + c$ 是并集大小，d 是两个对象都"没有的特征"。

## Hamming 距离

Hamming 距离衡量两个等长二进制向量在多少个位上的值不同：

$$d_H(A, B) = a + b = \sum_{i=1}^{n}(A_i \oplus B_i)$$

即按位 XOR 后的 popcount。计算极其简单——一次 XOR 运算加一次 popcount。

Hamming 距离取值范围为 $[0, n]$，值越小表示越相似。它是一种严格的距离度量，满足非负性、对称性和三角不等式。

Hamming 距离对所有差异位一视同仁，不区分"共同置 1"和"共同置 0"的贡献。在指纹稀疏（大部分位为 0）的场景下，d 会占据绝对多数，此时 Hamming 距离的区分度会下降——两个完全无关的指纹可能因为大量共同为 0 的位而显得"很近"。

**应用场景**：SimHash 去重是 Hamming 距离最典型的应用。Google 最早提出：两个网页的 SimHash 指纹 Hamming 距离不超过 3（在 64 位指纹上），即判定为近似重复。这种方法将 O(n²) 的全量比较转化为基于前缀树的快速匹配，大幅降低计算量。

## Jaccard 相似度

Jaccard 相似度衡量两个集合的重叠程度：

$$J(A, B) = \frac{c}{a + b + c} = \frac{|A \cap B|}{|A \cup B|}$$

分子是共同置 1 的位数（`A AND B` 的 popcount），分母是至少一个向量置 1 的位数（`A OR B` 的 popcount）。

Jaccard 相似度取值范围为 $[0, 1]$，值越大表示越相似。它忽略共同为 0 的位 d，只关注"有特征"的重叠——这对稀疏指纹特别合适，因为共同为 0 不应被视为"相似"的证据（两个分子都没有某个罕见子结构，不代表它们相似）。

**应用场景**：文档去重和相似文档发现。MinHash 是 Jaccard 相似度的经典近似算法，通过对集合进行多次独立哈希采样来估计 Jaccard 值，可将 Jaccard 计算从 O(n) 降到 O(k)（k 为哈希函数数量，通常远小于指纹长度）。MinHash 配合 LSH（局部敏感哈希）可实现大规模数据集的亚线性相似度搜索。

## Tanimoto 系数

Tanimoto 系数在二进制向量上的定义为：

$$T(A, B) = \frac{c}{a + b + c} = \frac{|A \cap B|}{|A \cup B|}$$

对于二进制向量，Tanimoto 系数与 Jaccard 相似度在数学上完全等价。两者的区别在于学术传统：

- "Jaccard 相似度"是信息检索、数据挖掘领域的标准术语，得名于 Paul Jaccard（1912 年）
- "Tanimoto 系数"是化学信息学领域的标准术语，得名于 T.T. Tanimoto（1960 年代）

在化学信息学中，"Tanimoto ≥ 0.85"是判断两个分子"结构相似"的经验阈值，广泛应用于虚拟筛选和化合物聚类。这个阈值并非理论推导，而是大量实证研究的经验总结——Tanimoto 值高于 0.85 的分子对在生物活性上往往表现出显著的相关性。

Tanimoto 系数也可以推广到连续值向量，此时与 Jaccard 不再等价：

$$T_{cont}(\mathbf{x}, \mathbf{y}) = \frac{\mathbf{x} \cdot \mathbf{y}}{\|\mathbf{x}\|^2 + \|\mathbf{y}\|^2 - \mathbf{x} \cdot \mathbf{y}}$$

当向量退化为二进制时，此公式自然回归为 Jaccard/Tanimoto 的形式。

## Superstructure 筛选

Superstructure 是化学信息学中的概念。给定查询分子 Q 和目标分子 T，如果 T 是 Q 的子结构（T 被包含在 Q 中），则称 Q 是 T 的超结构（Superstructure）。

在指纹层面，这是一个子集关系判断：

$$\text{bits}(T) \subseteq \text{bits}(Q)$$

即 T 中置 1 的所有位，在 Q 中也必须为 1。等价地，$(T \mathbin{\&} \neg Q) = 0$——T 中没有任何不属于 Q 的置 1 位。

实际操作中，只需计算 `T AND (NOT Q)` 的 popcount 是否为零。这是一个布尔判断（匹配或不匹配），不像前三种度量返回连续值。

指纹层面的 Superstructure 筛选是**必要条件而非充分条件**。因为哈希碰撞，指纹可能"虚假地"满足子集关系（某个位因碰撞而置 1，但分子并不真正包含对应的结构特征）。因此指纹筛选通过后，还需做精确的子图同构验证来排除假阳性。

**应用场景**：逆向分子搜索——已知一个核心骨架（如苯环），找出数据库中所有包含该骨架的分子。药物发现中寻找具有特定药效团的所有化合物。

## Substructure 筛选

Substructure 是 Superstructure 的镜像。给定查询分子 Q 和目标分子 T，如果 Q 被包含在 T 中，则称 Q 是 T 的子结构（Substructure）。

在指纹层面：

$$\text{bits}(Q) \subseteq \text{bits}(T)$$

即 Q 中置 1 的所有位，在 T 中也必须为 1。等价地，$(Q \mathbin{\&} \neg T) = 0$。

这等价于 $a = 0$（Q 中没有不属于 T 的置 1 位）且 $c > 0$（至少有一个共同置 1 的位）。

Substructure 筛选同样只是必要条件。精确匹配需要子图同构算法（如 Ullmann 算法、VF2 算法），这是一个 NP-complete 问题，因此指纹预筛选的意义重大——它可以在对数或线性时间内排除绝大多数不匹配的分子，只把少量候选交给昂贵的精确验证。

**应用场景**：子结构搜索是化学数据库最常用的查询方式。化学家用一个查询分子（或子结构模式）在百万甚至十亿级分子库中快速筛选，先指纹过滤，再精确匹配。在信息检索中，Substructure 筛选等价于"查询的所有特征都被文档覆盖"的包含性搜索。

## 相似度度量对比

| 度量 | 公式 | 值域 | 类型 | 考虑共同为 0 | 典型应用 |
|------|------|------|------|------------|---------|
| Hamming | $a + b$ | $[0, n]$ | 距离 | 是 | SimHash 去重 |
| Jaccard | $c / (a+b+c)$ | $[0, 1]$ | 相似度 | 否 | 文档去重、聚类 |
| Tanimoto | $c / (a+b+c)$ | $[0, 1]$ | 相似度 | 否 | 分子相似度、虚拟筛选 |
| Superstructure | $T \subseteq Q$ | {true, false} | 布尔 | - | 逆向分子搜索 |
| Substructure | $Q \subseteq T$ | {true, false} | 布尔 | - | 子结构查询 |

Hamming 和 Jaccard/Tanimoto 适用于"相似程度"的度量（返回连续值），Superstructure 和 Substructure 适用于"包含关系"的判断（返回布尔值）。在稀疏指纹场景下，Jaccard/Tanimoto 通常优于 Hamming，因为后者会被大量共同为 0 的位稀释区分度。

## 实际应用

**化学信息学**：

- **RDKit**：开源化学信息学库，支持 Morgan、MACCS、RDKit 等多种指纹，内置 Tanimoto 相似度搜索和子结构筛选，支持指纹批量计算和相似度矩阵生成
- **OpenBabel**：支持 FP2、FP3、FP4 等指纹类型，提供相似度搜索和子结构匹配
- **PubChem / ChEMBL**：基于指纹和 Tanimoto 系数提供大规模化合物相似度搜索服务，PubChem 的化合物库超过十亿级别
- **化学数据库子结构搜索**：典型的两阶段流程——先指纹筛选（快速排除不匹配的分子），再精确子图同构验证（确认匹配），指纹筛选通常能排除 95% 以上的不匹配分子

**信息检索**：

- **Google 网页去重**：使用 SimHash + Hamming 距离检测近似重复页面，64 位 SimHash 指纹 Hamming 距离 ≤ 3 即判定为近似重复
- **MinHash + LSH**：在大规模文档集合中快速估计 Jaccard 相似度，用于聚类、去重和近似最近邻搜索
- **推荐系统**：基于用户行为生成二进制指纹（浏览/购买/点击），用 Jaccard/Tanimoto 计算用户或物品的相似度
