---
title: Flink 实战笔记
subtitle: Savepoint、Checkpoint 原理与常用操作
date: 2026-06-07T12:06:27+08:00
slug: 836da93
draft: false
author:
  name:
  link:
  email:
  avatar:
description: Flink Savepoint 与 Checkpoint 底层机制对比及常用 CLI 操作速查
keywords:
  - flink
  - savepoint
  - checkpoint
license:
comment: false
weight: 0
tags:
  -
categories:
  -
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

```shell
./bin/start-cluster.sh

./bin/stop-cluster.sh
```

## Savepoint

手动触发 savepoint，用于作业升级/迁移时保留状态快照：

```shell
# 触发 savepoint
./bin/flink savepoint <jobId> [/tmp/savepoints]

# 后台模式（大状态时避免客户端超时）
./bin/flink savepoint -d <jobId> /tmp/savepoints

# 删除 savepoint
./bin/flink savepoint -d /tmp/savepoints/savepoint-xxx

# 从 savepoint 恢复作业
./bin/flink run -s /tmp/savepoints/savepoint-xxx ./my-job.jar

# 跳过无法恢复的状态（删过算子时用）
./bin/flink run -s /tmp/savepoints/savepoint-xxx --allowNonRestoredState ./my-job.jar

# 优雅停止作业并保存 savepoint
./bin/flink stop --savepointPath /tmp/savepoints <jobId>
```

## Checkpoint

自动或手动触发的容错快照，由 Flink 自动管理：

```shell
# 手动触发 checkpoint
./bin/flink checkpoint <jobId>

# 触发全量 checkpoint（即使配置了增量）
./bin/flink checkpoint <jobId> --full
```

在代码或配置中开启自动 checkpoint：

```yaml
# flink-conf.yaml
execution.checkpointing.interval: 60s
execution.checkpointing.mode: EXACTLY_ONCE
execution.checkpointing.timeout: 10min
execution.checkpointing.min-pause: 30s
execution.checkpointing.max-concurrent-checkpoints: 1
```

## Savepoint 与 Checkpoint 底层机制对比

### 底层快照机制：完全一致

两者共享同一套基于 Chandy-Lamport 算法的 Barrier 分布式快照流程。`CheckpointCoordinator` 负责触发和协调，向 Source 注入 `CheckpointBarrier`，Barrier 随数据流向下游传播。每个算子收到 Barrier 后执行本地状态快照，完成后向下游转发 Barrier。所有算子确认后汇总为 `CompletedCheckpoint`。

源码证据：`triggerSavepoint()` 内部直接调用 `triggerCheckpoint()`，唯一区别是传入的 `CheckpointProperties` 不同。调用链在 `startTriggeringCheckpoint()` 处完全汇合：

```
triggerSavepoint()                              triggerCheckpoint()
       │                                               │
       ▼                                               ▼
triggerSavepointInternal()              triggerCheckpointFromCheckpointThread()
       │                                               │
       └──────────────┬────────────────────────────────┘
                      ▼
        triggerCheckpointFromCheckpointThread(props, targetLocation, isPeriodic)
                      │
                      ▼
        startTriggeringCheckpoint(request)  ← 统一核心入口
```

### CheckpointProperties：属性差异的核心

`CheckpointProperties` 是区分两者的核心数据结构。Savepoint 的所有 `discard*` 标志均为 `false`，不会被系统自动清理；Checkpoint 按策略自动清理。

| 属性 | Savepoint | Checkpoint (NEVER_RETAIN) | Checkpoint (RETAIN_ON_CANCELLATION) |
|------|:---------:|:------------------------:|:----------------------------------:|
| discardSubsumed | 保留 | 丢弃 | 丢弃 |
| discardFinished | 保留 | 丢弃 | 丢弃 |
| discardCancelled | 保留 | 丢弃 | 保留 |
| discardFailed | 保留 | 丢弃 | 保留 |
| discardSuspended | 保留 | 丢弃 | 保留 |

### 存储格式差异

这是快照流程中唯一出现分支的地方。Checkpoint 直接使用状态后端原始格式（如 RocksDB SST 文件），支持增量快照和 shared state。

Savepoint 支持两种格式：**标准格式（Canonical）** 将状态重新序列化为统一格式，实现跨后端可移植性；**原生格式（Native，Flink 1.15+）** 直接拷贝后端原生数据，速度更快但不可跨后端恢复。无论哪种格式，Savepoint 的文件均为 exclusive state，确保自包含、可移动。

### 功能差异速查

| 维度 | Checkpoint | Savepoint |
|------|-----------|-----------|
| 底层快照机制 | Barrier 快照 | 完全相同 |
| 元数据结构 | CompletedCheckpoint | 完全相同 |
| 触发方式 | 自动周期性触发 | 手动触发（CLI/API） |
| 生命周期 | 系统自动管理（按策略清理） | 用户手动管理（永不自动清理） |
| 增量支持 | 支持（RocksDB） | 不支持，始终全量 |
| 非对齐模式 | 支持（Flink 1.11+） | 不支持 |
| 跨状态后端恢复 | 不支持 | 支持（标准格式） |
| 自包含可移动 | 否 | 是 |
| State Processor API | 不支持写 | 支持读写修改 |

**一句话总结**：底层完全一样，差异仅在 CheckpointProperties 配置导致的不同生命周期管理策略和存储格式选择。

## SQL Client

```shell
# 启动 SQL Client（嵌入式模式）
./bin/sql-client.sh embedded

# 指定初始化 SQL 文件
./bin/sql-client.sh embedded -i init.sql

# 指定配置
./bin/sql-client.sh embedded -d conf.yaml
```

SQL Client 内置命令：

```sql
-- 查看所有表
SHOW TABLES;

-- 查看所有数据库
SHOW DATABASES;

-- 创建数据库
CREATE DATABASE mydb;
USE mydb;

-- 创建表
CREATE TABLE orders (
  id INT,
  amount DECIMAL(10, 2),
  product STRING,
  order_time TIMESTAMP(3),
  WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
) WITH (
  'connector' = 'datagen',
  'rows-per-second' = '1'
);

-- 查看表结构
DESC orders;

-- 查询
SELECT * FROM orders LIMIT 10;

-- 插入
INSERT INTO orders_sink SELECT * FROM orders;

-- 创建视图
CREATE VIEW orders_view AS SELECT product, SUM(amount) AS total FROM orders GROUP BY product;

-- 创建 Catalog
CREATE CATALOG my_catalog WITH ('type' = 'generic_in_memory');
USE CATALOG my_catalog;

-- 查看函数
SHOW FUNCTIONS;

-- 查看 Catalog
SHOW CATALOGS;

-- 查看当前执行计划
EXPLAIN SELECT * FROM orders;

-- 删除表/视图
DROP TABLE orders;
DROP VIEW orders_view;

-- 退出
QUIT;
```

## 连接 MySQL

### 前置准备

下载 JAR 包放入 `lib/` 目录：

- `flink-connector-jdbc-3.2.0-1.19.jar`
- `mysql-connector-j-8.0.33.jar`

### SQL Client 示例

```sql
-- 创建 MySQL源表（读取）
CREATE TABLE mysql_source (
  id INT,
  name STRING,
  age INT,
  PRIMARY KEY (id) NOT ENFORCED
) WITH (
  'connector' = 'jdbc',
  'url' = 'jdbc:mysql://localhost:3306/mydb',
  'table-name' = 'users',
  'username' = 'root',
  'password' = '123456',
  'scan.fetch-size' = '1000',
  'scan.auto-commit' = 'true'
);

-- 创建 MySQL目标表（写入）
CREATE TABLE mysql_sink (
  id INT,
  name STRING,
  age INT,
  PRIMARY KEY (id) NOT ENFORCED
) WITH (
  'connector' = 'jdbc',
  'url' = 'jdbc:mysql://localhost:3306/mydb',
  'table-name' = 'users_copy',
  'username' = 'root',
  'password' = '123456'
);

-- 读取并写入
INSERT INTO mysql_sink SELECT * FROM mysql_source;
```

### DataStream API 示例

```java
// Maven 依赖
// org.apache.flink:flink-connector-jdbc:3.2.0-1.19
// com.mysql:mysql-connector-j:8.0.33

import org.apache.flink.connector.jdbc.JdbcConnectionOptions;
import org.apache.flink.connector.jdbc.JdbcExecutionOptions;
import org.apache.flink.connector.jdbc.JdbcSink;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Tuple3<Integer, String, Integer>> source = env.fromElements(
    Tuple3.of(1, "Alice", 25),
    Tuple3.of(2, "Bob", 30)
);

JdbcConnectionOptions jdbcOptions = new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
    .withUrl("jdbc:mysql://localhost:3306/mydb")
    .withDriverName("com.mysql.cj.jdbc.Driver")
    .withUsername("root")
    .withPassword("123456")
    .build();

source.addSink(JdbcSink.sink(
    "INSERT INTO users (id, name, age) VALUES (?, ?, ?) ON DUPLICATE KEY UPDATE name=VALUES(name), age=VALUES(age)",
    (ps, t) -> {
        ps.setInt(1, t.f0);
        ps.setString(2, t.f1);
        ps.setInt(3, t.f2);
    },
    JdbcExecutionOptions.builder()
        .withBatchSize(1000)
        .withBatchIntervalMs(5000)
        .withMaxRetries(3)
        .build(),
    jdbcOptions
));

env.execute("Flink MySQL Sink");
```