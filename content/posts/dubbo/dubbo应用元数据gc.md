---
title: Dubbo应用元数据gc
subtitle: dubbo-go 应用级元数据的定时续约与过期清理策略
date: 2026-06-26T11:42:30+08:00
slug: d443f5f
draft: false
author:
  name:
  link:
  email:
  avatar:
description: 分析 dubbo-go 应用级元数据在 remote 模式下缺少清理策略导致旧 revision 堆积的问题，以及基于定时续约与 GC 的解决方案。
keywords:
  - dubbo-go
  - metadata
  - GC
  - renew
  - 应用级元数据
license:
weight: 0
tags:
  - 
categories:
  - dubbo
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary: dubbo-go 应用级元数据定时续约与 GC 清理策略的设计与实现，对应 Issue #3355 / PR #3371。
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

# Dubbo 应用级元数据 GC

> 对应 Issue: [#3355](https://github.com/apache/dubbo-go/issues/3355) ｜ PR: [#3371](https://github.com/apache/dubbo-go/pull/3371)
> 归属版本: dubbo-go `3.3.2`

## 问题描述

### 背景

自 #2534 起，dubbo-go 对 metadata center 的使用聚焦于两类数据：应用级 metadata（`/dubbo/{application}/{revision}`）与 service-app mapping。本 issue 不涉及恢复服务级 provider/consumer metadata，只关注**应用级 metadata 在 metadata center 中的更新与清理**。

应用级服务发现中，metadata 模式为 remote 时，Provider 启动会向元数据中心写入一个 `{application}/{revision}` 节点，其中 `revision` 由该应用暴露的接口列表计算得出。当接口发生变动（新增 / 删除方法等），会产生新的 revision 并写入新节点。

### 当前问题

应用级 metadata 在元数据中心缺少清晰的更新和清理语义，具体表现如下：

1. **无清理策略**：app metadata 按 revision 写入后，没有 unpublish 或 cleanup 机制。版本发布多次后，元数据中会累积大量无用旧 revision，造成存储浪费与查询性能下降。

2. **Zookeeper 不更新已存在节点**：`PublishAppMetadata` 使用 `CreateWithValue` 创建节点，遇到 `ErrNodeExists` 时直接返回 nil——既不更新内容，也不刷新时间。这意味着续约心跳方案在 Zookeeper 下根本无法生效。

3. **各后端行为不一致**：Nacos 和 etcd 基本是覆盖写，Zookeeper 则是「存在即忽略」，三套实现语义割裂。

4. **etcd 路径拼接错误**：etcd 实现中 `rootDir + application + "/" + revision` 缺少 `rootDir` 与 `application` 之间的路径分隔符，导致 key 路径与其他后端不一致。

5. **service-app mapping 无下线语义**：provider unregister 后是否应更新 service-app mapping 不清晰，当前 provider 注册只做 mapping 追加。

涉及代码：`metadata/report/{nacos,zookeeper,etcd}/report.go`、`registry/servicediscovery/service_discovery_registry.go`。

### 讨论与决策

Issue 评论区（eye-gu 与 Alanxtl）经过讨论达成以下共识：

- **定时心跳上报**：remote 模式下应用元数据每天定时上报（续约），刷新 `lastUpdatedTime`。
- **过期检测与清理**：上报完成后检测其他 revision 是否超过 N 天未更新，再交叉验证当前存活实例中是否已无该 revision，无则删除。
- **Zookeeper 必须修复**：否则心跳方案无法工作，由 eye-gu 负责改为覆盖更新。
- **不引入 delete-candidate 标记**：原方案考虑二次校验（先标记候选、下一轮再删），但配置成 N+1 天删除效果等同，增加复杂度无收益，故舍弃。
- **service-app mapping 不自动清理**：mapping 需等接口代码彻底删除后才能安全移除，改为文档说明，由用户在接口下线流程后手动删除。

## 解决方案

### 整体设计

核心思路是给应用级元数据引入「心跳续约 + 过期回收」的生命周期管理：

```
┌──────────────────────────────────────────────────────┐
│  service_discovery_registry.go (调度层)               │
│  ┌────────────────┐    ┌─────────────────────────┐   │
│  │ 续约 Timer       │ →  │ doRenewAppMetadata()    │   │
│  │ (次日凌晨2点     │    │   ↓ 刷新 lastUpdatedTime│   │
│  │  + 随机偏移)     │    │ doGarbageCollect()      │   │
│  └────────────────┘    └─────────────────────────┘   │
├──────────────────────────────────────────────────────┤
│  MetadataReport 接口 (抽象层)                          │
│  + UnPublishAppMetadata(app, revision)                │
│  + ListAppRevisions(app) → []AppRevision              │
│  + URL() *common.URL                                  │
├──────────────────────────────────────────────────────┤
│  实现层 (Zookeeper / Nacos / Etcd)                    │
│  各自实现 UnPublishAppMetadata 和 ListAppRevisions     │
├──────────────────────────────────────────────────────┤
│  数据模型                                              │
│  MetadataInfo.LastUpdatedTime (unix ms)               │
│  AppRevision{Revision, ModifyTime}                    │
│  配置: gc.enabled / gc.window / renew-on-startup      │
└──────────────────────────────────────────────────────┘
```

本 PR 共修改 14 个文件（+1536 / -315），改动分为四层：调度层、接口抽象层、各后端实现层、数据模型与配置层。

### 定时续约（Renew）

#### 调度机制

```
RegisterService()
    └─ startMetadataTimers()          // 仅 remote 模式 + metadataReport != nil 时启动
       └─ startRenewAppMetadataTimer()
          ├─ 启动时立即执行一次（metadata.renew-on-startup=true，可关闭）
          └─ time.AfterFunc(delay)
             │  delay = 次日凌晨 2:00 + [0, 4h) 随机偏移
             ├─ doRenewAppMetadata()
             └─ Reset(24h)            // 循环
```

随机偏移（0~4 小时抖动）用于防止同一应用多个实例同时续约造成惊群。

#### 续约逻辑

`doRenewAppMetadata` 的执行流程：

1. 取当前 `registryID` 对应的 `MetadataInfo`，若为空或 `Revision == "0"`（空 revision）则跳过。
2. 调用 `Snapshot()` 做深拷贝，避免与主协程并发修改冲突。
3. 设置 `snapshot.LastUpdatedTime = time.Now().UnixMilli()`。
4. 调用 `PublishAppMetadata` 覆盖写，刷新元数据中心中的时间戳。
5. 若 `metadata.gc.enabled=true`，执行 `doGarbageCollect()`。

```go
func (s *serviceDiscoveryRegistry) doRenewAppMetadata() {
    registryID := s.url.GetParam(constant.RegistryIdKey, "")
    metaInfo := metadata.GetMetadataInfo(registryID)
    if metaInfo == nil || metaInfo.Revision == "0" {
        return
    }
    // Copy snapshot to avoid data race
    snapshot := metaInfo.Snapshot()
    snapshot.LastUpdatedTime = time.Now().UnixMilli()
    if err := s.metadataReport.PublishAppMetadata(snapshot.App, snapshot.Revision, &snapshot); err != nil {
        logger.Errorf("...")
    }
    // Run garbage collection if enabled
    reportURL := s.metadataReportURL()
    if reportURL != nil && reportURL.GetParamBool(constant.MetadataGCEnabledKey, true) {
        s.doGarbageCollect()
    }
}
```

### GC 清理

`doGarbageCollect` 分四步完成过期 revision 的安全回收：

```
Step 1  ListAppRevisions(app)
        ↓ 获取该应用在元数据中心的所有 revision 列表
Step 2  筛选过期候选
        ↓ 跳过特殊 revision: "0" / "N/A" / "" / 当前实例 revision
        ↓ ModifyTime == 0 → 跳过（老版本未设置时间戳，无法判断过期）
        ↓ ModifyTime < cutoff(now - gc.window 天) → 标记为候选
Step 3  获取存活实例
        ↓ serviceDiscovery.GetInstances(app)
        ↓ 提取每个实例 metadata 中的 "dubbo.exported-services.revisions"
        ↓ 构建 aliveRevisions map
Step 4  安全删除
        ↓ 候选 revision 不在 aliveRevisions 中 → UnPublishAppMetadata 删除
```

安全保证：

- 仅删除**既过期又无存活实例引用**的 revision，双重校验。
- `ModifyTime == 0` 的条目不会被误删，兼容未设置时间戳的老版本数据。
- 当前实例自身的 revision 始终被跳过。
- 窗口期默认 5 天，极端场景（N 天未上报恰好检测时启动）风险极低。

### 接口与数据模型扩展

#### MetadataReport 接口

文件 `metadata/report/report.go` 新增三个方法：

```go
type AppRevision struct {
    Revision   string
    ModifyTime int64 // unix timestamp in milliseconds
}

type MetadataReport interface {
    // ... 原有方法 ...

    // UnPublishAppMetadata 幂等删除指定 revision 的元数据，删除不存在的 revision 不返回错误
    UnPublishAppMetadata(application, revision string) error
    // ListAppRevisions 列出应用的所有 revision 及其最后修改时间
    ListAppRevisions(application string) ([]AppRevision, error)
    // URL 返回创建该 report 的 URL（用于读取配置参数）
    URL() *common.URL
}

// ParseMetadataLastUpdatedTime 从元数据 JSON 中解析 lastUpdatedTime 字段
func ParseMetadataLastUpdatedTime(data []byte) int64
```

设计决策：`ModifyTime` 不依赖 Zookeeper / etcd 的节点 stat（ctime/mtime），而是统一从 JSON 内容的 `lastUpdatedTime` 字段解析。这保证跨后端语义一致，也使 Nacos（基于 config API）能统一处理。

#### MetadataInfo 数据模型

文件 `metadata/info/metadata_info.go` 新增字段与深拷贝方法：

```go
type MetadataInfo struct {
    // ... existing fields
    LastUpdatedTime int64 `json:"lastUpdatedTime,omitempty" hessian:"-"`
}

// Snapshot creates a deep copy for safe concurrent access
func (info *MetadataInfo) Snapshot() MetadataInfo
```

`Snapshot()` 做深拷贝是因为续约在定时器 goroutine 中执行，需避免与主协程并发修改冲突。`hessian:"-"` 标签确保该字段不参与 Hessian 序列化，仅用于 JSON / 远程存储。

### 各注册中心后端改动

#### Zookeeper

| 方法 | 关键改动 |
|------|---------|
| `PublishAppMetadata` | **修复**：`CreateWithValue` 遇到 `ErrNodeExists` 时，改为 `SetContent` 覆盖更新（原实现直接返回 nil） |
| `UnPublishAppMetadata` | 新增：`Delete` 路径，`ErrNoNode` 视为幂等成功 |
| `ListAppRevisions` | 新增：`Children` 列出子节点 → 逐个 `Get` 读取内容 → 解析 `lastUpdatedTime` |
| `URL()` | 新增：返回 `m.url` |

`zkClient` 接口抽象了 `ZookeeperClient` 操作，便于单元测试 mock。Zookeeper 的修复是整个方案的前提——不修复则续约心跳无法刷新时间戳，GC 也就无从判断过期。

#### Nacos

| 方法 | 关键改动 |
|------|---------|
| `UnPublishAppMetadata` | 新增：先删 Java 兼容格式（dataId=app, group=revision），再删 dubbo-go 3.1.x 旧格式（dataId=app:revision） |
| `ListAppRevisions` | 新增：`SearchConfig` 按 dataId=application 分页搜索，过滤出 group=revision 的条目 |
| `URL()` | 新增：`nacosMetadataReport` 增加 `url` 字段 |

#### Etcd

| 方法 | 关键改动 |
|------|---------|
| `UnPublishAppMetadata` | 新增：`Delete` key |
| `ListAppRevisions` | 新增：`GetChildren` 按 prefix 获取，解析 key 最后一段为 revision |
| key 路径 | **修复**：从 `rootDir + app + "/" + rev` 改为 `rootDir + "/" + app + "/" + rev`，与其他实现一致 |
| 客户端 | `etcdClient` 接口 + `etcdClientWrapper` 抽象 `gxetcd.Client`，便于测试 |

#### DelegateMetadataReport

文件 `metadata/report_instance.go` 透传新增的 `UnPublishAppMetadata`、`ListAppRevisions`、`URL()` 到底层实现。

### 配置项

文件 `common/constant/key.go` 新增配置常量，通过 metadata report URL 配置：

| 配置键 | 默认值 | 说明 |
|--------|--------|------|
| `metadata.gc.enabled` | `true` | 是否启用 GC 清理 |
| `metadata.gc.window` | `5`（天） | GC 过期窗口，超过此天数未更新的 revision 成为候选；范围 1-365 |
| `metadata.renew-on-startup` | `true` | 启动时是否立即执行一次续约 |
| `cycle.report`（已有） | `true` | 是否启用定时续约，同时控制 Timer 是否启动 |

### 生命周期管理

- **启动**：`RegisterService()` 注册完成后调用 `startMetadataTimers()`（仅 remote 模式），内部启动续约 Timer。
- **销毁**：`Destroy()` 调用 `stopMetadataTimers()`，Stop Timer 并置 nil，防止销毁后触发。

```go
type serviceDiscoveryRegistry struct {
    // ... existing fields
    renewAppMetadataTimer *time.Timer
}
```

### 已知风险

GC 竞态风险：若持有某 revision 的服务下线超过 N 天，被其他实例判定为过期候选；恰在此时老服务重新上线并写入元数据，新服务可能将该刚写入的元数据删除，导致老服务短暂处于无元数据状态。正常流程下该场景发生概率极低，且下一轮续约会重新写入。

### 小结

本次改动通过「每日定时续约刷新时间戳 + 过期窗口交叉校验存活实例后删除」的组合策略，解决了应用级元数据无限堆积的问题，同时修复了 Zookeeper 不更新已存在节点和 etcd 路径拼接两个缺陷。设计上坚持最小复杂度：不引入 delete-candidate 二次标记，不自动清理 service-app mapping（交由用户手动处理），用单一 `lastUpdatedTime` 字段统一跨后端的过期判定语义。

## Dubbo（Java 版）实现相同逻辑的困难

上述方案在 dubbo-go 中已落地，但 Java 版 Dubbo 若要实现相同的 GC 逻辑，在 Nacos 后端上会遇到 SDK 能力缺口。

核心障碍在于 `ListAppRevisions`——枚举某个应用在元数据中心的所有 revision 配置项。Nacos 2.x 版本的 Java client 没有提供 `searchConfig` 接口，该接口直到 3.x 才引入，而且并不在常用的 `ConfigService` 上，而是在 `ConfigMaintainerService` 中。这意味着在 Nacos 2.x 下，应用无法通过标准 SDK API 获取到自身名下的全部配置列表，GC 的第一步（列出所有 revision）就无法完成。

要绕过这个限制，需要自行构造 Nacos OpenAPI 的 HTTP 请求（`/nacos/v1/cs/configs` 搜索接口）来获取配置列表，这增加了实现复杂度与维护成本，也是该 GC 能力在 Nacos 上跨语言落地时需要特别注意的约束。
