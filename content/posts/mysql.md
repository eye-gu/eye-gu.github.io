---
title: Mysql
subtitle:
date: 2024-10-30T14:06:52+08:00
slug: 64667e3
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
  - database
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

```shell
podman run -d --name mysql \
    -e MYSQL_ROOT_PASSWORD=12345678 \
    -e TZ=Asia/Shanghai \
    -v /Users/guzemin/docker/mysql/data:/bitnami/mysql/data:U \
    -v /Users/guzemin/docker/mysql/my.cnf:/opt/bitnami/mysql/conf/my.cnf \
    -p 3306:3306 \
    bitnami/mysql:8.2.0
```

### vector

> MySQL 社区版仅支持 VECTOR 类型存储，DISTANCE 函数和向量索引仅限 HeatWave/MySQL AI。
> MariaDB 11.7+ 原生支持向量索引（HNSW）和距离函数。

```shell
docker run -d --name mariadb \
    -e MARIADB_ROOT_PASSWORD=12345678 \
    -e TZ=Asia/Shanghai \
    -v /Users/guzemin/docker/mariadb/data:/var/lib/mysql:Z \
    -p 3306:3306 \
    docker.m.daocloud.io/mariadb:11.8
```

## 导入导出

```shell
mysqldump -h 127.0.0.1 -u root -p123456 product reptile_product > /bitnami/mysql/data/reptile_product.sql

mysql -h localhost -u sorcara -psorcara -D product
source /home/sorcara/data/reptile_product.sql
```

## 面试题

### InnoDB和MyISAM的区别

| 维度     | InnoDB                    | MyISAM                  |
| -------- | ------------------------- | ----------------------- |
| 事务     | 支持                      | 不支持                  |
| 锁粒度   | 行锁(默认)+表锁           | 表锁                    |
| 外键     | 支持                      | 不支持                  |
| 索引结构 | 聚簇索引, 数据和主键索引存在一起 | 非聚簇索引, 数据和索引分离 |
| 崩溃恢复 | 支持(redo log)            | 不支持                  |
| 全文索引 | 5.6+支持                  | 支持                    |
| 适用场景 | OLTP, 事务, 高并发        | 只读为主, 统计报表      |

MySQL 5.5之后默认存储引擎是InnoDB。MyISAM不支持事务和行锁, 写操作会锁整张表, 并发性能差, 且崩溃后数据容易损坏, 目前已不推荐使用。

### MySQL为什么用B+树作为索引结构

常见的索引数据结构有哈希表、二叉搜索树、B树、B+树, MySQL选择B+树的原因:

哈希表: 等值查询O(1)很快, 但不支持范围查询和排序, 不适合数据库场景。

二叉搜索树: 极端情况下退化为链表, 查询O(n)。平衡二叉树(AVL/红黑树)虽然保证O(logN), 但树高随数据量增长较快, 磁盘IO次数多。

B树: 每个节点既存索引又存数据, 单节点能存的key数量有限, 树仍然较高。且范围查询需要中序遍历, 效率低。

B+树的非叶子节点只存索引不存数据, 单节点能存更多key, 树更矮(通常3-4层即可支撑千万级数据)。所有数据都在叶子节点, 且叶子节点通过双向链表连接, 范围查询和排序只需遍历链表, 非常高效。InnoDB默认页大小16KB, 单个页能存约1170个索引项(假设主键8字节+指针6字节), 三层B+树可存储约2000万条记录(1170 * 1170 * 16)。

### 聚簇索引和非聚簇索引的区别

聚簇索引(Clustered Index): 叶子节点存储的是完整行数据, 数据物理上按主键顺序存储, 一张表只能有一个聚簇索引。InnoDB的主键索引就是聚簇索引。

非聚簇索引(Secondary Index): 叶子节点存储的是主键值而非完整行数据, 通过非聚簇索引查找时需要先找到主键, 再回表到聚簇索引中找完整数据, 这个过程叫回表。一张表可以有多个非聚簇索引。

MyISAM的数据和索引分离, 主键索引和二级索引都是非聚簇的, 叶子节点存储的是数据行的物理地址。

InnoDB推荐使用自增主键, 因为聚簇索引按主键顺序存储, 自增主键的插入是顺序追加, 不会导致页分裂; 如果用UUID等随机主键, 插入位置随机, 频繁页分裂导致性能下降和空间浪费。

### 什么是回表和覆盖索引

回表: 通过非聚簇索引(二级索引)查到主键值后, 再去聚簇索引中查找完整行数据的过程。一次查询走了两棵B+树。

覆盖索引: 查询的字段全部包含在索引中, 不需要回表。例如表有联合索引 `(name, age)`, 查询 `SELECT name, age FROM t WHERE name = '张三'`, 索引中已包含 name 和 age, 直接从索引返回, 无需回表。

覆盖索引能减少IO次数, 显著提升查询性能。实际开发中, 对于高频查询, 可以通过建立合适的联合索引来覆盖查询字段, 避免回表。explain中如果Extra列显示 `Using index`, 说明用到了覆盖索引。

### 索引失效的场景有哪些

1. **违反最左前缀原则**: 联合索引 `(a, b, c)` 中, 查询条件没有a, 或跳过a直接用b/c, 索引失效
2. **索引列上做运算或函数**: `WHERE age + 1 = 20` 或 `WHERE YEAR(create_time) = 2024`, 索引失效, 应改为 `WHERE age = 19` 或 `WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'`
3. **类型隐式转换**: 字符串列 `WHERE phone = 13800000000`(数字), MySQL会做隐式类型转换, 等价于对列加了函数, 索引失效。应保证查询条件类型与列类型一致
4. **LIKE以通配符开头**: `WHERE name LIKE '%张'` 索引失效, `LIKE '张%'` 可以走索引(前缀匹配)
5. **OR连接的条件**: `WHERE a = 1 OR b = 2`, 如果b没有索引, 整个查询会走全表扫描。需要a和b都有索引才能走index merge
6. **NOT IN / NOT EXISTS / !=**: 这些操作通常不走索引(优化器认为扫描行数太多不如全表扫描), 但 `IN` 可以走索引
7. **IS NULL / IS NOT NULL**: 通常不走索引, 但如果NULL值比例很小, 优化器可能选择走索引
8. **ORDER BY非索引列**: 排序字段没有索引时会触发filesort
9. **优化器认为全表扫描更快**: 当索引扫描的行数超过全表行数的某个比例(约30%), 优化器会放弃索引走全表扫描。这不是真正的"失效", 而是优化器的成本估算

### 联合索引的最左前缀原则

联合索引 `(a, b, c)` 的B+树按a排序, a相同按b排序, b相同按c排序。因此查询条件必须从最左列开始才能利用索引:

- `WHERE a = 1`: 走索引
- `WHERE a = 1 AND b = 2`: 走索引
- `WHERE a = 1 AND b = 2 AND c = 3`: 走索引
- `WHERE b = 2`: 不走索引(缺少最左列a)
- `WHERE a = 1 AND c = 3`: 只能用到a的索引部分, c无法利用索引(中间跳过了b)
- `WHERE a = 1 AND b > 2 AND c = 3`: a和b走索引, 但b是范围查询, c无法走索引(范围查询右侧的列无法利用索引)

注意MySQL 8.0引入了索引跳跃扫描(Index Skip Scan), 在某些场景下即使没有最左前缀也能利用索引, 但前提是前导列的distinct值很少。

### MySQL的事务隔离级别有哪些

| 隔离级别         | 脏读 | 不可重复读 | 幻读 |
| ---------------- | ---- | ---------- | ---- |
| READ UNCOMMITTED | 有   | 有         | 有   |
| READ COMMITTED   | 无   | 有         | 有   |
| REPEATABLE READ  | 无   | 无         | 有*  |
| SERIALIZABLE     | 无   | 无         | 无   |

脏读: 事务A读到了事务B未提交的数据, B回滚后A读到的是脏数据。

不可重复读: 事务A两次读取同一行, 中间事务B修改并提交, A两次读到不同结果。

幻读: 事务A两次执行同一查询, 中间事务B插入并提交, A第二次多出了行, 像幻觉一样。

InnoDB默认隔离级别是REPEATABLE READ(RR), 但InnoDB在RR级别下通过MVCC + 临键锁(Next-Key Lock)解决了幻读问题, 这是InnoDB对SQL标准的增强。Oracle默认是READ COMMITTED(RC)。

RC级别下每次SELECT都会生成新的Read View, 能读到已提交的最新数据; RR级别下事务第一次SELECT时生成Read View, 后续复用, 保证可重复读。

### MVCC的原理是什么

MVCC(Multi-Version Concurrency Control, 多版本并发控制)是InnoDB实现RR和RC隔离级别的核心机制, 通过维护数据的多个版本, 让读操作不加锁也能读到一致性快照, 实现读写不冲突。

MVCC依赖三个组件:

1. **隐藏字段**: 每行数据有两个隐藏字段, `trx_id`(最近修改该行的事务ID)和`roll_pointer`(指向undo log中该行的上一个版本)
2. **undo log**: 数据每次修改都会在undo log中记录修改前的版本, 通过roll_pointer串联成版本链
3. **Read View**: 事务执行SELECT时生成的一致性视图, 包含当前活跃(未提交)事务ID列表

Read View判断版本可见性的规则: 如果某版本的trx_id小于Read View中最小活跃事务ID, 说明该版本在当前事务之前已提交, 可见; 如果trx_id大于Read View中最大事务ID, 说明该版本在Read View生成后才产生, 不可见; 如果trx_id在活跃事务列表中, 说明该版本对应的事务还未提交, 不可见, 顺着undo log版本链找到下一个可见版本。

RC隔离级别下每次SELECT都生成新Read View, 因此能读到最新已提交数据; RR隔离级别下整个事务复用第一次SELECT的Read View, 保证可重复读。

### MySQL有哪些锁

按粒度分:

1. **全局锁**: `FLUSH TABLES WITH READ LOCK`, 整库只读, 用于备份
2. **表锁**: `LOCK TABLES t READ/WRITE`, 显式加锁。MDL(元数据锁)在DDL/DML时自动加, 防止DDL和DML冲突
3. **行锁**: InnoDB特有, 分为记录锁(Record Lock)、间隙锁(Gap Lock)、临键锁(Next-Key Lock)

按类型分:

1. **共享锁(S锁, 读锁)**: `SELECT ... LOCK IN SHARE MODE`, 多个事务可同时持有
2. **排他锁(X锁, 写锁)**: `UPDATE/DELETE/SELECT ... FOR UPDATE`, 互斥, 与S锁也互斥

InnoDB行锁是加在索引上的, 不是加在数据行上。如果查询没有走索引, 会对所有行加锁, 退化为表锁。这是生产环境中锁冲突的常见原因。

### 什么是间隙锁和临键锁

记录锁(Record Lock): 锁定索引上的一条记录。

间隙锁(Gap Lock): 锁定索引记录之间的间隙, 但不包含记录本身。目的是防止其他事务在这个间隙中插入数据, 解决幻读。例如索引上有记录10, 15, 20, 间隙锁可以锁定(10, 15)这个区间, 阻止其他事务插入11, 12等。

临键锁(Next-Key Lock): 记录锁 + 间隙锁的组合, 锁定一个左开右闭区间 (10, 15]。InnoDB在RR隔离级别下默认使用临键锁, 防止幻读。

间隙锁只在RR隔离级别下存在, RC隔离级别下没有间隙锁。间隙锁之间不冲突, 两个事务可以同时持有同一间隙的间隙锁, 但任何一个事务都无法在该间隙插入数据。

注意: 唯一索引等值查询命中记录时, 临键锁会退化为记录锁(不需要间隙锁, 因为唯一索引已保证不会有重复值插入)。非唯一索引等值查询, 或范围查询, 都会使用临键锁。

### redo log、undo log、binlog的区别

| 维度     | redo log                      | undo log                  | binlog                       |
| -------- | ----------------------------- | ------------------------- | ---------------------------- |
| 层级     | InnoDB引擎层                  | InnoDB引擎层              | MySQL Server层               |
| 作用     | 崩溃恢复, 保证持久性          | 事务回滚, MVCC            | 主从复制, 数据恢复           |
| 内容     | 物理日志, 记录页的物理修改    | 逻辑日志, 记录修改前的数据 | 逻辑日志, 记录SQL或行变更    |
| 写入方式 | 循环写, 空间固定              | 随机写, 存在回滚段中      | 追加写, 文件切换             |
| 是否必须 | 是                            | 是                        | 否(主从架构下需要)           |

redo log是WAL(Write-Ahead Logging)的核心, 事务提交时先写redo log再写数据页, 保证宕机后能恢复已提交的数据。redo log是循环写的, 通过`innodb_flush_log_at_trx_commit`控制刷盘策略。

binlog是MySQL Server层的日志, 所有引擎都有, 用于主从复制和基于时间点的恢复。通过`sync_binlog`控制刷盘策略。

两阶段提交: InnoDB写redo log(prepare状态) -> 写binlog -> 写redo log(commit状态), 保证redo log和binlog的一致性。如果binlog写完前宕机, 恢复时回滚事务; 如果binlog写完后宕机, 恢复时提交事务。

### MySQL的主从复制原理

主从复制基于binlog实现, 流程:

1. 主库执行事务, 写入binlog
2. 从库的IO线程连接主库, 请求读取binlog
3. 主库的Dump线程将binlog发送给从库
4. 从库IO线程将binlog写入relay log(中继日志)
5. 从库SQL线程读取relay log, 重放SQL, 更新数据

复制方式有三种:

1. **异步复制**: 主库写完binlog即返回, 不等从库。默认模式, 主库宕机可能丢数据
2. **半同步复制**: 主库写完binlog后, 至少等待一个从库收到binlog并ack后才返回。平衡性能和可靠性
3. **组复制(MGR)**: 基于Paxos协议的强一致复制, 多主写入, 自动故障转移

复制延迟是主从架构的核心问题, 原因包括: 从库单线程重放(5.7+支持并行复制)、大事务、网络延迟、从库性能不足。排查可通过`SHOW SLAVE STATUS`的`Seconds_Behind_Master`。

### explain各字段的含义

```sql
EXPLAIN SELECT * FROM t WHERE name = '张三';
```

关键字段:

- **id**: 查询序号, 越大越先执行。相同时从上往下执行
- **select_type**: 查询类型, SIMPLE(简单查询)、PRIMARY(最外层)、SUBQUERY(子查询)、DERIVED(派生表)等
- **table**: 涉及的表名
- **type**: 访问类型, 性能从好到差: `system > const > eq_ref > ref > range > index > ALL`。`const`/`eq_ref`最优, `ALL`是全表扫描, 需要优化。通常要求至少达到`range`级别
- **possible_keys**: 可能用到的索引
- **key**: 实际使用的索引, NULL表示没走索引
- **key_len**: 索引使用的字节数, 判断联合索引用了几个字段
- **ref**: 索引比较的列或常量
- **rows**: 优化器估算的扫描行数, 越小越好
- **filtered**: 过滤后剩余行数的百分比
- **Extra**: 额外信息, `Using index`(覆盖索引)、`Using where`(服务层过滤)、`Using temporary`(临时表, 需优化)、`Using filesort`(额外排序, 需优化)

### 如何排查和优化慢SQL

1. **开启慢查询日志**: `slow_query_log=ON`, `long_query_time=1`(超过1秒记录)
2. **分析慢日志**: `mysqldumpslow -s t -t 10 slow.log` 按时间排序取前10条
3. **explain分析**: 查看执行计划, 重点关注type、key、rows、Extra
4. **优化方向**:
   - 是否走索引, 没走索引就加索引或调整查询条件
   - 是否回表, 能否通过覆盖索引优化
   - 是否有filesort或temporary, 优化ORDER BY和GROUP BY
   - 大分页用游标分页(WHERE id > last_id LIMIT 10)代替LIMIT offset
   - 避免SELECT *, 只查需要的字段
   - 大事务拆小, 减少锁持有时间
5. **SQL改写**: 子查询改JOIN, OR改UNION, NOT IN改LEFT JOIN ... IS NULL
6. **数据层面**: 历史数据归档, 减少单表数据量; 冷热分离

### count(*)、count(1)、count(列)的区别

`count(*)`: 统计所有行, 包括NULL行。InnoDB专门优化过, 不取值直接累加, 性能最优。

`count(1)`: 统计所有行, 对每行返回常量1再统计, 性能和count(*)几乎相同。

`count(列)`: 统计该列非NULL的行数, 需要读取每行的该列值判断是否为NULL, 如果列上没有索引, 性能不如count(*)。

InnoDB对count(*)做了优化, 优化器会选择最小的索引来扫描计数, 但仍然是O(N)的扫描。如果需要精确的行数, 只能count(*)。如果允许近似值, 可以用 `SHOW TABLE STATUS` 的 `Rows` 字段或 `information_schema.tables.table_rows`, 但这些是估算值。

### MySQL的WAL机制

WAL(Write-Ahead Logging, 预写日志)是InnoDB保证持久性的核心机制: 先写日志, 再写数据页。

事务提交时, InnoDB不会立即将数据页刷盘(数据页是随机写, 性能差), 而是先将修改记录到redo log(顺序写, 性能好), 再返回成功。数据页由后台线程异步刷盘(Buffer Pool -> 磁盘)。

如果宕机, 数据页可能还没刷盘, 但redo log已持久化, 重启后通过redo log重放, 恢复已提交的数据。

redo log的刷盘策略由 `innodb_flush_log_at_trx_commit` 控制:
- 0: 每秒刷盘一次, 宕机最多丢1秒数据
- 1(默认): 每次提交都刷盘, 最安全, 性能最差
- 2: 每次提交写到OS cache, 每秒刷盘一次, 宕机且OS崩溃才丢数据

生产环境推荐设为1, 保证不丢数据。

### MySQL如何实现高可用

1. **主从切换**: 主库故障时将从库提升为主库。早期用MHA, 现在常用Orchestrator或云厂商托管方案
2. **MGR(Group Replication)**: 基于Paxos的强一致集群, 自动故障检测和转移, 支持多主写入, 推荐单主模式
3. **MHA**: 第三方高可用方案, 监控主库, 故障时从多个从库中选出数据最新的提升为主库, 已逐渐被MGR替代
4. **读写分离**: 写走主库, 读走从库, 通过ProxySQL/MySQL Router等中间件实现
5. **分库分表**: 数据量大时, 通过ShardingSphere/Vitess等中间件水平拆分, 每个分片独立高可用

高可用的核心是数据不丢和服务不中断。数据不丢靠半同步复制或MGR的多数派确认; 服务不中断靠自动故障检测和切换。通常前面挂VIP或DNS, 切换时改变VIP/DNS指向。

### 分库分表的方案

垂直拆分:

1. **垂直分库**: 按业务拆分, 如用户库、订单库、商品库, 解决单库表太多的问题
2. **垂直分表**: 将宽表拆分为多个表, 如将不常用的大字段单独拆出, 减少IO

水平拆分:

1. **水平分库**: 同一张表的数据按规则分散到不同库, 如按用户ID取模
2. **水平分表**: 同一张表的数据按规则分散到同库的多张表

分片策略:

1. **范围分片**: 按ID或时间范围分片, 如0-1000万一个分片。优点是扩容方便, 缺点是热点问题(最新数据集中在一个分片)
2. **哈希分片**: 按key取模, 数据分布均匀, 但扩容需要迁移大量数据(一致性哈希可缓解)
3. **一致性哈希**: 扩缩容只影响相邻节点, 但实现复杂

分库分表引入的问题: 跨库JOIN无法做, 需要在应用层组装; 分布式事务; 跨库分页排序; 全局唯一ID(用雪花算法)。因此分库分表是最后的手段, 优先通过索引优化、读写分离、数据归档等手段解决性能问题。

### 如何定位死锁

```sql
-- 查看最近一次死锁信息
SHOW ENGINE INNODB STATUS;

-- 开启死锁日志
SET GLOBAL innodb_print_all_deadlocks = ON;
```

`SHOW ENGINE INNODB STATUS` 输出的 `LATEST DETECTED DEADLOCK` 部分会记录死锁的两个事务、各自持有和等待的锁信息。

死锁的常见场景: 两个事务以不同顺序加锁多张表或多行记录, 形成循环等待。例如事务A先锁行1再锁行2, 事务B先锁行2再锁行1。

避免死锁的方法: 多表操作时保持一致的加锁顺序; 大事务拆小, 减少锁持有时间; 在事务中先查再改的场景使用SELECT ... FOR UPDATE代替先SELECT再UPDATE; 合理使用索引避免锁升级; 设置合理的锁超时时间 `innodb_lock_wait_timeout`。

InnoDB默认开启死锁检测(`innodb_deadlock_detect=ON`), 检测到死锁后主动回滚代价较小的事务。高并发场景下死锁检测本身有性能开销, 可考虑关闭检测并依赖锁超时。

### 当前读和快照读的区别

**快照读(一致性非锁定读)**: 普通的 `SELECT` 语句(不加锁), 通过 MVCC 读取历史版本, 不加锁。RC 级别每次 SELECT 生成新 Read View, RR 级别复用事务第一次的 Read View。

**当前读(锁定读)**: 读取的是最新已提交数据, 并加锁。以下语句触发当前读:

```sql
SELECT ... FOR UPDATE;      -- 加X锁
SELECT ... LOCK IN SHARE MODE; -- 加S锁
UPDATE / DELETE / INSERT;   -- 加X锁
```

在 RR 级别下, 快照读通过 MVCC 避免幻读, 当前读通过 Next-Key Lock 避免幻读。注意: 当前读总是读最新数据, 即使事务前面有快照读, 当前读也不会受 Read View 影响。

```sql
-- 事务A                              -- 事务B
BEGIN;
SELECT * FROM t WHERE id = 1; -- 快照读, 假设name='a'
                                      BEGIN;
                                      UPDATE t SET name='b' WHERE id = 1;
                                      COMMIT;
SELECT * FROM t WHERE id = 1; -- 快照读, 仍是name='a' (RR)
SELECT * FROM t WHERE id = 1 FOR UPDATE; -- 当前读, name='b'
```

### Buffer Pool的工作原理

Buffer Pool 是 InnoDB 的内存缓存区, 所有读写操作都先经过 Buffer Pool, 是性能的核心。

**结构**:

- **数据页(16KB)**: 缓存表数据和索引数据
- **Change Buffer**: 缓存二级索引的修改(对非唯一索引页的写优化)
- **Adaptive Hash Index(AHI)**: 自动对热点索引页建立哈希索引, 等值查询O(1)
- **Log Buffer**: 缓存 redo log

**管理算法**: 基于 LRU 变体, 将链表分为 **young 区(5/8)** 和 **old 区(3/8)**:

1. 新读入的页放入 old 区头部
2. old 区的页再次被访问且间隔超过 `innodb_old_blocks_time`(默认1秒), 才提升到 young 区头部
3. 全表扫描时大量页只会停留在 old 区, 不会冲刷 young 区的热数据

```sql
-- 查看Buffer Pool状态
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

生产环境建议 `innodb_buffer_pool_size` 设为物理内存的 60%-80%。

### Change Buffer是什么

Change Buffer 是 Buffer Pool 的一部分, 用于优化**二级非唯一索引**的写性能。

对二级索引的修改, 如果目标页不在 Buffer Pool 中, 不会立即从磁盘读取, 而是先在 Change Buffer 中记录修改, 等到该页被读取时再 merge。

```sql
-- 假设name列有非唯一索引
UPDATE t SET name = 'b' WHERE id = 1;
-- 1. 聚簇索引(主键)页在Buffer Pool中, 直接修改
-- 2. 二级索引页可能不在Buffer Pool中, 先缓存到Change Buffer
-- 3. 后续SELECT用到该索引页时, 读取磁盘页 + merge Change Buffer
```

**为什么唯一索引不能用 Change Buffer**: 唯一索引插入时必须检查唯一性约束, 需要读取磁盘页判断是否冲突, 无法延迟。所以唯一索引的写性能不如普通索引。

### 唯一索引和普通索引的性能差异

| 维度 | 唯一索引 | 普通索引 |
|------|---------|---------|
| 查找 | 找到第一条即返回 | 找到第一条还需找下一条确认(B+树有序) |
| 插入 | 需读取磁盘页检查唯一性 | 可用 Change Buffer 延迟写 |
| Change Buffer | 不支持 | 支持 |
| 业务保证 | 数据库保证唯一 | 应用层保证唯一 |

**结论**: 如果业务层已保证唯一(如用户表手机号注册前查重), 用普通索引性能更好。如果需要数据库强保证, 用唯一索引。

### 索引下推(Index Condition Pushdown, ICP)

MySQL 5.6 引入, 在**存储引擎层**过滤, 减少回表次数。

```sql
-- 联合索引 (name, age)
SELECT * FROM t WHERE name LIKE '张%' AND age > 18;
```

**无 ICP(5.6之前)**:
1. 存储引擎通过索引找到所有 `name LIKE '张%'` 的主键
2. 回表取完整行
3. Server 层过滤 `age > 18`

**有 ICP(5.6+)**:
1. 存储引擎通过索引找到 `name LIKE '张%'` 的记录
2. **在存储引擎层直接判断 `age > 18`** (age 在索引中)
3. 只有满足条件的才回表

ICP 适用于联合索引中, WHERE 条件包含部分索引列但不能完全走索引的场景。explain 中 Extra 显示 `Using index condition` 表示用了 ICP。

### JOIN的执行原理和优化

MySQL 的 JOIN 采用 **Nested-Loop Join(嵌套循环连接)** 算法, 驱动表全表扫描, 被驱动表通过索引查找:

1. **Simple Nested-Loop Join**: 对驱动表每行, 扫描被驱动表全表。性能极差, MySQL 不用
2. **Index Nested-Loop Join**: 被驱动表有索引, 对驱动表每行走索引查找被驱动表。推荐
3. **Block Nested-Loop Join(BNL)**: 被驱动表无索引, 将驱动表数据放入 join_buffer, 批量匹配。减少被驱动表扫描次数

```sql
-- 驱动表选择: 优化器选择小表(结果集小的)作为驱动表
EXPLAIN SELECT * FROM t1 JOIN t2 ON t1.id = t2.t1_id WHERE t1.age > 20;
```

**MySQL 8.0 引入 Hash Join**: 替代 BNL, 先扫描驱动表构建内存哈希表, 再扫描被驱动表探测。性能优于 BNL, 不要求被驱动表有索引。

**优化建议**:
- 被驱动表的 JOIN 列加索引
- 小表驱动大表(可用 `STRAIGHT_JOIN` 强制驱动表顺序)
- 减少 JOIN 的列, 用覆盖索引避免回表
- `join_buffer_size` 适当调大, 能容纳驱动表数据时只需扫描被驱动表一次

### binlog的三种格式

| 格式 | 说明 | 优缺点 |
|------|------|--------|
| **STATEMENT** | 记录SQL语句 | 日志小, 但 `NOW()`/`UUID()` 等函数在主从不一致 |
| **ROW** | 记录每行变更前后的值 | 主从一致, 但日志大(尤其批量操作) |
| **MIXED** | 自动选择, 一般用STATEMENT, 含不确定函数时用ROW | 折中方案 |

```sql
-- 查看当前格式
SHOW VARIABLES LIKE 'binlog_format';
-- MySQL 5.7.7+ 默认ROW
```

**生产环境推荐 ROW**: 主从数据一致性最好, 配合 `binlog_row_image=MINIMAL` 减少日志量。缺点是日志体积大, 但可通过 `binlog_expire_logs_seconds` 控制保留时间。

### doublewrite(双写)机制

doublewrite 是 InnoDB 防止**页撕裂(partial page write)**的机制。

页大小 16KB, 操作系统页大小通常 4KB, 写一个 InnoDB 页需要 4 次 OS 写。如果写到一半宕机, 该页损坏, redo log 也无法恢复(redo log 是基于页的物理日志, 页损坏无法重放)。

**doublewrite 流程**:
1. 脏页先写入 doublewrite buffer(内存中 2MB)
2. doublewrite buffer 顺序写入共享表空间的 doublewrite 区域(磁盘上 2MB 连续空间)
3. 再将页写入各自的表空间文件

崩溃恢复时, 如果发现页损坏, 从 doublewrite 区域恢复完整页, 再通过 redo log 重放。

```sql
SHOW VARIABLES LIKE 'innodb_doublewrite';  -- 默认ON
```

### Undo Log版本链和Read View的详细工作流程

```
事务100: INSERT name='a'  -- 初始插入
事务200: UPDATE name='b'  -- 第一次修改
事务300: UPDATE name='c'  -- 第二次修改
```

版本链(trx_id 为修改该行的事务ID):

```
当前行: name='c', trx_id=300, roll_pointer →
  undo: name='b', trx_id=200, roll_pointer →
    undo: name='a', trx_id=100, roll_pointer → NULL
```

Read View 包含四个核心字段:

| 字段 | 含义 |
|------|------|
| `m_ids` | 生成 Read View 时当前活跃(未提交)事务ID列表 |
| `min_trx_id` | m_ids 中最小值 |
| `max_trx_id` | 系统下一个要分配的事务ID |
| `creator_trx_id` | 生成该 Read View 的事务ID |

**可见性判断规则**:

1. `trx_id == creator_trx_id`: 自己修改的, 可见
2. `trx_id < min_trx_id`: 修改该版本的事务在 Read View 之前已提交, 可见
3. `trx_id >= max_trx_id`: 修改该版本的事务在 Read View 之后才开启, 不可见
4. `min_trx_id <= trx_id < max_trx_id` 且 `trx_id in m_ids`: 事务未提交, 不可见
5. `min_trx_id <= trx_id < max_trx_id` 且 `trx_id not in m_ids`: 事务已提交, 可见

不可见时, 顺着 roll_pointer 找下一个版本, 重复判断。

### InnoDB的刷脏机制

InnoDB 有多个后台线程负责刷脏(将 Buffer Pool 中的脏页写入磁盘):

1. **Master Thread**: 主线程, 每秒和每10秒触发一次, 根据负载决定刷脏量
2. **Purge Thread**: 回收 undo log 页(事务提交后旧版本不再需要时)
3. **Page Cleaner Thread**: 专门刷脏, 减轻 Master Thread 负担(MySQL 5.7+)

**刷脏触发条件**:
- redo log 空间不足(最紧急, 会阻塞用户线程)
- Buffer Pool 空间不足(LRU 淘汰脏页)
- MySQL 正常关闭
- 后台定时刷脏

**关键参数**:
- `innodb_max_dirty_pages_pct`: 脏页比例上限(默认75%), 超过加速刷脏
- `innodb_max_dirty_pages_pct_lwm`: 脏页比例下限(默认0), 开启预刷脏
- `innodb_io_capacity`: InnoDB 后台任务每秒 IO 吞吐(建议设为磁盘 IOPS)

### 长事务的危害和排查

长事务指运行时间过长的事务, 危害:

1. **undo log 膨胀**: 事务未提交, 旧版本不能回收, undo 表空间持续增长
2. **锁占用**: 长时间持有行锁/间隙锁, 阻塞其他事务
3. **复制延迟**: 主库大事务导致从库重放慢
4. **MVCC 版本链过长**: Read View 依赖的活跃事务列表不释放, 读操作需要遍历长版本链

```sql
-- 查询超过60秒的事务
SELECT * FROM information_schema.innodb_trx
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;

-- 设置事务超时(5.7+)
SET GLOBAL innodb_kill_idle_transaction = 60;
```

**避免长事务**: 业务中不要在事务中夹杂 RPC 调用; 避免在事务中处理大文件; `SET SESSION max_execution_time = 10000`(毫秒)限制语句执行时间; 监控 `innodb_trx` 表。

### 深分页优化

```sql
-- 慢: LIMIT 1000000, 10 扫描100万行后丢弃
SELECT * FROM t ORDER BY id LIMIT 1000000, 10;

-- 优化1: 游标分页(推荐), 利用主键有序
SELECT * FROM t WHERE id > 1000000 ORDER BY id LIMIT 10;

-- 优化2: 延迟关联, 先通过覆盖索引查出主键, 再关联
SELECT t.* FROM t
INNER JOIN (SELECT id FROM t ORDER BY id LIMIT 1000000, 10) tmp
ON t.id = tmp.id;
```

游标分页最快但不支持跳页(不能直接跳到第10万页); 延迟关联适合必须用 LIMIT offset 的场景。

### ORDER BY优化

```sql
-- 联合索引 (a, b)
SELECT * FROM t WHERE a = 1 ORDER BY b;  -- 走索引, 无filesort
SELECT * FROM t WHERE a > 1 ORDER BY b;  -- 走索引范围扫描, 但仍可能filesort
SELECT * FROM t WHERE a = 1 ORDER BY b DESC; -- 走索引(B+树双向链表)
SELECT * FROM t WHERE a = 1 ORDER BY c;  -- 不走索引, filesort
```

**filesort 的两种算法**:
- **单路排序**: 取出所有字段放入 sort_buffer 排序。内存足够时用, 数据量大需临时文件
- **双路排序**: 只取出排序字段+主键排序, 排序后回表取数据。`max_length_for_sort_data` 超限时用

**优化**: 排序字段加联合索引; `sort_buffer_size` 适当调大; 只查需要的字段(减少 sort_buffer 使用); 用覆盖索引避免回表。

### Online DDL和加索引的锁影响

MySQL 5.6+ 支持 Online DDL, 加索引时尽量不阻塞业务:

```sql
-- Online DDL (5.6+默认)
ALTER TABLE t ADD INDEX idx_name(name), ALGORITHM=INPLACE, LOCK=NONE;

-- ALGORITHM:
--   COPY:    创建新表复制数据, 锁表(不推荐)
--   INPLACE: 原地修改, 不复制全表(推荐)
--   INSTANT: 元数据修改, 不动数据(8.0.12+, 仅部分操作支持)

-- LOCK:
--   EXCLUSIVE: 排他锁, 阻塞所有DML
--   SHARED:    共享锁, 阻塞写, 允许读
--   NONE:      无锁, 允许读写(目标)
```

**加索引过程(INPLACE)**:
1. **Prepare阶段**: MDL锁(EXCLUSIVE), 创建临时 frm 文件, 很短
2. **Execute阶段**: MDL降级为SHARED, 允许读写, 构建索引(最耗时)
3. **Commit阶段**: MDL升级为EXCLUSIVE, 替换旧表, 很短

大部分时间在 Execute 阶段不阻塞业务, 但 Prepare/Commit 的短暂排他锁在高并发下仍可能触发锁等待。生产环境建议用 **pt-online-schema-change** 或 **gh-ost** 工具, 基于触发器/binlog 实现零锁等待。

### MySQL主从延迟的原因和解决

**原因**:
1. 主库写 binlog 是顺序写, 从库重放也是顺序, 但从库单线程重放(5.7前)
2. 大事务: 单个事务 binlog 过大, 重放耗时长
3. 从库硬件差, 或从库有读压力消耗资源
4. 网络延迟

**解决方案**:
1. **并行复制(5.7+)**: 基于 group commit 的逻辑时钟并行, 从库多线程重放
2. **WRITESET 复制(8.0+)**: 基于行依赖关系的并行, 并行度更高
3. 大事务拆小
4. 从库关闭 binlog (`log_bin=OFF`, `log_slave_updates=OFF`)
5. 从库关闭半同步等待
6. 读写分离时, 写后立即读走主库(强制路由)
7. 关键业务用 MGR 强一致方案

### InnoDB的页结构

InnoDB 以页(默认16KB)为基本存储单位, 页结构:

| 区域 | 大小 | 说明 |
|------|------|------|
| File Header | 38B | 页号、前后页指针、页类型 |
| Page Header | 56B | 索引页元信息(槽位数、空闲位置等) |
| Infimum/Supremum | 26B | 虚拟最小/最大记录 |
| User Records | 变长 | 实际数据行, 按链表组织 |
| Free Space | 变长 | 空闲空间 |
| Page Directory | 变长 | 槽位目录, 二分查找加速 |
| File Trailer | 8B | 校验和+LSN, 保证页完整性 |

**Page Directory 的作用**: 将页内记录分组, 每组最后一条记录地址存入槽(slot)。查找时先二分定位槽, 再在组内遍历, 将页内查找从 O(n) 降到 O(log n)。

**页分裂与合并**: 页满时插入新记录触发页分裂(一半数据移到新页); 删除记录使页填充率低于 `MERGE_THRESHOLD`(默认50%)时与相邻页合并。频繁页分裂导致碎片和性能下降, 这也是推荐自增主键的原因。

### 自适应哈希索引(AHI)

InnoDB 自动监控热点查询, 如果某个索引页被频繁以等值条件访问, 会在内存中为其建立哈希索引, 将 O(log N) 的 B+树查找优化为 O(1)。

```sql
SHOW VARIABLES LIKE 'innodb_adaptive_hash_index';  -- 默认ON
SHOW STATUS LIKE 'Innodb_adaptive_hash%';
-- Innodb_adaptive_hash_hits: 命中次数
-- Innodb_adaptive_hash_total_size: 内存使用
```

AHI 由后台线程自动维护, 无法手动指定。高并发等值查询场景受益明显, 但写多读少时维护哈希表有开销, 且占用 Buffer Pool 内存。如果 AHI 成为瓶颈, 可考虑关闭。

### MySQL 8.0的重要新特性

1. **隐藏索引(Invisible Index)**: 索引对优化器不可见, 用于安全测试索引影响
   ```sql
   ALTER TABLE t ALTER INDEX idx_name INVISIBLE;
   ALTER TABLE t ALTER INDEX idx_name VISIBLE;
   ```
2. **降序索引**: 8.0真正支持降序存储B+树, `INDEX(a DESC)` 不再忽略
3. **函数索引**: 对表达式建索引
   ```sql
   CREATE INDEX idx_lower_name ON t ((LOWER(name)));
   ```
4. **Hash Join**: 替代BNL, 无索引JOIN性能大幅提升
5. **窗口函数**: `ROW_NUMBER()`, `RANK()`, `LEAD()`, `LAG()` 等
6. **WRITESET并行复制**: 从库并行度更高
7. **原子DDL**: DDL操作支持事务, 不会出现半完成状态
8. **CTE(公用表表达式)**: `WITH` 语法, 简化复杂查询
9. **角色(Role)**: 类似Oracle的权限组, 简化权限管理
