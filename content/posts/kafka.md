---
title: kafka
subtitle:
date: 2024-10-30T10:47:41+08:00
slug: 3bbcf35
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
  - mq
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



## docker部署

```shell
docker run -d  \
  --name kafka \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.0.232:9092 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  -e KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=0 \
  -e KAFKA_NUM_PARTITIONS=1 \
  -p 9092:9092 \
  apache/kafka:3.8.0
```

bitnami的貌似有点问题，还没有解决

```shell
docker run -d --name kafka \
    -e TZ=Asia/Shanghai \
    -e KAFKA_CFG_NODE_ID=0 \
    -e KAFKA_CFG_PROCESS_ROLES=controller,broker \
    -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
    -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
    -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@localhost:9093 \
    -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
    -v /Users/guzemin/docker/kafka/data:/bitnami/kafka \
    -p 9092:9092 \
    bitnami/kafka:3.7
```

## 本地部署

```shell
## zk
bin/kafka-server-start.sh config/server.properties

## kraft Ol32dbtpRTKtUJX2KKpRHQ
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
bin/kafka-storage.sh format -t Ol32dbtpRTKtUJX2KKpRHQ -c config/kraft/server.properties

bin/kafka-server-start.sh config/kraft/server.properties
```

```shell
./kafka-configs.sh --alter --bootstrap-server 192.168.0.232:9092 --entity-type topics  --entity-name error_log --add-config retention.ms=86400000

./kafka-configs.sh --describe --bootstrap-server 192.168.0.232:9092 --entity-type topics  --entity-name error_log
```

## efak

```shell
export KE_HOME=/Users/guzemin/kafka/efak-web-3.0.1

JMX_PORT=9999 ./bin/kafka-server-start.sh ./config/server.properties

./bin/ke.sh start
./bin/ke.sh stop
```



## kafka-ui

```shell
docker run -it -p 8080:8080 -d --name kafka-ui -e DYNAMIC_CONFIG_ENABLED=true provectuslabs/kafka-ui:v0.7.2
```

## 面试题

### Kafka的核心架构是什么

Kafka的核心组件包括Producer、Broker、Topic、Partition、Replica、Consumer、Consumer Group、ZooKeeper/KRaft。

消息流程: Producer -> Topic -> Partition -> Replica -> Consumer Group -> Consumer

一个Topic分为多个Partition, 每个Partition是一个有序的、不可变的追加日志, 存储在磁盘上。每个Partition有多个Replica, 其中一个是Leader负责读写, 其余是Follower负责同步。Consumer Group内的消费者共同消费一个Topic, 每个Partition只被组内一个消费者消费, 实现负载均衡。

KRaft模式(Kafka 3.3+)用Broker内部的Controller节点替代了ZooKeeper, 通过Raft协议管理元数据, 简化了部署和运维。

### Kafka为什么这么快

Kafka的高性能是多个设计共同作用的结果:

1. **顺序写磁盘**: 消息以追加方式写入日志文件, 磁盘顺序写速度接近内存随机写, 甚至更高
2. **零拷贝(sendfile)**: 消费者读取消息时, 数据直接从内核态Page Cache通过sendfile传输到网卡, 跳过用户态拷贝, 减少CPU开销
3. **Page Cache**: Kafka不维护自己的缓存, 直接利用操作系统的Page Cache, 写入即落缓存, 读取走缓存命中, 内存够大时几乎不读盘
4. **批量处理**: 生产者批量发送(batch.size), Broker批量写入, 消费者批量拉取, 减少网络和IO次数
5. **压缩**: 生产者可对batch做压缩(snappy/lz4/zstd), 减少网络传输和磁盘占用
6. **分区并行**: Topic分为多个Partition, 分布在不同Broker上, 天然水平扩展

其中顺序写、零拷贝、Page Cache是最核心的三个设计。

### Kafka如何保证消息不丢失

消息从生产到消费经过三个环节, 每个环节都可能丢失:

1. **生产者到Broker**: 配置`acks=all`, Broker的Leader和所有ISR副本都写入后才返回ack。同时开启重试`retries`, 并设置`enable.idempotence=true`防止重试导致重复
2. **Broker自身**: 配置`min.insync.replicas >= 2`, 配合`replication.factor >= 3`, 保证至少两个副本写入成功才算成功。如果min.insync.replicas=1, Leader宕机且未同步完就会丢消息
3. **Broker到消费者**: 关闭自动提交`enable.auto.commit=false`, 消费者处理完成后再手动提交offset。如果处理过程中宕机, 消息会被重新消费

acks=all + min.insync.replicas>=2 + replication.factor>=3 + 手动提交offset, 这套组合是生产环境的标准配置。

### Kafka如何保证消息的顺序性

Kafka只保证单Partition内消息有序, 不保证全局有序。保证顺序性的方案:

1. **单Partition**: 整个Topic只有一个Partition, 全局有序, 但失去并行度, 吞吐量低
2. **按Key路由**: 生产者发送消息时指定Key, 同一Key的消息路由到同一Partition, 保证同一业务实体(如同一订单)的消息有序。这是最常用的方案

注意: 如果消费失败导致重试, 仍可能破坏顺序。可以设置`max.in.flight.requests.per.connection=1`(牺牲并发)或开启幂等性(允许重试但保证不乱序)来避免。

### 如何处理消息重复消费(幂等性)

Kafka默认保证at-least-once, 消费者宕机后offset未提交的消息会被重新消费, 因此消费端必须保证幂等:

1. **唯一ID + 去重表**: 消息体携带业务唯一ID, 消费前查库判断是否已处理
2. **Redis SETNX**: 用消息ID作为key写入Redis, 设置过期时间, 写入成功才消费
3. **数据库唯一约束**: 利用唯一索引兜底, 重复插入抛异常
4. **状态机**: 业务状态有先后顺序, 已到终态的消息直接忽略

生产者侧, Kafka通过`enable.idempotence=true`和事务(transactional.id)实现Exactly Once语义, 但这是Broker侧的保证, 消费端的幂等仍需业务自己实现。

### Kafka的高可用机制是什么

Kafka的高可用建立在副本机制之上:

1. **Replica机制**: 每个Partition有多个副本, 分布在不同Broker上, 一个Leader负责读写, 其余Follower从Leader同步数据
2. **ISR(In-Sync Replicas)**: 与Leader保持同步的副本集合, 落后过多的Follower会被移出ISR。Leader选举只从ISR中选, 避免数据丢失
3. **Leader选举**: Leader宕机后, Controller从ISR中选举新Leader, 保证已提交的消息不丢
4. **Controller**: 负责管理Broker状态、Partition Leader选举, KRaft模式下Controller通过Raft保证高可用
5. **Broker宕机恢复**: Follower被选为新Leader后继续提供服务, 旧Leader恢复后成为Follower, 从新Leader追赶数据

`replication.factor=3` + `min.insync.replicas=2`是常见配置, 容忍1个节点宕机不丢数据。

### Consumer Group和Rebalance机制

Consumer Group是Kafka实现负载均衡和广播的核心机制: 同一个Group内的消费者共同消费一个Topic, 每个Partition只被组内一个消费者消费; 不同Group之间互不影响, 各自消费全量消息。

Rebalance是指Consumer Group内Partition的分配关系发生变化, 触发场景: 消费者加入/离开Group、订阅的Topic变化、Partition数量变化。

Rebalance的问题: 过程中消费者无法消费(stop the world), 频繁Rebalance会影响吞吐。优化方案: 合理配置`session.timeout.ms`和`heartbeat.interval.ms`避免误判离线; 使用Cooperative Rebalance(2.4+)减少影响范围; 确保消费逻辑不会长时间阻塞导致心跳超时。

### 分区分配策略有哪些

Consumer Group内Partition和Consumer的分配关系由`partition.assignment.strategy`决定:

1. **RangeAssignor(默认)**: 按Topic分别分配, 每个Topic的Partition按数字范围分配给消费者。可能导致先分配的消费者负载更重
2. **RoundRobinAssignor**: 所有Topic的Partition打散后轮询分配, 负载更均衡, 但要求消费者订阅的Topic相同
3. **StickyAssignor**: 尽量保持原有分配不变, Rebalance时只移动必要的Partition, 减少震荡
4. **CooperativeStickyAssignor(2.4+)**: 在Sticky基础上支持增量Rebalance, 不需要停止所有消费者

推荐使用CooperativeStickyAssignor, 兼顾均衡和稳定性。

### 生产者的acks参数有什么区别

| acks | 含义                                   | 可靠性   | 吞吐量 |
| ---- | -------------------------------------- | -------- | ------ |
| 0    | 不等待任何确认, 发送即忘               | 最低, 可能丢 | 最高   |
| 1    | Leader写入成功即返回ack                | 中, Leader宕机可能丢 | 中     |
| all(-1) | Leader和所有ISR副本都写入成功才返回ack | 最高     | 最低   |

生产环境推荐`acks=all`, 配合`min.insync.replicas>=2`和`retries`使用。注意开启`enable.idempotence=true`后, acks默认会被强制设为all。

### ISR、OSR、AR分别是什么

- **AR(Assigned Replicas)**: 一个Partition的所有副本, 包括Leader和所有Follower
- **ISR(In-Sync Replicas)**: 与Leader保持同步的副本集合, 是AR的子集, 包含Leader自身。同步延迟超过`replica.lag.time.max.ms`的Follower会被移出ISR
- **OSR(Out-of-Sync Replicas)**: 同步落后、被移出ISR的副本

AR = ISR + OSR。Leader选举和acks=all只关心ISR, OSR中的副本不参与写入确认, 追赶上后才会重新加入ISR。

### Kafka的消息存储机制

Kafka的消息以日志文件形式存储在磁盘上, 每个Partition对应一个目录:

1. **Log Segment**: Partition的日志被切分为多个Segment, 每个Segment由`.log`(数据)、`.index`(稀疏索引)、`.timeindex`(时间索引)三个文件组成。当前写满的Segment变为只读, 新建一个Segment继续写
2. **追加写**: 消息只能追加到当前活跃Segment末尾, 不可修改, 这是顺序写的基础
3. **稀疏索引**: `.index`文件记录offset到物理位置的映射, 间隔存储(默认每4KB一条), 查找时二分定位Segment, 再用索引粗定位, 最后顺序扫描
4. **消息保留**: 按`retention.ms`(时间)或`retention.bytes`(大小)清理过期Segment, 清理粒度是整个Segment, 不是单条消息

这种设计使得写入是纯追加, 读取可随机定位, 兼顾了写入吞吐和读取效率。

### Kafka如何实现Exactly Once语义

Kafka的Exactly Once通过幂等生产者 + 事务实现:

1. **幂等生产者**(`enable.idempotence=true`): 生产者分配PID(Producer ID), 每条消息携带SequenceNumber, Broker端去重, 保证单分区单会话内不重复
2. **事务**(`transactional.id`): 跨分区、跨会话的Exactly Once。生产者开启事务, 写入多个Partition后提交, 所有Partition要么全部可见要么全部不可见。配合`initTransactions`、`beginTransaction`、`commitTransaction`使用
3. **消费端**: 消费者设置`isolation.level=read_committed`, 只读取已提交的事务消息

在流处理场景(Kafka Streams), 通过consume-process-produce的事务, 可以实现端到端的Exactly Once。但注意, 这只限于Kafka内部, 如果消费端写入外部系统(如数据库), 仍需要业务侧保证幂等。

### 消息堆积如何处理

消息堆积说明消费速度跟不上生产速度, 解决思路:

1. **增加消费者**: Consumer Group内消费者数量不能超过Partition数量, 如果已达上限, 需要增加Partition(注意增加Partition会破坏Key路由的顺序性)
2. **提升单消费者吞吐**: 调大`fetch.max.bytes`和`max.poll.records`, 批量处理消息, 减少网络和IO次数
3. **异步处理**: 消费者只做接收和分发, 实际处理交给线程池异步执行, 但要注意控制并发避免OOM
4. **临时扩容**: 如果短期无法优化消费逻辑, 可以新建一个Consumer Group消费同一Topic, 将消息转发到新Topic(更多Partition), 再用更多消费者处理
5. **根因排查**: 消费者是否有慢查询、外部依赖超时、异常吞没等, 从根本上提升消费速度

### offset如何管理

消费者通过offset记录消费进度, 有两种管理方式:

1. **自动提交**(`enable.auto.commit=true`): 消费者定期(默认5s)自动提交已消费的最大offset, 简单但有风险——消息已拉取但未处理完就提交了, 宕机会丢消息; 或者处理完未提交就宕机, 重启后重复消费
2. **手动提交**(`enable.auto.commit=false`): 处理完成后调用`commitSync()`(同步, 阻塞, 可靠)或`commitAsync()`(异步, 非阻塞, 可能失败)。推荐同步提交, 性能要求高可异步+定期同步兜底

Kafka将offset存储在内部Topic `__consumer_offsets`中, 这是一个Compact Topic, 每个Consumer Group-Partition的offset只保留最新一条。

生产环境推荐手动提交, 在业务处理完成后再提交offset, 配合幂等消费保证最终一致。

### Controller的作用是什么

Controller是Kafka集群中的一个特殊Broker, 负责管理集群元数据和协调操作:

1. **Partition Leader选举**: Broker宕机后, Controller负责为该Broker上的Leader Partition选举新Leader(从ISR中选)
2. **Broker状态管理**: 监听Broker的上下线, 更新集群元数据并通知其他Broker
3. **Topic/Partition管理**: 创建/删除Topic, 增加Partition时分配副本
4. **Preferred Leader选举**: 触发Preferred Leader选举, 恢复最初的副本分配

ZooKeeper模式下Controller是从所有Broker中选举一个, 依赖ZK; KRaft模式下Controller是一组固定节点, 通过Raft协议选举Leader, 不依赖外部组件, 元数据以日志形式存储在Kafka内部Topic中。

### Kafka和RabbitMQ的区别

| 维度       | Kafka                                  | RabbitMQ                          |
| ---------- | -------------------------------------- | --------------------------------- |
| 设计目标   | 日志流, 高吞吐                         | 消息路由, 灵活投递                |
| 模型       | Partition + Offset                     | AMQP, Exchange + Queue            |
| 吞吐量     | 百万级                                 | 万级                              |
| 延迟       | 毫秒级                                 | 微秒级                            |
| 消息保留   | 按策略保留, 可回放                     | 消费即删除                        |
| 顺序保证   | 单Partition内有序                      | 单队列FIFO                        |
| 适用场景   | 日志收集、流处理、事件溯源             | 业务消息、任务队列、RPC           |
| 路由能力   | 弱(仅按Partition)                      | 强(多种交换机)                    |
| 运维复杂度 | 较高(依赖ZooKeeper/KRaft)              | 中等                              |

选型: 需要高吞吐、消息回放、流处理选Kafka; 需要灵活路由、低延迟、消息优先级选RabbitMQ。

### 如何选择分区数量

分区数量影响并行度和吞吐, 但不是越多越好:

1. **太少**: 并行度不够, 消费者数量受限于分区数, 吞吐上不去
2. **太多**: 增加Broker内存开销(每个Partition都有索引文件)、增加Leader选举和Rebalance开销、降低可用性(单Broker宕机影响更多Partition)、增加端到端延迟

经验值: 单个Partition的吞吐通常在10MB/s左右, 根据预期吞吐量反推分区数。例如目标100MB/s, 至少10个Partition。同时考虑未来增长, 适当冗余。但单Broker分区数建议不超过4000, 集群总分区数不超过20万。

增加分区不可逆, 建议初始设置一个合理值, 后续按需增加。
