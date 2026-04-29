---
title: Topn查询
subtitle:
date: 2026-04-21T16:30:15+08:00
slug: 30c8e5d
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
  - banyandb
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



## TopN查询分析

数据来源不是原始 measure，而是 topnAggregation 写入的数据，主要涉及滚动时间桶、桶内去重裁剪等，详见 [topn写入]({{< relref "posts/banyandb/topn写入.md" >}})。

### standalone 模式

**代码分支：`main`**

standalone 和 distribute 模式查询逻辑不同，通过 `criteria.GetAgg() != 0` 判断是否为 standalone 模式。standalone 模式下会进行 group 聚合，数据范围是查询时间范围内所有时间桶的数据。

```go
// pkg/query/logical/measure/topn_analyzer.go
// standalone 的话 Agg 不等于 AGGREGATION_FUNCTION_UNSPECIFIED
if criteria.GetAgg() != 0 {
    groupByProjectionTags := sourceMeasureSchema.GetEntity().GetTagNames()
    groupByTags := [][]*logical.Tag{logical.NewTags(measure.TopNTagFamily, groupByProjectionTags...)}
    plan = newUnresolvedGroupBy(plan, groupByTags, false)
    plan = newUnresolvedAggregation(plan,
        &logical.Field{Name: topNAggSchema.FieldName},
        criteria.GetAgg(),
        true,
        false,
        false)
}

// standalone 和 distribute 都会做 topn 截断
plan = top(plan, &measurev1.QueryRequest_Top{
    Number:         criteria.GetTopN(),
    FieldName:      topNAggSchema.FieldName,
    FieldValueSort: criteria.GetFieldValueSort(),
})
```

### distribute 模式

**代码分支：`main`**

distribute 模式下，单个 datanode 不进行聚合，只进行 topn 排序裁剪，范围是查询时间范围内所有时间桶的数据。datanode 全部返回后，通过 `topNPostProcessor` 进行去重聚合。去重的原因：当前是广播查询所有 datanode，一个 shard 有多个副本，每个副本都会参与查询，同一个数据会被多个 datanode 返回。

```go
// banyand/dquery/topn.go
// distribute 会将 Agg 置为 AGGREGATION_FUNCTION_UNSPECIFIED，然后广播发给 datanode
agg := request.Agg
request.Agg = modelv1.AggregationFunction_AGGREGATION_FUNCTION_UNSPECIFIED
ff, err := t.broadcaster.Broadcast(defaultTopNQueryTimeout, data.TopicTopNQuery, bus.NewMessageWithNodeSelectors(now, nodeSelectors, request.TimeRange, request))
if err != nil {
    resp = bus.NewMessage(now, common.NewError("execute the query %s: %v", request.GetName(), err))
    return
}
```

### 问题

1. **standalone 和 distribute 查询逻辑不一致**：standalone 做 group 聚合后 topn，distribute 是每个datanode做 topn 截断, 再聚合。

2. **distribute 模式下 entity 覆盖不足**：如果某个 entity 的 value 特别大（例如某个 instance 延迟一直很高），datanode 的 topn 结果全是该 entity，最终 liaison 只返回该 entity 聚合的结果，无法满足 N 个不同 entity 的需求。

---

## 方案一：datanode 不截断

**代码分支：`fix-10277-interface`**

就个人第一反应而言, standalone 是正确的查询逻辑，正确处理了 topn 写入阶段所有数据的聚合查询。distribute 模式存在两个截断问题：

1. **跨节点截断**：某一 datanode 上排名较低的 entity，合并所有节点的部分结果后，可能在整体范围排名较高。

2. **节点内截断**：单个 datanode 可能包含同一 entity 的多个数据点（分布在不同 shard 上），但本地 topn 截断只保留最大的那个。例如 COUNT 聚合、TopN=3 时，某节点包含 entity5 的两个数据点，只有较大的发送给 liaison，导致计数为 1 而非 2。

   注意到 distribute 查询开始阶段特地 `request.Agg = AGGREGATION_FUNCTION_UNSPECIFIED`，因此认为可能是忘了处理 topn 参数——只要让 datanode 返回所有数据，liaison 就能正确聚合。

   **修改方式**：设置 `topn=0` 作为特殊值，修改 datanode 侧 `TopQueue#Insert` 逻辑，等于 0 时不截断，全部保留。

   **问题**：数据量大时容易 OOM，网络传输也是重大耗时。

---

## 方案二：下推聚合（pushdown）

**代码分支：`fix-topn-distributed-query`**

Issue: https://github.com/apache/skywalking/issues/13811

针对方案一 OOM 和网络传输问题，采用下推方案：

1. **单节点聚合**：datanode 下推进行聚合计算，不丢弃数据。
2. **跨节点截断**：理论上下推后单节点聚合结果中排名靠后的 entity 全局可能靠前。但经与维护者确认，**设计上不允许 entity 跨节点**（当前仅未对 sharding_key 做校验），只要保证 sharding_key 是 entity tags 的前缀，就能保证 entity 不跨节点，因此无需考虑此问题。

### 副本去重

liaison 进行 reduce 聚合时，同一 shard 的不同副本都会返回聚合结果。因此 datanode 聚合时需要按 shardId 分组，不跨 shard 聚合；返回给 liaison 时携带 shardId，liaison 根据 shardId + entity 去重。

### 性能分析

scan 数据量不变，磁盘 IO、内存 GC、反序列化 CPU、网络 IO 都一致，仅仅是 datanode 在内存中多做了一次 group 聚合。写入阶段每个时间桶的最大数量已确定，整体数据量不大。详见 [topn下推性能测试]({{< relref "posts/banyandb/topn下推性能测试.md" >}})。

---

## 方案三：entity 去重

**代码分支：`fix-13837-topn-entity-deduplication`**

Issue: https://github.com/apache/skywalking/issues/13837

前两种方案社区维护者均未同意，最终决定的 datanode 查询语义为：

```sql
SELECT entity, MAX(value) as val FROM data GROUP BY entity ORDER BY val DESC LIMIT N
```

从原来的全局 topn 排序，改成 entity 去重后的全局 topn 排序。这个逻辑存在问题，例如 COUNT 聚合时 liaison 结果一定为 1。但维护者坚持认同此方案。

所以 standalone 也需要进行逻辑修改, 保持两者查询逻辑相同.
