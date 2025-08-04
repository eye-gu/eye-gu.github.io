---
title: Okhttp
subtitle:
date: 2025-08-01T16:42:23+08:00
slug: 1699fd9
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

## **okhttp**

在公司代码中看到如下代码

![img](/okhttp/p1.png)

可以看到, 在方法内部初始化okhttpclient, 而且原本是共享的, 是特地改成如此.

![img](/okhttp/p2.png)

查看okhttpclient的注释, 标明了应该是共享对象.如果不按照这个注释, 不是共享对象, 会有什么问题呢?

### **测试**

对现在的代码进行简单的模拟测试, 看看是不是会有问题

```
public static void main(String[] args) throws Exception {

    long start = System.currentTimeMillis();
    List<CompletableFuture<String>> futures = new ArrayList<>();

    for (int i = 0; i < 200; i++) {
        OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(60, TimeUnit.SECONDS)
                .readTimeout(60, TimeUnit.SECONDS)
                .writeTimeout(60, TimeUnit.SECONDS)
                .build();

        Request request = new Request.Builder()
                .url("https://www.baidu.com")
                .get()
                .build();

        futures.add(request(client, request));
    }
    for (CompletableFuture<String> future : futures) {
        future.get();
    }
    long end = System.currentTimeMillis();
    System.out.println("cost:" + (end - start));
    System.in.read();
}

private static CompletableFuture<String> request(OkHttpClient client, Request request) {
    CompletableFuture<String> future = new CompletableFuture<>();
    client.newCall(request).enqueue(new Callback() {
        @Override
        public void onFailure(@NotNull Call call, @NotNull IOException e) {
            future.completeExceptionally(e);
        }

        @Override
        public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
            future.complete(response.body().string());
        }
    });
    return future;
}
```

![img](/okhttp/p3.png)

![img](/okhttp/p4.png)

测试对百度进行200次请求, 同样是异步模式

然后用visualvm查看线程情况, 发现有很多Okhttp的线程, 过一会(应该是gc回收了okhttpclient对象最终回收了线程)才被回收.

说明线程没有复用, 一次请求最少就新建了2个线程, 不仅加大了请求的耗时, 对线程,内存,cpu等宝贵的资源都产生了巨大的压力.

最终耗时:4499毫秒



改成共享方式再次测试:

```
static OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(60, TimeUnit.SECONDS)
        .readTimeout(60, TimeUnit.SECONDS)
        .writeTimeout(60, TimeUnit.SECONDS)
        .build();

public static void main(String[] args) throws Exception {

    long start = System.currentTimeMillis();
    List<CompletableFuture<String>> futures = new ArrayList<>();

    for (int i = 0; i < 200; i++) {

        Request request = new Request.Builder()
                .url("https://www.baidu.com")
                .get()
                .build();

        futures.add(request(client, request));
    }
    for (CompletableFuture<String> future : futures) {
        future.get();
    }
    long end = System.currentTimeMillis();
    System.out.println("cost:" + (end - start));
    System.in.read();
}
```

![img](/okhttp/p4.png)

可以看到只创建了几个线程, 但是耗时却来到了14382毫秒!

### **优化**

共享client虽然资源消耗稳定了, 但是耗时却增加了. 而且在注释掉共享okhttpclient, 改成每个方法新建的那次commit message中写着是因为任务累计, 不知道之前修改的真实的情况, 但很可能是相同的原因, 为了优化耗时做的.

现在让我们尝试进行优化, 第一个配置项是maxRequestsPerHost, 这控制了对相同host的最大并发, 因为调用内部服务都是91jzx.cn的域名, 因此并发大于这个配置时, 就会排队等待. 而默认该值为5, 耗时很有可能是因为在这里进行了排队.

```
static Dispatcher dispatcher = new Dispatcher();
static {
    dispatcher.setMaxRequests(100);
    dispatcher.setMaxRequestsPerHost(100);
}
static OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(60, TimeUnit.SECONDS)
        .readTimeout(60, TimeUnit.SECONDS)
        .writeTimeout(60, TimeUnit.SECONDS)
        .dispatcher(dispatcher)
        .build();
```

修改该配置为100, 就是可以对同个host同时有100个请求, 继续测试, 耗时为4201, 现在耗时又和最初一样了.



```
static Dispatcher dispatcher = new Dispatcher();
static {
    dispatcher.setMaxRequests(100);
    dispatcher.setMaxRequestsPerHost(100);
}
static OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(60, TimeUnit.SECONDS)
        .readTimeout(60, TimeUnit.SECONDS)
        .writeTimeout(60, TimeUnit.SECONDS)
        .dispatcher(dispatcher)
        .connectionPool(new ConnectionPool(100, 10, TimeUnit.SECONDS))
        .retryOnConnectionFailure(true)
        .build();
```

再使用连接池继续优化, 配置连接池后, 后续请求可以复用http链接, 耗时只有1897毫秒了!

不过因为连接池中的链接可能已经被close了, 需要加上retryOnConnectionFailure配置进行重试.



## **对比webclient**

底层实现的对比, 这里就贴claude sonnet 4的回答:

```
这是一个重要的区别。OkHttp 的异步底层仍然是阻塞 I/O，而 WebClient 是真正的非阻塞 I/O。
OkHttp 异步的本质
OkHttp 的异步实际上是：

阻塞 I/O + 线程池 的模式
每个请求仍然会阻塞一个线程等待网络响应
只是把阻塞操作放到了后台线程池中执行

java// OkHttp 异步调用的底层实现
final class AsyncCall extends NamedRunnable {
  @Override 
  protected void execute() {
    // 这个方法在线程池的工作线程中执行
    // 但底层的 socket I/O 仍然是阻塞的
    Response response = getResponseWithInterceptorChain();
    // 线程在这里会阻塞等待网络响应
  }
}
WebClient 的非阻塞 I/O
WebClient 基于 Netty 或其他 NIO 框架：

使用 事件循环 (Event Loop) 模型
真正的非阻塞 I/O，线程不会等待网络响应
通过回调/Future/Promise 机制处理结果

java// WebClient 底层使用 Netty 的非阻塞模型
// 线程发送请求后立即返回，不会阻塞等待
// 当数据到达时，通过事件循环触发回调
具体对比
线程使用模式
OkHttp：
请求1 -> 线程1 (阻塞等待响应)
请求2 -> 线程2 (阻塞等待响应)  
请求3 -> 线程3 (阻塞等待响应)
...

每个并发请求需要一个线程
线程在等待网络 I/O 时被阻塞
默认最大64个并发请求 = 最多64个线程同时阻塞

WebClient：
请求1 -> 发送后立即返回，线程可处理其他任务
请求2 -> 发送后立即返回，线程可处理其他任务
请求3 -> 发送后立即返回，线程可处理其他任务
...
所有响应 -> 通过事件循环异步处理

少量线程（通常是 CPU 核心数的2倍）处理所有请求
线程永不阻塞，持续处理新请求和响应

资源消耗对比
1000个并发请求的场景：
OkHttp：

需要约1000个线程（受限于 maxRequests）
每个线程栈空间约 1MB
总内存消耗：~1GB+ 栈空间
线程切换开销大

WebClient：

只需要少量工作线程（如8-16个）
总内存消耗：远小于 OkHttp
无线程切换开销

验证方式
你可以通过监控线程来验证这个差异：
java// OkHttp 测试
OkHttpClient client = new OkHttpClient();
for (int i = 0; i < 100; i++) {
    client.newCall(request).enqueue(callback);
    // 观察线程数会增长到接近100
}

// WebClient 测试  
WebClient client = WebClient.create();
for (int i = 0; i < 100; i++) {
    client.get().uri("/api").retrieve()
        .bodyToMono(String.class).subscribe();
    // 观察线程数保持很少（通常 < 20）
}
总结

OkHttp 异步 = 阻塞 I/O + 线程池，本质上还是"一请求一线程"模型
WebClient = 真正的非阻塞 I/O + 事件循环，线程复用率极高

这就是为什么在高并发场景下，WebClient 通常比 OkHttp 更高效的根本原因。OkHttp 的"异步"主要解决的是不阻塞调用线程的问题，但底层 I/O 模型仍然是传统的阻塞模式。
```

继续让claude sonnet 4做性能对比:

```
在高并发性能对比上，WebClient 通常会明显优于 OkHttp 异步，但具体差异取决于场景。让我详细分析：
性能对比维度
1. 吞吐量 (TPS)
低并发 (< 100)：

OkHttp: 8000-12000 TPS
WebClient: 6000-10000 TPS
结论: OkHttp 略优，因为没有 Reactive 框架开销

中等并发 (100-1000)：

OkHttp: 5000-8000 TPS，开始出现线程竞争
WebClient: 8000-15000 TPS，事件循环优势显现
结论: WebClient 开始领先

高并发 (1000+)：

OkHttp: 2000-4000 TPS，线程池成为瓶颈
WebClient: 12000-20000 TPS，保持稳定
结论: WebClient 明显优于 OkHttp

2. 延迟表现
并发数     OkHttp P99延迟    WebClient P99延迟
100        50ms             45ms  
500        200ms            80ms
1000       800ms            150ms
2000       2000ms+          300ms
3. 资源消耗对比
内存使用：
并发连接数   OkHttp内存      WebClient内存
100         ~100MB          ~50MB
500         ~500MB          ~80MB  
1000        ~1GB            ~120MB
2000        ~2GB            ~200MB
线程使用：
并发请求    OkHttp线程数    WebClient线程数
100         64 (达到限制)    8-16
500         64 (队列等待)    8-16  
1000        64 (大量排队)    8-16
实际性能测试代码HTTP客户端性能测试对比Code import okhttp3.*;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.time.Duration;
import java.time.Inst性能瓶颈分析
OkHttp 瓶颈点：

线程池限制：默认最大64个并发请求
内存消耗：每个线程约1MB栈空间
上下文切换：大量线程切换开销
连接管理：连接池竞争加剧

WebClient 瓶颈点：

事件循环饱和：极高并发下事件循环可能成为瓶颈
背压处理：需要合理配置背压策略
内存泄漏风险：不当使用可能导致内存泄漏

实际场景建议
选择 OkHttp 当：

低并发 (< 100 请求/秒)
简单场景，不需要复杂的流处理
团队熟悉传统的回调模式
Android 应用

选择 WebClient 当：

高并发 (> 500 请求/秒)
微服务架构，需要大量服务间调用
流式数据处理
Spring 生态项目

优化建议
OkHttp 优化：
javaOkHttpClient client = new OkHttpClient.Builder()
    .dispatcher(new Dispatcher(
        Executors.newFixedThreadPool(200))) // 增加线程池大小
    .connectionPool(new ConnectionPool(200, 5, TimeUnit.MINUTES))
    .build();
WebClient 优化：
javaWebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create().option(ChannelOption.SO_KEEPALIVE, true)
                          .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)))
    .build();
总结
在高并发场景下：

WebClient 明显优于 OkHttp，特别是在 1000+ 并发时
资源利用率：WebClient 内存和 CPU 使用更高效
可扩展性：WebClient 可以处理更高的并发量
延迟表现：WebClient 在高并发下延迟更稳定

但在低并发场景下，OkHttp 的简单性和成熟度可能更有优势。选择应该基于你的具体并发需求和团队技术栈。
```

可以看到, webclient是真正的非阻塞, 而okhttp的异步底层还是同步的, 是通过线程池做到的异步. 两者在高并发的情况下, 性能差距也非常大.



## **灰度**

之前通过javaagent, 还有skywalking插件这两种形式, 做过dubbo,grpc等服务的路由灰度等功能. 类似的, 可以在`DefaultWebClientBuilder`构造webclient对象时, 自动添加一个`ExchangeFilterFunction`实现, filter方法中可以拿到request和一些context信息, 做一些路由策略

![img](/okhttp/p5.png)
