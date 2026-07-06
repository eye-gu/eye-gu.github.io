---
title: es
subtitle:
date: 2024-10-30T09:36:47+08:00
slug: 27add30
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
categories:
  - search
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



## docker

https://www.cnblogs.com/nhdlb/p/16551485.html


包含kibana

```shell
docker network create elasticsearch_network

docker run -d --name elasticsearch \
-e ELASTICSEARCH_PASSWORD=123456 \
-e TZ=Asia/Shanghai \
-v /Users/guzemin/docker/elasticsearch/data:/bitnami/elasticsearch/data \
-p 9200:9200 \
--net=elasticsearch_network \
bitnami/elasticsearch:8.13.2

docker run -d -p 5601:5601 --name kibana --net=elasticsearch_network \
-e KIBANA_ELASTICSEARCH_URL=elasticsearch \
bitnami/kibana:8.13.2
```



```JSON
{"query":{ 
"match":{"rowKey":"11158546311053316"} 
}} 

{"query":{"match_all":{}}} 
```



## 创建index

```
PUT /product4
{
  "mappings": {
      "properties":{
        "name":{
          "type":"keyword"
        },
        "age":{
          "type": "long"
        },
        "address":{
          "type":"text"
        },
        "birthday":{
           "type": "date",
           "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis" 
        }
      }
   
  },
  "settings":{
    "index":{
      "number_of_shards":1,
      "number_of_replicas":1
    }
  }
}
```



## 删除index

```shell
curl -XDELETE 'http://127.0.0.1:9200/product' --user elastic:123456
```

## es7.10

```shell
docker run -d --name elasticsearch7 -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.10.1
```


## 经典面试题


### 1. 什么是 Elasticsearch？它的核心特性是什么？

Elasticsearch 是一个基于 Apache Lucene 的分布式、RESTful 风格的搜索和分析引擎。

核心特性：分布式架构（自动分片与副本）、近实时搜索、全文检索、支持结构化/非结构化/地理位置等多样化数据、水平可扩展、提供丰富的聚合分析能力。


### 2. ES 的基本概念：Index、Type、Document、Field、Shard、Replica

Index（索引）：类似数据库中的 database，是 Document 的集合。ES 7.x 开始一个 Index 默认只有一个 Type，8.x 完全移除 Type。

Type（类型）：类似数据库中的 table，ES 7.x 后已废弃，8.x 彻底移除。

Document（文档）：类似数据库中的 row，是 ES 中可被索引的基本信息单元，JSON 格式。

Field（字段）：类似数据库中的 column，是 Document 中的键值对。

Shard（分片）：Index 可以拆分为多个 Shard 分布在不同节点上，分为主分片（primary shard）和副本分片（replica shard）。主分片数在索引创建时确定，不可更改；副本分片数可动态调整。

Replica（副本）：主分片的拷贝，用于高可用和读负载均衡。当主分片所在节点故障时，副本可提升为主分片。


### 3. 倒排索引是什么？ES 为什么快？

倒排索引（Inverted Index）是 ES 全文检索的核心数据结构。它不是从文档到词，而是从词到文档的映射。

构建过程：对文档内容分词 → 去停用词 → 词项归一化（stemming 等）→ 建立 term 到 document id 列表的映射，同时记录词频和位置信息（用于短语查询和高亮）。

```
# 正排索引
doc1: "hello world"
doc2: "hello elasticsearch"

# 倒排索引
hello    -> [doc1, doc2]
world    -> [doc1]
elasticsearch -> [doc2]
```

ES 快的原因：倒排索引使得查询时无需扫描全部文档，直接通过 term 定位到对应的 doc id 列表；Lucene 还使用了 FST（Finite State Transducer）压缩 term 字典，跳表（Skip List）加速 posting list 合并，以及 BKD-Tree 处理数值/地理范围查询，Block-K-Way Merge 合并段等优化。


### 4. text 和 keyword 类型的区别？

text 类型：会被分词器（analyzer）处理，拆分为多个 term 存入倒排索引，适用于全文检索（如文章正文、商品描述）。text 字段不用于排序和聚合。

keyword 类型：不分词，整个值作为一个 term 存入倒排索引，适用于精确匹配、排序、聚合（如 ID、标签、状态码）。

```json
{
  "properties": {
    "title": { "type": "text" },
    "tags": { "type": "keyword" }
  }
}
```

实际使用中常对同一字段同时映射 text 和 keyword：

```json
{
  "properties": {
    "title": {
      "type": "text",
      "fields": {
        "keyword": { "type": "keyword", "ignore_above": 256 }
      }
    }
  }
}
```


### 5. 分词器（Analyzer）的组成和工作流程？

分词器由三部分组成：

Character Filters（字符过滤器）：在分词前对原始文本处理，如 HTML 去标签（html_strip）。

Tokenizer（分词器）：将文本拆分为 term，如 standard、whitespace、keyword。

Token Filters（token 过滤器）：对分词后的 term 处理，如小写化（lowercase）、去停用词（stop）、同义词（synonym）、词干提取（stemmer）。

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip"],
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "stemmer"]
        }
      }
    }
  }
}
```

中文分词常用 IK 分词器，支持 `ik_smart`（粗粒度）和 `ik_max_word`（细粒度）两种模式。


### 6. match、term、match_phrase 的区别？

term：不对查询条件分词，精确匹配。适用于 keyword、数值、布尔等字段。

match：对查询条件分词后与倒排索引匹配，多个 term 之间默认是 OR 关系。适用于 text 字段。

match_phrase：分词后要求所有 term 都出现且顺序一致、位置相邻（可通过 slop 参数允许一定距离的间隔）。

```json
// term - 精确匹配
{ "query": { "term": { "status": "published" } } }

// match - 全文匹配
{ "query": { "match": { "title": "elasticsearch guide" } } }

// match_phrase - 短语匹配
{ "query": { "match_phrase": { "title": "elasticsearch guide" } } }
```


### 7. bool 查询的几种类型？

bool 查询组合多个查询条件，支持四种子句：

must：必须匹配，参与打分。

should：至少匹配一个（当没有 must/filter 时）或匹配越多得分越高（有 must 时）。可用 `minimum_should_match` 控制最少匹配数。

must_not：必须不匹配，不参与打分。

filter：必须匹配，但不参与打分（结果缓存，性能优于 must）。

```json
{
  "query": {
    "bool": {
      "must": [{ "match": { "title": "elasticsearch" } }],
      "filter": [{ "range": { "price": { "gte": 100, "lte": 500 } } }],
      "must_not": [{ "term": { "status": "deleted" } }],
      "should": [{ "term": { "category": "tech" } }],
      "minimum_should_match": 1
    }
  }
}
```


### 8. filter 和 query 的区别？为什么 filter 更快？

query context：计算相关性得分（_score），用于判断文档匹配程度。

filter context：只判断是否匹配（yes/no），不计算得分。ES 会缓存 filter 的结果，后续相同查询直接命中缓存。

结论：不需要排序的场景应优先用 filter，性能更好。


### 9. 深度分页问题及解决方案？

ES 分页有三种方式：

from + size：默认方式。`from` 表示跳过的文档数。ES 需要从每个分片取 `from + size` 条数据，协调节点合并排序后取 size 条。当 from 很大时（如 from=10000），每个分片都要取大量数据，内存和排序开销巨大。默认限制 `from + size <= 10000`。

```json
{ "from": 9900, "size": 100 }
```

scroll：创建快照，通过 scroll_id 逐批获取。适用于深分页和数据导出，但不适合实时查询（数据快照不变），且占用内存资源。

```json
// 第一次请求
POST /index/_search?scroll=1m
{ "size": 100, "query": { "match_all": {} } }

// 后续请求
POST /_search/scroll
{ "scroll": "1m", "scroll_id": "xxx" }
```

search_after：基于上一页最后一条数据的排序值获取下一页，要求必须有唯一排序字段。适用于实时深分页，但不支持随机跳页。

```json
// 第一页
{
  "size": 100,
  "sort": [{ "timestamp": "asc" }, { "_id": "asc" }]
}

// 下一页（传入上一页最后一条的 sort 值）
{
  "size": 100,
  "sort": [{ "timestamp": "asc" }, { "_id": "asc" }],
  "search_after": ["2024-01-01", "abc123"]
}
```


### 10. 如何保证 ES 与数据库的数据一致性？

常见方案：

同步双写：业务代码中先写数据库再写 ES，事务中处理。简单但耦合度高，ES 写入失败需处理。

异步消息（MQ）：写数据库后发消息到 MQ，消费者异步写 ES。解耦，但有短暂延迟，需处理消息消费失败。

Binlog 订阅（Canal/Debezium）：监听数据库 binlog，解析后同步到 ES。业务代码零侵入，最终一致性好，是主流方案。

定时任务补偿：定时全量或增量同步。简单但实时性差，通常作为兜底方案。

实践建议：根据一致性要求选择，金融等强一致场景用同步双写 + 补偿；一般业务用 Binlog 订阅 + MQ 异步。


### 11. ES 的相关性打分（TF-IDF / BM25）原理？

ES 5.x 之前使用 TF-IDF 算法，5.x 之后默认使用 BM25。

TF-IDF：

TF（Term Frequency）= 词在文档中出现次数 / 文档总词数。出现越多，得分越高。

IDF（Inverse Document Frequency）= log(总文档数 / 包含该词的文档数)。词在越多文档出现，区分度越低，得分越低。

得分 = TF * IDF。

BM25（Best Matching 25）：基于 TF-IDF 改进，引入两个参数：

k1（默认 1.2）：控制词频饱和度。TF-IDF 中词频无上限，BM25 中词频贡献逐渐递减趋近上限，避免高频词过度影响得分。

b（默认 0.75）：控制文档长度归一化。文档越长，相同词频的得分越低，但影响有上限。

```
score(D, Q) = Σ IDF(qi) * (f(qi, D) * (k1 + 1)) / (f(qi, D) + k1 * (1 - b + b * |D| / avgdl))
```


### 12. ES 集群的状态和选主机制？

集群状态（Cluster State）有三种：

Green：所有主分片和副本分片都正常分配。

Yellow：所有主分片正常，但部分副本分片未分配。

Red：部分主分片不可用，数据不完整，查询结果可能不准。

选主机制（ES 7.0+）：

基于 Raft 协议改进。节点启动后广播发现集群，若没有 master 则发起选举。只有持有 master-eligible 角色的节点可参与选举。获得多数派（quorum）投票的节点成为 master。master 负责维护集群状态、分片分配、节点故障检测。


### 13. ES 写入数据的流程？

1. 客户端写请求发到任意节点（协调节点）。

2. 协调节点根据 routing 计算目标主分片：`shard = hash(routing) % number_of_primary_shards`，默认 routing 为文档 _id。

3. 请求转发到主分片所在节点，主分片写入数据。

4. 主分片写入成功后并行转发到所有副本分片。

5. 所有副本写入成功后，主分片节点返回成功给协调节点，协调节点返回客户端。

写入细节：数据先写入 indexing buffer（内存），同时写入 translog（事务日志，持久化到磁盘）。refresh 操作（默认 1 秒）将 buffer 刷新为新的 segment，数据变为可搜索。flush 操作将 translog 清空并 fsync 所有 segment 到磁盘。


### 14. 近实时（NRT）是什么意思？refresh、flush、fsync 的区别？

ES 是近实时的，数据写入后默认 1 秒后可搜索，而非立即搜索。

refresh：将 indexing buffer 中的数据生成新的 segment 并写入文件系统缓存（OS cache），清空 buffer。此时数据可搜索，但尚未持久化到磁盘。默认 1 秒一次。

flush：将 OS cache 中的 segment fsync 到磁盘，清空 translog。数据真正持久化。默认 translog 达到 512MB 或 30 分钟触发。

fsync：操作系统级别的磁盘同步操作，确保数据从 OS cache 写入物理磁盘。


### 15. 如何优化 ES 的查询性能？

索引设计层面：合理设置分片数（单个分片建议 30-50GB），避免过多小分片；对不需要排序的字段关闭 doc_values；对不需要全文检索的字段用 keyword 代替 text。

查询层面：用 filter 代替 query（命中缓存）；避免 from 过大的深分页，用 search_after；只查询需要的字段（_source filtering）；避免使用 wildcard 前缀查询；script 查询性能差，尽量用内置查询。

聚合层面：合理设置分片大小和 shard_size；用 doc_values 而非 fielddata；避免在 text 字段上聚合。


### 16. 如何优化 ES 的写入性能？

批量写入：使用 bulk API，单次 batch 建议 5-15MB。

增大 refresh interval：从默认 1s 调大到 30s 甚至更长，减少 segment 生成。

写入时关闭副本：`number_of_replicas: 0`，写完再恢复。减少副本同步开销。

禁用 refresh 和 translog 的 flush：大量写入时临时禁用。

增大 indexing buffer：`indices.memory.index_buffer_size`，默认 10%。

合理设置分片数：分片太少写入瓶颈，太多增加开销。


### 17. ES 的脑裂问题是什么？如何避免？

脑裂：网络分区导致集群中多个节点同时认为自己是 master，产生多个独立的集群状态，数据不一致。

ES 7.0+ 通过以下机制避免脑裂：

自动管理 master-eligible 节点数量，无需手动配置 `discovery.zen.minimum_master_nodes`。选主必须获得多数派投票（quorum = master_eligible / 2 + 1），确保只有一个 master。

旧版本（7.0 之前）需手动设置 `discovery.zen.minimum_master_nodes` 为 master-eligible 节点数的一半加一。


### 18. mapping 和 dynamic mapping 的区别？

dynamic mapping（动态映射）：ES 自动推断字段类型并创建映射。方便但有风险——可能推断出不理想的类型，且字段一旦确定无法修改（需 reindex）。

explicit mapping（显式映射）：手动定义字段类型，推荐生产环境使用。

```json
PUT /my_index
{
  "mappings": {
    "dynamic": "strict",  // 严格模式：未知字段报错
    "properties": {
      "title": { "type": "text" },
      "price": { "type": "double" }
    }
  }
}
```

dynamic 可选值：true（自动添加）、runtime（运行时字段）、false（忽略新字段）、strict（报错）。


### 19. ES 如何实现聚合查询？

三种聚合类型：

Bucket aggregation（桶聚合）：按条件分组，类似 SQL GROUP BY。

Metric aggregation（指标聚合）：计算统计值，如 avg、max、min、sum、cardinality。

Pipeline aggregation（管道聚合）：基于其他聚合结果二次计算。

```json
{
  "aggs": {
    "by_category": {
      "terms": { "field": "category.keyword", "size": 10 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "price_stats": { "stats": { "field": "price" } }
      }
    }
  }
}
```


### 20. ES 和 MySQL 各自的适用场景？

MySQL：强事务一致性、复杂关联查询（JOIN）、固定 schema、数据量适中。适用于订单、账户、核心业务数据。

ES：全文检索、海量数据的多维聚合分析、近实时搜索、日志分析。适用于搜索、推荐、日志、监控、数据分析。

典型架构：MySQL 作为主数据存储，ES 作为搜索和分析引擎，通过数据同步保持最终一致。



