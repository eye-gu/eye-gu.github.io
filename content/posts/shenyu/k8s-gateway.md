---
title: K8s Gateway
subtitle:
date: 2026-06-07T15:23:39+08:00
slug: e1e308d
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
  - shenyu
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

# Apache ShenYu 支持 Kubernetes Gateway API 技术文档

## Issue 背景

| 项目 | 内容 |
|---|---|
| **Issue** | [#6346](https://github.com/apache/shenyu/issues/6346) |
| **标题** | [BUG] Support Kubernetes Gateway API |
| **关联 PR** | [#6347](https://github.com/apache/shenyu/pull/6347) |
| **分支** | `fix-6346` |

**问题陈述**: ShenYu 目前仅支持 Kubernetes **Ingress** 资源做服务发现和路由。但 Ingress 已不再是推荐方案，**Gateway API** 是下一代 Kubernetes 网络标准，提供了更具表达力和面向角色的路由能力（GatewayClass、Gateway、HTTPRoute 等）。需要 ShenYu 作为 Gateway API 控制器运行。

## 方案概述

在已有的 `shenyu-kubernetes-controller` 模块中，新增 Gateway API 模式（与原有 Ingress 模式并行），通过 Kubernetes Informer/Controller 机制 Watch Gateway API CRD 资源，将其转换为 ShenYu 内部的 Selector/Rule 配置。

**模式切换**: 配置 `shenyu.k8s.mode=gateway-api` 启用 Gateway API 模式。

## 架构设计

### 核心资源映射关系

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  GatewayClass    │────▶│     Gateway      │◀────│    HTTPRoute     │
│ (cluster-scoped) │     │  (namespaced)    │     │  (namespaced)    │
│ controllerName   │     │ gatewayClassName │     │   parentRefs     │
│  = "shenyu"      │     │   = "shenyu"     │     │   → Gateway      │
└─────────────────┘     └──────────────────┘     └──────────────────┘
                                                          │
                                                          ▼
                                                ┌──────────────────┐
                                                │    Endpoints     │
                                                │  (后端 Pod IP)    │
                                                └──────────────────┘
```

### 组件协作流程

```
GatewayClassReconciler ──▶ GatewayReconciler ──▶ HTTPRouteReconciler
       │                        │                       │
       │  Accepted=True         │  Accepted=True        │  解析 HTTPRoute
       │  状态回写              │  重新入队关联Route     │  生成 Selector+Rule
       │                        │                       │  写入 ShenYu Cache
       ▼                        ▼                       ▼
   K8s API Server         K8s API Server         ShenyuCacheRepository
```

### ShenYu 概念映射

| Gateway API 概念 | ShenYu 概念 |
|---|---|
| `HTTPRoute.spec.rules[].backendRefs` | `SelectorData.handle` (DivideUpstream 列表，即后端地址) |
| `HTTPRoute.spec.hostnames` | `SelectorData.conditionList` (DOMAIN 类型条件) |
| `HTTPRoute.spec.rules[].matches[].path` | `SelectorData/RuleData.conditionList` (URI 类型条件) |
| `HTTPRoute.spec.rules[].matches[].headers` | `SelectorData/RuleData.conditionList` (HEADER 类型条件) |
| `HTTPRoute.spec.rules[].matches[].queryParams` | `SelectorData/RuleData.conditionList` (QUERY 类型条件) |
| backend service 的 Pod endpoints | `DivideUpstream.upstreamUrl` (ip:port) |

## 核心模块详解

### 常量定义 — `GatewayApiConstants`

```java
GATEWAY_API_GROUP = "gateway.networking.k8s.io"
GATEWAY_API_VERSION = "v1"
SHENYU_GATEWAY_CLASS_NAME = "shenyu"          // GatewayClass 名称标识
SHENYU_CONTROLLER_NAME = "gateway.shenyu.apache.org/shenyu-controller"  // controllerName
```

关键：使用 `DynamicKubernetesObject`（而非编译时强类型模型）处理 Gateway API CRD，因为这些 CRD 不在 kubernetes-client/java 核心库中。

### Reconciler 层 — 三级协调器

#### GatewayClassReconciler
- **Watch 资源**: GatewayClass（cluster-scoped）
- **职责**: 匹配 `spec.controllerName == SHENYU_CONTROLLER_NAME`，更新 status 为 `Accepted=True`
- **级联删除**: GatewayClass 被删除时，重新入队所有引用它的 Gateway

#### GatewayReconciler
- **Watch 资源**: Gateway（namespaced）
- **职责**: 匹配 `spec.gatewayClassName == "shenyu"`，更新 status 为 `Accepted=True` + `Programmed=Unknown`
- **联动机制**: Gateway 创建/更新时，重新入队所有引用该 Gateway 的 HTTPRoute（支持跨 namespace）
- **级联删除**: Gateway 被删除时，级联删除所有关联 Route 的 ShenYu Selector+Rule
- **状态回写**: 通过 `merge-patch+json` 直接调用 K8s API 更新 `/status` subresource

#### HTTPRouteReconciler（核心）
- **Watch 资源**: HTTPRoute（namespaced）
- **职责**:
  1. 验证 HTTPRoute 的 `parentRefs` 引用了 ShenYu Gateway
  2. 如果指定了 `sectionName`，验证 Gateway 有匹配的 listener
  3. **先清理旧配置**（delete-then-recreate 策略）
  4. 调用 `HttpRouteParser` 解析为 ShenYu 配置
  5. 通过 `ShenyuCacheRepository` 写入内存
  6. 在 `GatewayRouteCache` 中绑定 Gateway-Route 关系
  7. 更新 HTTPRoute status（`Accepted=True` + `ResolvedRefs=True`）

**幂等保护**: `isRouteStatusAlreadySet()` 检查已有 status，避免无意义的 patch 导致无限 reconcile 循环。

### 解析层 — `HttpRouteParser`

**核心逻辑**:
1. 遍历 `spec.rules[]`，对每条 rule 解析 `backendRefs`（后端服务引用）
2. 通过 `EndpointsLister` 查找后端 Service 的 Pod IP，构造 `DivideUpstream` 列表
3. 解析 `spec.hostnames` 为 DOMAIN 类型条件
4. 解析 `matches[].path` 支持 `Exact`→EQ、`PathPrefix`→STARTS_WITH、`RegularExpression`→MATCH
5. 解析 `matches[].headers` 和 `matches[].queryParams`
6. **每个 hostname × match 组合** 生成独立的 Selector+Rule 对（保证 AND 语义正确）

**默认参数**: 负载均衡 `RANDOM`，重试 3 次，超时 3000ms。

### 缓存层 — `GatewayRouteCache`（单例）

维护三张内存映射表：

| 映射表 | Key | Value | 用途 |
|---|---|---|---|
| `ROUTE_SELECTOR_MAP` | `namespace/routeName-pluginName` | `List<selectorId>` | 追踪每个 Route 生成的 Selector |
| `GATEWAY_ROUTE_MAP` | `gatewayNamespace/gatewayName` | `List<routeKey>` | 追踪每个 Gateway 关联的所有 Route |
| `ROUTE_GATEWAY_MAP` | `routeNamespace/routeName` | `gatewayKey` | 反向索引：Route → Gateway |

ID 生成器：`AtomicLong` 从 10000 开始递增。

### 仓储层 — `ShenyuCacheRepository`

ShenYu K8s Controller 与 ShenYu 内部缓存之间的桥梁：
- `CommonPluginDataSubscriber`: 操作 Selector/Rule/Plugin 内存缓存
- `CommonDiscoveryUpstreamDataSubscriber`: 操作上游发现数据
- `MetaDataSubscriber` / `MetaDataCacheSubscriber`: 操作元数据缓存

配置中自动启用 `GLOBAL`、`URI`、`NETTY_HTTP_CLIENT`、`DIVIDE`、`GENERAL_CONTEXT` 五个插件。

### Spring Boot 自动配置 — `GatewayApiControllerConfiguration`

**激活条件**: `shenyu.k8s.mode=gateway-api`

由于 K8s Java Client 的 `SharedInformerFactory` 使用 `Class` 做 key，而 Gateway API 的三种 CRD 都映射为 `DynamicKubernetesObject`，必须使用**三个独立的 SharedInformerFactory** 避免碰撞：

| Factory Bean | 监听资源 |
|---|---|
| `gatewayclass-shared-informer-factory` | GatewayClass |
| `gateway-shared-informer-factory` | Gateway |
| `httproute-shared-informer-factory` | HTTPRoute + Endpoints |

每个 Factory 对应一个独立的 ControllerManager，共享一个 `ExecutorService`（daemon 线程池）。

**Controller 配置**:

| Controller | Worker 数 | Resync 周期 |
|---|---|---|
| gatewayClassController | 1 | 1 min |
| gatewayController | 2 | 1 min |
| httpRouteController | 2 | 1 min |

## 关键设计决策

| 决策 | 理由 |
|---|---|
| 使用 `DynamicKubernetesObject` 而非强类型模型 | Gateway API CRD 不在 k8s-client/java 核心库中，运行时动态解析 |
| 三个独立 SharedInformerFactory | 避免同类型 `DynamicKubernetesObject` 的 class key 碰撞 |
| delete-then-recreate 策略 | 简化更新逻辑，避免 diff，保证幂等 |
| hostname × match 拆分为独立 Selector | 请求一次只能匹配一个 hostname，需要保持 AND 语义正确 |
| 直接 okhttp patch status | K8s client 的 `updateStatus` 对 Gson JsonElement 序列化有问题 |
| 缓存路由-网关绑定 | 避免大规模部署下的全集群扫描 |

## 模块结构

### 核心模块

```
shenyu-kubernetes-controller/src/main/java/org/apache/shenyu/k8s/
├── cache/
│   ├── GatewayRouteCache.java       -- Gateway-Route 绑定缓存（单例）
│   ├── IngressCache.java            -- Ingress 资源缓存（原有 Ingress 模式）
│   ├── IngressSecretCache.java
│   ├── IngressSelectorCache.java
│   ├── K8sResourceCache.java
│   ├── SelectorCache.java
│   └── ServiceIngressCache.java
├── common/
│   ├── GatewayApiConstants.java     -- Gateway API 常量
│   ├── IngressConstants.java        -- Ingress 注解常量（原有 Ingress 模式）
│   ├── IngressConfiguration.java    -- 路由配置封装 (SelectorData + RuleData + MetaData)
│   └── ShenyuMemoryConfig.java      -- 内存配置模型
├── parser/
│   ├── HttpRouteParser.java         -- HTTPRoute → ShenYu 配置解析（核心）
│   ├── DivideIngressParser.java     -- Divide Ingress 解析器
│   ├── DubboIngressParser.java
│   ├── GrpcParser.java
│   ├── SofaParser.java
│   ├── WebSocketParser.java
│   ├── ContextPathParser.java
│   ├── IngressParser.java
│   ├── K8sResourceListParser.java
│   └── K8sResourceParser.java
├── reconciler/
│   ├── GatewayClassReconciler.java  -- GatewayClass 协调器
│   ├── GatewayReconciler.java       -- Gateway 协调器
│   ├── HTTPRouteReconciler.java     -- HTTPRoute 协调器（核心）
│   ├── EndpointsReconciler.java     -- Endpoints 协调器
│   └── IngressReconciler.java       -- Ingress 协调器（原有模式）
└── repository/
    └── ShenyuCacheRepository.java   -- ShenYu 缓存仓库
```

### Spring Boot Starter

```
shenyu-spring-boot-starter/shenyu-spring-boot-starter-k8s/
├── GatewayApiControllerConfiguration.java   -- Gateway API 模式配置
└── IngressControllerConfiguration.java      -- Ingress 模式配置（原有）
```

## 依赖

核心外部依赖为 `client-java-spring-integration`（Kubernetes Java Client + Spring 集成），提供 ApiClient、Informer、Controller 框架。

## 集成测试

`shenyu-integrated-test/shenyu-integrated-test-k8s-gateway-api-http/` — 基于 Kind 集群的 Gateway API HTTP 集成测试。
