---
title: Topn写入
subtitle:
date: 2026-03-16T21:50:55+08:00
slug: 418c478
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



## 数据分布

```go
shard_id = hash(sharding_key) % shard_num
node = (shard_id + replica_id) % node_count
```



## topn写入


### 1. 概述

TopN Aggregation 是 BanyanDB 中针对 Measure 数据的实时流式聚合机制。其核心思想是：当原始 Measure 数据写入时，同步将数据点分流到 TopN 流处理管道，经过 `Filter → Map → Window(Tumbling) → TopN聚合 → Sink` 的流水线处理，最终将 TopN 聚合结果以二进制编码形式写回到一个特殊的内部 Measure（名为 `_top_n_result`）中。

本文聚焦于 **写入阶段**（Write Phase），即从数据点进入系统到 TopN 聚合结果被持久化的完整流程。

---

### 2. 核心数据结构

#### 2.1 TopNAggregation Schema（Proto 定义）

**文件**: `api/proto/banyandb/database/v1/schema.proto`

```protobuf
message TopNAggregation {
  common.v1.Metadata metadata = 1;           // 聚合的身份标识
  common.v1.Metadata source_measure = 2;     // 数据源 Measure
  string field_name = 3;                     // 排序字段名
  model.v1.Sort field_value_sort = 4;        // 排序方向(ASC/DESC/UNSPECIFIED)
  repeated string group_by_tag_names = 5;    // 分组标签名
  model.v1.Criteria criteria = 6;            // 过滤条件
  int32 counters_number = 7;                 // 跟踪的计数器数量(默认1000)
  int32 lru_size = 8;                        // 内存中允许维护的窗口数
  google.protobuf.Timestamp updated_at = 9;
}
```

**关键字段说明**：
- `field_value_sort`：`SORT_UNSPECIFIED` 表示同时创建 ASC 和 DESC 两个处理器；`SORT_ASC` 为 BottomN；`SORT_DESC` 为 TopN
- `counters_number`：每个分组内跟踪的 TopN 数量上限（即 N 值）
- `lru_size`：LRU 缓存中允许同时存在的最大时间窗口数

#### 2.2 `_top_n_result` 内部 Measure Schema

**文件**: `banyand/measure/metadata.go`

系统在每个 group 中自动创建一个名为 `_top_n_result` 的特殊 Measure：

| 组成部分 | 名称 | 类型 |
|----------|------|------|
| Tag Family | `_topN` | - |
| Tag: name | `name` | STRING（TopN 聚合名称） |
| Tag: direction | `direction` | INT（排序方向枚举值） |
| Tag: group | `group` | STRING（分组键） |
| Tag: parameters | `parameters` | STRING（TopN 参数 JSON） |
| Entity | 以上4个 Tag | 复合实体 |
| Field | `value` | BINARY（序列化的 TopNValue） |

#### 2.3 核心运行时结构体

```
schemaRepo
  └── topNProcessorMap: sync.Map   // key: "group/measureName", value: *topNProcessorManager
        └── topNProcessorManager    // 每个 Measure 一个
              ├── m: *databasev1.Measure
              ├── registeredTasks: []*databasev1.TopNAggregation
              └── processorList: []*topNStreamingProcessor  // 每个 TopN + 排序方向 一个
                    ├── src: chan interface{}           // 数据注入通道
                    ├── streamingFlow: flow.Flow       // 流处理管道
                    ├── in: chan flow.StreamRecord      // Sink 输入通道
                    ├── pipeline: queue.Client          // 消息发布客户端
                    └── topNSchema: *databasev1.TopNAggregation
```

---

### 3. 写入阶段完整数据流

#### 3.1 整体流程图

```
用户写入 Measure 数据点
    │
    ▼
writeCallback.Rev() / writeQueueCallback.Rev()
    │
    ├─► processDataPoint() ──► 编码写入原始 Measure 的 tsTable
    │
    └─► schemaRepo.inFlow()  ──► 分流到 TopN 处理管道（异步 goroutine）
              │
              ▼
    topNProcessorManager.onMeasureWrite()
              │  (go func, 异步执行)
              ▼
    ┌─────────────────────────────────────────────┐
    │           流处理管道 (Streaming Pipeline)      │
    │                                              │
    │  src(channel)                                │
    │     │                                        │
    │     ▼                                        │
    │  Source(sourceChan) ──► 提取 StreamRecord     │
    │     │                                        │
    │     ▼                                        │
    │  Filter ──► 按 criteria 过滤数据点            │
    │     │                                        │
    │     ▼                                        │
    │  Map ──► 提取排序键/分组键/seriesID            │
    │     │                                        │
    │     ▼                                        │
    │  TumblingWindow(interval, lru_size)          │
    │     │  ← LRU 缓存管理窗口，定期/超时触发 Flush  │
    │     ▼                                        │
    │  topNAggregatorGroup ──► TreeMap 内存排序      │
    │     │  ← 窗口 Flush 时 Snapshot()             │
    │     ▼                                        │
    │  Sink(topNStreamingProcessor).In()            │
    └─────────────────────────────────────────────┘
              │
              ▼
    writeStreamRecord() ──► TopNValue 二进制编码
              │
              ▼
    publisher.Publish(TopicMeasureWrite)
              │
              ▼
    writeCallback ──► 写入 _top_n_result Measure
              │
              ▼
    tsTable.mustAddDataPoints() ──► Block 持久化
              │
    ┌─────────┴─────────┐
    │  Block Merge 阶段  │
    │  mergeTopNResult() │
    │  PostProcessor     │
    │  合并+重排序 TopN   │
    └───────────────────┘
```

#### 3.2 阶段 1：Schema 注册 — 建立流处理管道

**触发时机**：用户通过 gRPC/REST API 创建 `TopNAggregation` schema

**入口**: `banyand/measure/metadata.go:258-279`

```go
case schema.KindTopNAggregation:
    topNSchema := metadata.Spec.(*databasev1.TopNAggregation)
    manager := sr.getSteamingManager(topNSchema.SourceMeasure, sr.pipeline)
    manager.register(topNSchema)
```

**`getSteamingManager()`**（`topn.go:89-135`）的逻辑：
1. 检查 repo 是否正在关闭，若关闭则返回 nil
2. 尝试加载 source Measure 的 schema
3. 若 Measure schema 尚未加载，创建一个"占位" manager（延迟初始化）
4. 若已加载，检查 `ModRevision` 判断是否需要重建
5. 将 manager 存入 `topNProcessorMap`

**`topNProcessorManager.start()`**（`topn.go:597-647`）构建完整流处理管道：

```
排序方向判断:
  SORT_UNSPECIFIED → 创建2个 processor (ASC + DESC)
  SORT_ASC/DESC    → 创建1个 processor

每个 processor 的管道构建:
  srcCh := make(chan interface{})
  src, _ := sources.NewChannel(srcCh)
  streamingFlow := streaming.New(name, src)
  streamingFlow = streamingFlow.Filter(filters)    // 过滤
  streamingFlow = streamingFlow.Map(mapper)        // 映射
  // 最后通过 .Window().TopN().To(sink).Open() 完成
```

#### 3.3 阶段 2：数据写入入口 — 分流到 TopN

**入口**: `write_standalone.go:198-203`（standalone 模式）或 `write_liaison.go:236-241`（liaison 模式）

```go
// 写入原始 Measure
sid, err := processDataPoint(dpt, req, writeEvent, stm, is, ts, metadata, spec)
// 分流到 TopN 流处理管道
w.schemaRepo.inFlow(stm.GetSchema(), sid, writeEvent.ShardId, writeEvent.EntityValues, req.DataPoint, spec)
```

**`inFlow()`**（`topn.go:68-87`）检查该 Measure 是否有关联的 TopN 聚合：

```go
func (sr *schemaRepo) inFlow(stm, seriesID, shardID, entityValues, dp, spec) {
    if p, _ := sr.topNProcessorMap.Load(getKey(stm.GetMetadata())); p != nil {
        p.(*topNProcessorManager).onMeasureWrite(...)
    }
}
```

> **设计要点**：`inFlow()` 仅当 Measure 关联了 TopN 聚合时才执行，零开销路径。

#### 3.4 阶段 3：数据注入 — 异步分发

**文件**: `topn.go:534-560`

```go
func (manager *topNProcessorManager) onMeasureWrite(...) {
    go func() {  // 异步执行，不阻塞原始写入
        manager.RLock()
        defer manager.RUnlock()
        // ...
        for _, processor := range manager.processorList {
            dpWithEntity := newDataPointWithEntityValues(dp, request.GetEntityValues(), ...)
            processor.src <- flow.NewStreamRecordWithTimestampPb(dpWithEntity, dp.GetTimestamp())
        }
    }()
}
```

**`dataPointWithEntityValues`**（`topn.go:145-152`）封装了数据点及其上下文信息：

| 字段 | 类型 | 说明 |
|------|------|------|
| `DataPointValue` | embedded | 原始数据点（Tag/Field 值） |
| `entityValues` | `[]*modelv1.TagValue` | 实体标签值 |
| `seriesID` | `uint64` | 系列 ID |
| `shardID` | `uint32` | 分片 ID |
| `fieldIndex` | `map[string]int` | 字段名到索引的映射 |
| `tagSpec` | `TagSpecRegistry` | Tag 规格注册表 |

> **设计要点**：使用 `go func()` 异步执行，不阻塞原始数据的写入路径。读写锁保证并发安全。

#### 3.5 阶段 4：Source — 从 Channel 到 StreamRecord

**文件**: `pkg/flow/streaming/sources/channel.go:48-63`

```go
func (s *sourceChan) run(ctx context.Context) {
    for val := range s.in {  // 从 src channel 读取
        select {
        case s.out <- flow.TryExactTimestamp(val):  // 提取时间戳，输出到 out channel
        case <-ctx.Done():
            return
        }
    }
}
```

`TryExactTimestamp()` 检查数据是否已是 `StreamRecord`，若是则直接使用；否则尝试从 `TimestampMillis()` 接口提取时间戳。

#### 3.6 阶段 5：Filter — 条件过滤

**文件**: `topn.go:663-686`

```go
func (manager *topNProcessorManager) buildFilter(criteria *modelv1.Criteria) (flow.UnaryFunc[bool], error) {
    if criteria == nil {
        return func(_ context.Context, _ any) bool { return true }, nil  // 无条件，全部通过
    }
    f, err := logical.BuildSimpleTagFilter(criteria)
    return func(_ context.Context, request any) bool {
        ok, matchErr := f.Match(logical.TagFamiliesForWrite(tffws), tagSpec)
        return ok
    }, nil
}
```

- 若 `criteria` 为 nil，所有数据点直接通过
- 否则，使用 `logical.BuildSimpleTagFilter` 构建标签过滤器
- 过滤在 `unaryOperator.run()` 中以 goroutine 并发执行（`unary.go:90-110`）

#### 3.7 阶段 6：Map — 提取排序键与分组键

**文件**: `topn.go:688-741`

Map 算子将 `dataPointWithEntityValues` 转换为标准化的 `flow.Data` 数组：

```go
// 无 groupBy 的情况
return flow.Data{
    dpWithEvs.entityValues,     // [0] 实体标签值
    "",                          // [1] 空分组键
    dpWithEvs.intFieldValue(...), // [2] 排序字段值(int64)
    dpWithEvs.shardID,          // [3] 分片 ID
    dpWithEvs.seriesID,         // [4] 系列 ID
}

// 有 groupBy 的情况
groupValues := []string{...}  // 提取 groupBy 标签值
return flow.Data{
    dpWithEvs.entityValues,
    GroupName(groupValues),    // "tag1|tag2|tag3" 用 "|" 拼接
    dpWithEvs.intFieldValue(...),
    dpWithEvs.shardID,
    dpWithEvs.seriesID,
}
```

**`flow.Data` 索引约定**：

| 索引 | 含义 | 类型 |
|------|------|------|
| `[0]` | entityValues（实体标签值） | `[]*modelv1.TagValue` |
| `[1]` | groupKey（分组键） | `string` |
| `[2]` | 排序字段值 | `int64` |
| `[3]` | shardID | `uint32` |
| `[4]` | seriesID | `uint64` |

> **注意**：如果 groupBy 标签已从 schema 中删除，系统会打印警告日志并忽略这些标签。若所有 groupBy 标签均不存在，则整个 TopN 聚合被视为无效。

#### 3.8 阶段 7：Window — Tumbling 时间窗口

**文件**: `pkg/flow/streaming/sliding_window.go`

窗口大小 = Measure 的 `interval` 配置值。

**窗口分配**（`AssignWindows`，`sliding_window.go:290-299`）：
```go
func (s *tumblingTimeWindows) AssignWindows(timestamp int64) (flow.Window, error) {
    start := getWindowStart(timestamp, s.windowSize)
    return timeWindow{start: start, end: start + s.windowSize}, nil
}
```

**核心数据结构**：

| 字段 | 说明 |
|------|------|
| `snapshots` | LRU Cache（`lru.Cache`），存储 `timeWindow → AggregationOp` |
| `timerHeap` | 定时器堆，管理窗口触发时间 |
| `windowCount` | 最大窗口数（= `lru_size`） |
| `flushInterval` | flush 间隔 = min(interval × 40%, maxFlushInterval) |

**数据处理流程**（`receive()`，`sliding_window.go:166-233`）：

```
1. 接收 StreamRecord
2. AssignWindows(timestamp) → 分配到对应 timeWindow
3. isWindowLate(window)? → 若窗口已过期且不在 LRU 中，丢弃
4. snapshots.Get(tw)? 
   → 命中：oldAggr.Add([elem])
   → 未命中：newAggr = aggregationFactory(); newAggr.Add([elem]); snapshots.Add(tw, newAggr)
5. eventTimeTriggerOnElement(tw) → 判断是否立即触发
6. 更新 watermark → flushDueWindows() + flushDirtyWindows()
```

**LRU 驱逐机制**：
- 当 LRU Cache 满时，最旧的窗口被驱逐
- 驱逐回调触发 `flushSnapshot()`，将脏窗口数据发送到下游

**Flush 策略**（双触发机制）：
1. **Watermark 推进触发**：watermark 超过窗口 `MaxTimestamp` 时触发
2. **定时触发**：每隔 `flushInterval` 时间刷一次所有脏窗口

```
|<------------ interval (e.g. 5min) ------------>|
|         40%         |         40%         |20% |
|       flush         |       flush         |    |
```

#### 3.9 阶段 8：TopN 内存聚合

**文件**: `pkg/flow/streaming/topn.go`

**`topNAggregatorGroup`** 作为 `AggregationOp` 实现，管理所有分组的 TopN 聚合：

```go
type topNAggregatorGroup struct {
    aggregatorGroup   map[string]*topNAggregator  // groupKey → aggregator
    keyExtractor      func(StreamRecord) uint64     // 提取 seriesID
    sortKeyExtractor  func(StreamRecord) int64      // 提取字段值
    groupKeyExtractor func(StreamRecord) string     // 提取分组键
    comparator        utils.Comparator              // 排序比较器
    cacheSize         int                           // N 值
    sort              TopNSort                      // ASC / DESC
}
```

**`topNAggregator`**（`topn.go:88-93`）——单分组内的 TopN 排序器：

```go
type topNAggregator struct {
    *topNAggregatorGroup
    treeMap *treemap.Map       // 有序映射（key: sortKey, value: []StreamRecord）
    dict    map[uint64]int64   // seriesID → sortKey（用于去重）
    dirty   bool
}
```

##### 3.9.1 Add 操作（`topn.go:126-143`）

```
对每个输入 StreamRecord:
  1. key = seriesID, sortKey = 字段值, groupKey = 分组键
  2. 获取或创建该 groupKey 的 topNAggregator
  3. removeExistedItem(key) → 若同 seriesID 已存在，先移除旧值
  4. checkSortKeyInBufferRange(sortKey) → 判断是否在 TopN 范围内
     - 若 treeMap 为空，直接接受
     - 若 sortKey 优于当前最差值，接受
     - 若 buffer 未满（size < cacheSize），接受
  5. put(key, sortKey, data) → 插入 TreeMap
  6. doCleanUp() → 若 size > cacheSize，移除最差元素
```

##### 3.9.2 去重机制（`topn.go:243-264`）

```go
func (t *topNAggregator) removeExistedItem(key uint64) {
    existed, ok := t.dict[key]   // 查找旧 sortKey
    if !ok { return }
    delete(t.dict, key)
    // 从 TreeMap 中定位并移除旧记录
    list, ok := t.treeMap.Get(existed)
    // 遍历列表，找到匹配的 seriesID 并移除
    // 若列表为空，从 TreeMap 中删除该 sortKey
}
```

> 同一 seriesID 的新数据会替换旧数据，保证每个 series 在 TopN 中只出现一次。

##### 3.9.3 TreeMap 排序

- **DESC**（默认 TopN）：比较器 `Int64Comparator(b, a)`，最小的在"最后"
- **ASC**（BottomN）：比较器 `Int64Comparator`（自然序），最大的在"最后"

##### 3.9.4 Snapshot 操作（`topn.go:145-178`）

窗口触发时调用 `Snapshot()`：

```go
func (t *topNAggregatorGroup) Snapshot() interface{} {
    groupRanks := make(map[string][]*Tuple2)
    for group, aggregator := range t.aggregatorGroup {
        if !aggregator.dirty { continue }  // 跳过未变化的分组
        aggregator.dirty = false
        iter := aggregator.treeMap.Iterator()
        for iter.Next() {
            // 按 TreeMap 顺序遍历，提取 (sortKey, StreamRecord) 对
        }
        groupRanks[group] = items
    }
    return groupRanks  // map[string][]*Tuple2
}
```

> **`Tuple2`** 结构：`V1` = sortKey（int64），`V2` = StreamRecord

#### 3.10 阶段 9：Sink — 将聚合结果写入内部 Measure

**文件**: `topn.go:269-436`

`topNStreamingProcessor` 实现了 `flow.Sink` 接口。

##### 3.10.1 数据接收

`Setup()` 启动一个后台 goroutine 从 `in` channel 读取聚合结果：

```go
func (t *topNStreamingProcessor) run(ctx context.Context) {
    buf := make([]byte, 0, 64)
    for {
        select {
        case record, ok := <-t.in:
            if !ok { return }
            t.writeStreamRecord(record, buf)
        case <-ctx.Done():
            return
        }
    }
}
```

##### 3.10.2 writeStreamRecord — 核心写入逻辑

**步骤分解**：

**① 时间降采样**（`topn.go:337-439`）

```go
eventTime := t.downSampleTimeBucket(record.TimestampMillis())
// eventTimeMillis - eventTimeMillis % interval.Milliseconds()
// 将时间对齐到时间桶的开始
```

**② 遍历分组，编码 TopNValue**

```go
for group, tuples := range tuplesGroups {
    topNValue.Reset()
    topNValue.setMetadata(fieldName, entityTagNames)
    for _, tuple := range tuples {
        data := tuple.V2.(flow.StreamRecord).Data().(flow.Data)
        topNValue.addValue(tuple.V1.(int64), data[0].([]*modelv1.TagValue))  // 排序值 + 实体
        shardID = data[3].(uint32)
    }
}
```

**③ 构建实体标签**

```
entityValues = [
    {Str: topN名称},
    {Int: 排序方向枚举值},
    {Str: groupKey},
    {Str: parameters字符串(=counters_number)},
]
```

**④ 二进制序列化 TopNValue**（`topn.go:859-889`）

```
编码格式:
┌──────────────────────────────────────────────────────┐
│ valueName (length-prefixed bytes)                     │
│ entityTagNames count (varuint)                        │
│ entityTagName[0] (length-prefixed bytes)              │
│ entityTagName[1] (length-prefixed bytes)              │
│ ...                                                   │
│ values count (varuint)                                │
│ encodeType (1 byte)                                   │
│ firstValue (varint)                                   │
│ encoded values length (varuint)                       │
│ encoded values bytes (delta/zigzag/etc.)              │
│ entityValues block (length-prefixed block encoding)   │
│   entityValues[0] (pb tag values marshaled)           │
│   entityValues[1] (pb tag values marshaled)           │
│   ...                                                 │
└──────────────────────────────────────────────────────┘
```

其中 `values` 使用 `encoding.Int64ListToBytes()` 进行增量编码（delta encoding），支持多种编码类型以优化存储。

**⑤ 构建 InternalWriteRequest**

```go
iwr := &measurev1.InternalWriteRequest{
    Request: &measurev1.WriteRequest{
        Metadata: {Name: "_top_n_result", Group: topN.group},
        DataPoint: {
            Timestamp: eventTime,
            TagFamilies: [{Tags: entityValues}],
            Fields: [{BinaryData: serializedTopNValue}],
            Version: time.Now().UnixNano(),
        },
    },
    EntityValues: entityValues,
    ShardId:      shardID,
}
```

**⑥ 发布到消息总线**

```go
message := bus.NewBatchMessageWithNode(bus.MessageID(...), "local", iwr)
publisher.Publish(context.TODO(), apiData.TopicMeasureWrite, message)
```

#### 3.11 阶段 10：实际持久化

TopN 聚合结果像普通 Measure 数据一样进入写入队列：

**Standalone 模式**（`write_standalone.go:445-514`）：
1. `writeCallback.Rev()` 接收消息
2. `processDataPoint()` 处理数据点（编码 tag/field）
3. `tsTable.mustAddDataPoints()` 将数据加入内存 part
4. 最终由 flusher 持久化到磁盘

**Liaison 模式**（`banyand/liaison/grpc/topn.go:40-72`）：
1. `topNHandler.Rev()` 接收消息
2. 通过 `nodeRegistry.Locate()` 定位目标节点
3. 转发到对应 data node 的写入队列

#### 3.12 阶段 11：Block 合并时的 TopN 后处理

**文件**: `banyand/measure/block.go:1132-1217`，`banyand/measure/topn_post_processor.go`

当 Block 被合并（merge）时，如果其中包含 TopN 数据（二进制 field），需要特殊处理：

##### `blockPointer.mergeAndAppendTopN()` 流程：

```
1. 反序列化 left block 的 TopNValue
2. 反序列化 right block 的 TopNValue
3. 将所有实体-值对放入 PostProcessor
4. PostProcessor.Flush() 使用堆(Heap)重新排序，保留 TopN
5. 将合并后的结果重新序列化写回
```

##### PostProcessor（`topn_post_processor.go`）

- **无聚合函数模式**：按时间戳分组，每个时间戳维护一个优先队列
- **有聚合函数模式**（AVG/SUM/MAX/MIN/COUNT）：跨时间戳聚合，使用全局堆
- **版本号机制**：较新版本的数据覆盖旧数据（`version >= item.version`）

```go
func (taggr *topNPostProcessor) Put(entityValues, val, timestamp, version) {
    // 按时间戳找到或创建 timeline
    // 按 entityValues.String() 作为 key 去重
    // 版本号比较：较新的覆盖旧的
    // 维护优先队列大小 <= topN
}
```

---

### 4. 关键设计要点

#### 4.1 异步解耦

`onMeasureWrite()` 通过 `go func()` 异步执行，不阻塞原始数据写入路径。读写锁（`sync.RWMutex`）保证并发安全。

#### 4.2 双排序方向

当 `field_value_sort == SORT_UNSPECIFIED` 时，同时创建 ASC 和 DESC 两个独立的 `topNStreamingProcessor`，各自维护独立的流处理管道和 TreeMap。

#### 4.3 LRU 窗口管理

使用 LRU Cache 管理时间窗口，`lru_size` 控制最大窗口数。超出的旧窗口被驱逐并触发持久化。这保证了：
- 内存使用有上限
- 旧数据能及时刷盘
- 允许一定程度的乱序数据处理

#### 4.4 二进制编码优化

TopN 结果使用自定义二进制格式（`TopNValue.marshal()`），支持：
- 增量编码（delta encoding）压缩 int64 排序值
- BytesBlock 编码压缩实体标签值
- 对象池复用（`topNValuePool`）

#### 4.5 去重机制

`topNAggregator` 使用 `dict map[uint64]int64` 按 seriesID 去重。同一 seriesID 的新数据会替换旧数据，保证每个 series 在 TopN 中只出现一次。

#### 4.6 合并后处理

Block 合并时，通过 `PostProcessor`（基于堆）重新合并排序 TopN 结果，支持聚合函数（AVG/SUM/MAX/MIN/COUNT）的合并计算。

---

### 5. 涉及文件索引

| 文件 | 职责 |
|------|------|
| `banyand/measure/topn.go` | TopN 流处理核心：ProcessorManager、StreamingProcessor、TopNValue 序列化 |
| `banyand/measure/metadata.go` | Schema 管理：TopN 注册、`_top_n_result` Measure 创建 |
| `banyand/measure/write_standalone.go` | Standalone 写入入口：数据点处理、分流到 TopN |
| `banyand/measure/write_liaison.go` | Liaison 写入入口：集群模式数据点处理 |
| `banyand/measure/block.go` | Block 操作：`mergeTopNResult`、`mergeAndAppendTopN` |
| `banyand/measure/topn_post_processor.go` | TopN 后处理器：合并/去重/重排序 |
| `pkg/flow/streaming/topn.go` | 内存聚合引擎：`topNAggregatorGroup`、`topNAggregator` |
| `pkg/flow/streaming/sliding_window.go` | 滑动窗口：`tumblingTimeWindows`，LRU + 定时 Flush |
| `pkg/flow/streaming/streaming.go` | 流框架：`streamingFlow` 生命周期管理 |
| `pkg/flow/streaming/unary.go` | Filter/Map 算子：`unaryOperator` |
| `pkg/flow/streaming/sources/channel.go` | 数据源：`sourceChan`（Go channel → StreamRecord） |
| `pkg/flow/types.go` | 流框架接口：`Flow`、`WindowedFlow`、`AggregationOp`、`Sink` |
| `banyand/liaison/grpc/topn.go` | Liaison gRPC：TopN 消息路由 |
| `api/proto/banyandb/database/v1/schema.proto` | TopNAggregation Proto 定义 |

---

### 6. 配置参数参考

| 参数 | 来源 | 默认值 | 说明 |
|------|------|--------|------|
| `counters_number` | TopNAggregation schema | 1000 | 每个 groupKey 跟踪的 TopN 数量 |
| `lru_size` | TopNAggregation schema | 2 | LRU 中最大窗口数 |
| `interval` | Measure schema | - | 窗口大小（时间粒度） |
| `maxFlushInterval` | 代码常量 | 1 分钟 | 最大 flush 间隔 |
| `resultPersistencyTimeout` | 代码常量 | 10 秒 | 结果发布超时 |
| `flushInterval` | 计算 | min(interval×40%, maxFlushInterval) | 实际 flush 间隔 |

---

### 7. 生命周期管理

#### 7.1 创建流程

```
1. 用户创建 TopNAggregation schema
2. metadata.OnAddOrUpdate() → schema.KindTopNAggregation
3. getSteamingManager() → 获取或创建 topNProcessorManager
4. manager.register(topNSchema) → start() → 构建流处理管道
5. 管道开始运行，等待数据注入
```

#### 7.2 更新流程

```
1. 用户更新 TopNAggregation schema（ModRevision 递增）
2. manager.register() 检测到 ModRevision 变化
3. removeProcessors(旧 schema) → 移除旧 processor
4. start(新 schema) → 创建新 processor
5. 异步 Close() 旧 processor
```

#### 7.3 删除流程

```
1. 用户删除 TopNAggregation schema
2. metadata.OnDelete() → stopSteamingManager()
3. manager.Close() → 并行关闭所有 processor
4. 从 topNProcessorMap 中移除
```
