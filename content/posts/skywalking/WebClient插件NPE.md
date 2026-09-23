---
title: WebClient插件NPE
subtitle:
date: 2026-09-23T10:20:44+08:00
slug: a2b6e71
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
  - skywalking
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

# WebClient 插件 NPE：writeTo 早于 Exit Span 创建

> Issue: [apache/skywalking#13589](https://github.com/apache/skywalking/discussions/13589)
> PR: [apache/skywalking-java#834](https://github.com/apache/skywalking-java/pull/834)
> 类型: Bug 修复 (bugfix)

---

## 现象

提问者（wh88725）的环境与配置：

| 项 | 值 |
|---|---|
| JDK | 17 |
| Spring Boot | 3.4.5 |
| WebClient | 6.2.6 |
| SkyWalking Java Agent | 9.5.0 |

`plugins` 目录放入 `apm-spring-webflux-6.x-plugin-9.5.0.jar`，并**排除** `spring-webflux-5.x-webclient-plugin-9.5.0.jar`——也就是由 6.x 插件单独接管 WebClient。

调用方式是常规写法：

```java
webClient.post().uri(...).headers(...).body(...).retrieve().bodyToFlux(String.class)
```

Agent 日志中刷出 NPE：

```
ERROR InstMethodsInter :
class[class org.springframework.web.reactive.function.client.DefaultClientRequestBuilder$BodyInserterRequest]
before method[writeTo] intercept failure

java.lang.NullPointerException: Cannot invoke
"org.apache.skywalking.apm.agent.core.context.ContextCarrier.items()"
because "contextCarrier" is null
    at ...spring.webflux.v6.webclient.BodyInserterRequestInterceptor.beforeMethod(BodyInserterRequestInterceptor.java:37)
    at ...plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:83)
    at ...DefaultClientRequestBuilder$BodyInserterRequest.writeTo(DefaultClientRequestBuilder.java)
    at ...ExchangeFunctions$DefaultExchangeFunction.lambda$exchange$1(ExchangeFunctions.java:103)
    at ...http.client.reactive.JdkClientHttpConnector.connect(JdkClientHttpConnector.java:117)
    at ...ExchangeFunctions$DefaultExchangeFunction.exchange(ExchangeFunctions.java)
    ...
```

拦截失败被 Agent 吞掉，只落日志，请求本身不受影响——这一点在后面的评审里会变成关键。

## 定位

### 插件有两个拦截点

`spring-webflux-5.x/6.x-webclient-plugin` 把一次 WebClient 调用拆成两个插桩点配合完成：

| # | 目标方法 | 拦截器 | 职责 |
|---|---|---|---|
| ① | `DefaultClientRequestBuilder$BodyInserterRequest#writeTo(ClientHttpRequest)` | `BodyInserterRequestInterceptor` | 从动态字段取 `ContextCarrier`，把 `sw8` 写进请求头 |
| ② | `ExchangeFunctions$DefaultExchangeFunction#exchange` | `WebFluxWebClientInterceptor` | 创建 ExitSpan、生成并持有 `ContextCarrier` |

②的 `afterMethod` 把返回值包了一层 `Mono.deferContextual`（5.x 为 `Mono.subscriberContext()`）：

```java
// WebFluxWebClientInterceptor.afterMethod（整理后）
Mono<ClientResponse> ret1 = (Mono<ClientResponse>) ret;
return Mono.deferContextual(ctx -> {
    AbstractSpan span = ContextManager.createExitSpan(operationName, remotePeer);
    ...
    final ContextCarrier contextCarrier = new ContextCarrier();
    ContextManager.inject(contextCarrier);
    if (request instanceof EnhancedInstance) {
        ((EnhancedInstance) request).setSkyWalkingDynamicField(contextCarrier);
    }
    return ret1.doOnSuccess(...).doOnError(...).doFinally(...);
});
```

注意：carrier 是写进 **`ClientRequest` 的动态字段**，而 ① 读的 `objInst` 正是同一个对象——`DefaultClientRequestBuilder.build()` 返回的 `BodyInserterRequest`，它同时被插桩成 `EnhancedInstance` 并作为参数传给 ②。**同一个对象、两个拦截点，靠赋值顺序传递数据。**

### 时序倒置

`deferContextual` 的 supplier 在**订阅时**才执行。而 `writeTo` 在**装配时**就被调用了：

```
装配期（DefaultWebClient.exchange() 调用链，此时尚未发生任何订阅）
  DefaultWebClient.exchange()
    └─ ObservationFilterFunction.filter
       └─ DefaultExchangeFunction.exchange                ← 拦截点 ②：afterMethod 只包一层 Mono，supplier 不执行
          └─ connector.connect(...)                       ← 直接调用，未包在 defer 中
             └─ requestCallback.apply(request)            ← JdkClientHttpConnector 同步回调
                └─ clientRequest.writeTo(request)         ← 拦截点 ①：carrier 仍为 null → NPE

订阅期（装配完成之后，晚于上面整条链路）
  Mono.deferContextual(...)                               ← ② 补在外层的包装
    ├─ ContextManager.createExitSpan(...)
    ├─ ContextManager.inject(contextCarrier)
    └─ request.setSkyWalkingDynamicField(contextCarrier)  ← carrier 直到这一刻才写入
```

先看调用方。`exchange` 把「把 body 写进请求」这件事包成一个回调，交给连接器去决定何时执行（spring-webflux 6.2.x，第 97-103 行）：

```java
// ExchangeFunctions$DefaultExchangeFunction#exchange
public Mono<ClientResponse> exchange(ClientRequest clientRequest) {
    Assert.notNull(clientRequest, "ClientRequest must not be null");
    HttpMethod httpMethod = clientRequest.method();
    URI url = clientRequest.url();

    return this.connector
            .connect(httpMethod, url, httpRequest -> clientRequest.writeTo(httpRequest, this.strategies))
            .doOnRequest(n -> logRequest(clientRequest))
            ...
}
```

第三个参数 `httpRequest -> clientRequest.writeTo(httpRequest, this.strategies)` 就是 `requestCallback`——堆栈里的 `lambda$exchange$1(ExchangeFunctions.java:103)` 正是它，和 `connect(...)` 在同一行。注意 `connect(...)` 直接写在 `return` 表达式里，方法体里没有任何 `Mono.defer`：这一行一求值，回调就已经交出去了，返回的 `Mono` 只是把后续链路拼装出来而已，尚未订阅。

再看被调用方。回调何时执行完全由连接器说了算，`JdkClientHttpConnector` 的选择是立刻执行——堆栈里的 `connect(JdkClientHttpConnector.java:117)` 就是这一句 `apply`：

```java
// JdkClientHttpConnector#connect
@Override
public Mono<ClientHttpResponse> connect(
        HttpMethod method, URI uri, Function<? super ClientHttpRequest, Mono<Void>> requestCallback) {

    JdkClientHttpRequest jdkClientHttpRequest = new JdkClientHttpRequest(method, uri, this.bufferFactory,
            this.readTimeout);

    return requestCallback.apply(jdkClientHttpRequest).then(Mono.defer(() -> {   // ← 117 行
        HttpRequest httpRequest = jdkClientHttpRequest.getNativeRequest();
        ...
    }));
}
```

`apply` 直接写在方法体里，所以 `connect()` 一被调用，回调就执行、`writeTo` 就被拦截。而真正发请求的 `getNativeRequest()` 被放在 `then(Mono.defer(...))` 里，要等请求体写完才执行：**body 写入被提前了，请求提交没有**，插件恰好卡在两者之间。

本该「② 先写 carrier、① 后读 carrier」的顺序被连接器的实现方式彻底颠倒：**先读后写**。

### 为什么只在部分连接器上出现

`ClientHttpConnector` 的契约只规定「返回一个冷 `Mono`」，并不规定何时回调 `requestCallback`——各实现自己拿主意：

| 连接器 | `requestCallback` 调用时机 | 结果 |
|---|---|---|
| `ReactorClientHttpConnector`（默认） | 推迟到订阅链路内部 | carrier 已就绪，正常 |
| `JdkClientHttpConnector` | `connect()` 方法体内同步调用 | 装配期就 `writeTo`，NPE |
| Jetty / HttpComponents | 同样急切调用 `writeTo` | 装配期就 `writeTo`，NPE |

这解释了为什么这个 bug 长期没被暴露：默认走 Reactor Netty，插件的隐式时序假设刚好成立，而该插件此前主要在 Gateway 场景下验证过。只有换成 JDK / Jetty / HttpComponents 这类连接器才会翻车——提问者只是恰好踩中了（堆栈里的 `JdkClientHttpConnector`）。

> 我这边也是在 `JdkClientHttpConnector` 下撞到同一个 NPE，才回头把这条链路理清。

## 修复

### 第一版：null 保护 + 延迟补注入

`BodyInserterRequestInterceptor` 的两侧都要改。

`beforeMethod` 加 null 判断——装配期拿不到 carrier 就跳过，不再 NPE：

```java
ClientHttpRequest clientHttpRequest = (ClientHttpRequest) allArguments[0];
ContextCarrier contextCarrier = (ContextCarrier) objInst.getSkyWalkingDynamicField();
if (contextCarrier != null) {
    inject(clientHttpRequest, contextCarrier);
}
```

`afterMethod` 负责补上这次错过的注入——如果 `beforeMethod` 时 carrier 还不存在，就把 `writeTo` 返回的 `Mono` 包一层 `Mono.defer`，等到订阅时再取一次 carrier：

```java
// Connectors like JdkClientHttpConnector invoke writeTo eagerly at assembly time,
// before the exchange interceptor sets the carrier at subscription time. Retry the
// injection when the returned Mono is subscribed, before the request is committed.
if (objInst.getSkyWalkingDynamicField() != null || !(ret instanceof Mono)) {
    return ret;
}
final ClientHttpRequest clientHttpRequest = (ClientHttpRequest) allArguments[0];
return Mono.defer(() -> {
    ContextCarrier contextCarrier = (ContextCarrier) objInst.getSkyWalkingDynamicField();
    if (contextCarrier != null) {
        inject(clientHttpRequest, contextCarrier);
    }
    return (Mono<?>) ret;
});
```

补注入的时机是安全的：订阅链路会先经过 ② 的 `deferContextual`（carrier 写入），再到这个 `defer`（carrier 读取并写头），最后才轮到 `JdkClientHttpRequest.getNativeRequest()` 把请求发出去——头仍然落在请求提交之前。

顺带把重复的注入逻辑抽成了私有 `inject()`，5.x 与 6.x 两个拦截器改动完全对称。

### 评审补充：重新订阅会打破延迟注入

第一版提交后，wu-sheng 在评审里指出 `Mono.defer` 这个包装会被**重新订阅**，从而把「日志问题」升级成「请求失败」。

触发场景是 `ExchangeFilterFunction` 里的重试：

```java
.filter((req, next) -> next.exchange(req)
    .flatMap(r -> r.statusCode().is5xxServerError()
        ? r.releaseBody().then(Mono.<ClientResponse>error(new IllegalStateException("5xx")))
        : Mono.just(r))
    .retry(1))
```

问题在于：JDK / Jetty / HttpComponents 下 `writeTo` 只在装配期执行一次，所以重试重新订阅的是**同一个 `Mono.defer` 包装**，作用于**同一个已经 committed 的 `ClientHttpRequest`**。此时 `getHeaders()` 返回的已是只读视图，`set` 抛 `UnsupportedOperationException`。

更要命的是异常的去向：supplier 运行在应用自己的 reactive chain 里，不在 `InstMethodsInter` 内，Agent 不会吞掉它——异常直接成为请求的错误。而修复前 NPE 是被捕获并只记日志的，请求照常发出。

对比复现（Spring 6.2.19 + JDK 连接器，服务端先返回 503 再返回 200）：

| 场景 | 结果 |
|---|---|
| 无 Agent | retry 成功 |
| 只加第一版的延迟注入 | 请求失败，原因为 `UnsupportedOperationException` |
| 延迟注入包在 try/catch 中 | retry 成功，只发送一个 `sw8` 值 |

按评审建议，两个拦截器都加了兜底：

```java
if (contextCarrier != null) {
    try {
        inject(clientHttpRequest, contextCarrier);
    } catch (Throwable t) {
        // headers are read-only once the request is committed (e.g. re-subscribed by a retry)
    }
}
```

吞掉异常不会丢上下文：第一次订阅注入的 `sw8` 头本来就随重试请求一起发出去了。

## 结果

- 合并提交 `5cd38a5`，进入 `apache:main`，里程碑 **9.8.0**，标签 `bug` / `plugin`
- 最终改动 3 个文件、64 行新增：两个拦截器 + `CHANGES.md`
- `CHANGES.md` 记录：

```text
* Fix the `NullPointerException` thrown by the `spring-webflux-5.x-webclient` and
 `spring-webflux-6.x-webclient` plugins when `DefaultClientRequestBuilder$BodyInserterRequest#writeTo` runs
 before any exit span exists. Connectors such as `JdkClientHttpConnector` call `writeTo` eagerly at assembly
 time, while the exchange interceptor creates the exit span and its `ContextCarrier` only at subscription, so
 the interception failed and the `sw8` header was not propagated. The carrier injection is now null-guarded
 and, if the carrier is still absent, retried when the returned `Mono` is subscribed (apache/skywalking#13589).
```

## 小结

- **插件内的隐式时序假设是脆的。** 两个拦截点靠「同一个对象上的动态字段」传递数据，这个约定的成立完全依赖外部框架的调用顺序。装配期与订阅期是两条不同的时间线，插桩点落在哪一条上必须逐个确认，不能靠默认实现的表现反推。
- **在应用的反应式链路里抛异常等于直接失败请求。** Agent 侧代码必须自己兜住所有异常，不能指望 `InstMethodsInter` 的兜底——那个兜底只覆盖拦截器自身的作用域。
- **默认实现跑通不代表没问题。** Reactor Netty 恰好把 `writeTo` 放在订阅链路里；换个连接器时序就反了。修复只在默认配置下验证，这类 bug 会一直沉在水下。
- 最终合并的改动没有附带回归测试（PR 模板里要求补单测），验证依赖评审环境的人工对比复现。

> 参考资料
> [apache/skywalking#13589](https://github.com/apache/skywalking/discussions/13589) · [apache/skywalking-java#834](https://github.com/apache/skywalking-java/pull/834)
