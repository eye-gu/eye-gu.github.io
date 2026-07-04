---
title: Java面试
subtitle:
date: 2026-07-04T11:00:54+08:00
slug: fcec117
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
categories:
  - java
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

# Java 面试知识点整理

## 目录

- [一、JVM 内存与垃圾回收](#一jvm-内存与垃圾回收)
- [二、引用与类加载](#二引用与类加载)
- [三、IO 模型](#三io-模型)
- [四、集合框架](#四集合框架)
- [五、并发编程](#五并发编程)
- [六、Spring 框架](#六spring-框架)
- [七、Netty](#七netty)
- [八、计算机网络(OSI 七层)](#八计算机网络osi-七层)
- [九、分布式与数据库](#九分布式与数据库)
- [十、设计模式](#十设计模式)

---

## 一、JVM 内存与垃圾回收

### 1. 运行时数据区

> 程序计数器,栈,虚拟机栈,方法区(永久代/元空间),堆,堆外内存

**答**：JVM 运行时数据区分两类:线程私有的程序计数器、虚拟机栈、本地方法栈;线程共享的堆、方法区。堆外内存(DirectMemory)由操作系统管理,不受 JVM 堆大小限制,NIO 的 DirectByteBuffer 就用它做零拷贝。

**深入**：

- 方法区是规范,实现有变迁:JDK7 永久代(在堆中)→ JDK8 元空间(用本地内存),解决永久代易 OOM 且难调优的问题。
- 程序计数器是唯一不会 OOM 的区域;栈抛 StackOverflowError/OOM;堆和方法区抛 OOM。

### 2. 堆分代比例

> 堆:新生代:老年代=1:2

**答**：堆默认按新生代:老年代 = 1:2 划分(可由 -XX:NewRatio 调整)。分代是因为大多数对象朝生夕灭,新生代单独高频回收效率更高。

**深入**：-XX:NewRatio=2 表示老年代是新生代的 2 倍;-XX:SurvivorRatio=8 设置 Eden 与单个 Survivor 的比。

### 3. 新生代结构

> 新生代:survivorFrom, survivorTo, eden

**答**：新生代分 Eden 和两个 Survivor(S0/S1),默认 Eden:S0:S1 = 8:1:1,采用复制算法。每次只用 Eden 和一块 Survivor,回收后存活对象复制到另一块 Survivor,始终留一块空闲。

**深入**：两个 Survivor 是为了避免复制时产生碎片;8:1:1 使空间利用率达 90%,是 IBM 统计对象存活率得出的经验值。

### 4. GC 判活算法

> Gc:引用计数,可达性分析

**答**：判断对象存活有两种方法。引用计数实现简单但无法解决循环引用;JVM 主流用可达性分析——从 GC Roots 出发遍历对象图,不可达即可回收。

**深入**：GC Roots 包括:虚拟机栈中引用的对象、方法区的静态属性/常量引用、本地方法栈 JNI 引用、被 synchronized 锁持有的对象、JVM 内部引用(如基本类型的 Class)。

### 5. GC 基础算法

> GC:标记清除,标记整理,复制,分代

**答**：四种基础算法。标记-清除产生碎片;复制算法无碎片但浪费空间;标记-整理无碎片但要移动对象较慢;分代收集是组合策略——新生代用复制、老年代用标记-整理。

**深入**：分代的理论依据是弱分代假说——绝大多数对象朝生夕灭;熬过越多次 GC 的对象越难回收。

### 6. GC 收集器谱系

> serial, serialOld, ParNew, parallScanvenge(Parallel Scavenge), ParallOld, cms, g1, zgc

**答**：按代与并行程度:Serial / Serial Old 单线程;ParNew 多线程新生代;Parallel Scavenge / Parallel Old 吞吐量优先;CMS 低停顿;G1 可预测停顿;ZGC 用着色指针实现亚毫秒级停顿。

**深入**：新生代(Serial/ParNew/Parallel Scavenge)用复制算法,老年代(Serial Old/Parallel Old/CMS)用标记-整理或标记-清除;G1/ZGC 不再按物理分代,整堆 Region 化。

### 7. CMS

> 初始标记,并发标记,重新标记,并发清理

**答**：CMS 以最小停顿为目标,四阶段:初始标记(STW,标记 GC Roots 直接引用)→ 并发标记(遍历对象图)→ 重新标记(STW,修正并发期变更)→ 并发清理。基于标记-清除,会产生浮动垃圾和碎片,JDK9 起废弃。

**深入**：碎片导致 Concurrent Mode Failure 退化为 Serial Old 做 Full GC(长停顿);浮动垃圾需预留老年代空间(-XX:CMSInitiatingOccupancyFraction)。

### 8. G1

```
Region+逻辑分代+rememberSet+Humongous,局部标记复制,整体标记清理
eden满触发younggc,stw
老年代占用达阈值(45%),触发并发标记:初始标记,并发标记,最终标记,筛选回收(筛选最有价值的region)
Mixedgc:回收所有young和部分old
fullgc兜底
-XX:MaxGCPauseMillis=200   最大停顿时间
-XX:G1HeapRegionSize=16M   region大小
-XX:G1NewSizePercent=5 -XX:G1MaxNewSizePercent=60   新生代占比
```

**答**：G1 把堆划分为大小相等的 Region(逻辑分代,Region 动态归属 Eden/Survivor/Old/Humongous),用 Remembered Set 记录跨 Region 引用避免全堆扫描。Eden 满触发 Young GC(STW 复制存活 Region);老年代达阈值触发并发标记后做 Mixed GC(阈值由 -XX:InitiatingHeapOccupancyPercent 控制,JDK 8 默认 45%;JDK 9 起默认开启自适应 IHOP,45% 仅作初始值,运行中按历史 GC 数据动态调整),回收所有年轻代 + 部分价值最高(回收空间/耗时比)的老年代 Region;Full GC 兜底。-XX:MaxGCPauseMillis=200 设目标停顿。

**深入**：

- 相比 CMS 优势:Region 化复制算法无碎片;可设停顿目标,通过优先回收价值最高的 Region 实现。
- Mixed GC 是 G1 核心,平衡停顿与回收效率;只有 Mixed GC 跟不上、堆几乎满时才退化为单线程 Serial 的 Full GC,生产中应极力避免。

---

## 二、引用与类加载

### 1. 四种引用

> 引用:强,软(oom前),弱(gc),虚

**答**：四种引用强度递减。强引用永不回收;软引用内存不足时回收(适合缓存);弱引用下次 GC 即回收(ThreadLocal 的 key);虚引用无法 get 对象,仅用于跟踪回收过程,必须配合引用队列。

**深入**：WeakHashMap 的 key 是弱引用,常用于缓存关联资源;虚引用唯一作用是对象被回收时收到通知,用于管理堆外内存释放(NIO 的 Cleaner)。

### 2. 类加载过程

> 类加载:加载,验证,准备,解析,初始化,卸载

**答**：加载(找 Class 文件生成 Class 对象)→ 验证(字节码安全)→ 准备(静态变量赋零值)→ 解析(符号引用转直接引用)→ 初始化(执行 `<clinit>` 赋真实值)→ 卸载。其中加载、验证、准备、解析合称"连接"。

**深入**：准备阶段 `static int a = 10` 先赋 0,真正赋 10 在初始化阶段;`<clinit>` 由静态变量赋值和 static 块按源码顺序合并,JVM 保证其线程安全。

### 3. 类加载器与双亲委派

> 类加载器:bootstrap, extension, application, 双亲委派

**答**：三层类加载器:Bootstrap(C++ 加载 rt.jar)、Extension(JDK9 起改名 Platform)、Application(加载 classpath)。双亲委派指加载类时先委托父加载器、失败才自己加载,保证核心类不被篡改、避免重复加载。

**深入**：打破双亲委派的典型场景:SPI/JDBC(用线程上下文类加载器加载厂商实现)、Tomcat(每个 Web 应用独立 ClassLoader 做隔离)、热部署、OSGi 网状委派。

---

## 三、IO 模型

> io:同步阻塞,同步非阻塞,nio,信号驱动io,异步io

**答**：五种 IO 模型:阻塞 IO、非阻塞 IO(忙轮询)、IO 多路复用(select/poll/epoll,一个线程监听多个 fd)、信号驱动 IO、异步 IO(内核完成数据拷贝后回调)。Java NIO 基于多路复用(Selector + Channel + Buffer),前四种本质都是同步(数据拷贝阶段仍阻塞)。

**深入**：

- select/poll 有 fd 数量/性能问题(线性遍历、每次全量拷贝 fd);epoll 用红黑树 + 就绪链表,事件驱动,O(1) 就绪通知,Linux 默认。
- NIO 三大组件:Buffer(读写缓冲)、Channel(双向通道)、Selector(多路复用器)。

---

## 四、集合框架

### 1. HashMap

> Hashmap:64+8->红黑树, <=6->链表

**答**：HashMap 链表长度 > 8 且数组长度 ≥ 64 时转红黑树,节点 ≤ 6 时退化回链表,避免哈希冲突严重时查询退化为 O(n)。JDK8 是 数组 + 链表/红黑树,扩容 2 倍重哈希。

**深入**：

- 阈值 8 来源:理想哈希下链表长度服从泊松分布,达 8 的概率约 0.00000006,几乎不会树化;退化用 6 留缓冲避免频繁切换。
- 线程不安全:JDK7 并发扩容头插法成环导致死循环;JDK8 尾插法解决死环但仍可能数据覆盖丢失,并发场景应使用 ConcurrentHashMap。

### 2. ConcurrentHashMap

> 没有node时cas,扩容时协助处理,synchronized锁node

**答**：JDK8 的 ConcurrentHashMap 用 CAS + synchronized:桶为空用 CAS 插入;非空 synchronized 只锁桶头节点(粒度细);扩容时其他线程通过 transfer 协助分段迁移。

**深入**：

- JDK7 是分段锁(Segment,默认 16 段并发),JDK8 改为 CAS + synchronized 锁单桶,并发度更高、实现更简单。
- size 用 baseCount + CounterCell 数组分散计数(CAS 失败时进入 cell),减少竞争。

---

## 五、并发编程

### 1. 线程池

> corePoolSize, maxpoolsize, keeplivetime, unit, workqueue, threadfactory, rejecthandler

**答**：七大参数:corePoolSize 核心线程数、maxPoolSize 最大线程数、keepAliveTime 非核心线程空闲存活时间、unit 单位、workQueue 任务队列、threadFactory 线程工厂、rejectHandler 拒绝策略。执行流程:核心线程满 → 进队列 → 队列满 → 开非核心线程至 max → 触发拒绝策略。

**深入**：

- 四种拒绝策略:Abort(默认抛异常)、CallerRuns(调用线程自己执行,起到限流反压)、Discard、DiscardOldest。
- 参数设置经验:CPU 密集型 N+1(N 为核数),IO 密集型 2N;队列应用有界队列避免任务无限堆积 OOM——这也是阿里规约禁用 Executors.newFixedThreadPool(内部是无界 LinkedBlockingQueue)的原因。

### 2. 任务队列(阻塞队列)

> LinkedBlockingQueue, ArrayBlockingQueue, SynchronousQueue, PriorityBlockingQueue, DelayQueue, LinkedTransferQueue

**答**：常见阻塞队列:ArrayBlockingQueue(有界数组)、LinkedBlockingQueue(链表,Executors 默认无界)、SynchronousQueue(无容量,直接传递)、PriorityBlockingQueue(优先级堆)、DelayQueue(到期才能取)、LinkedTransferQueue。

**深入**：

- 实现基础:大多基于 ReentrantLock 的 Condition(notFull/notEmpty)实现阻塞。
- 应用:newCachedThreadPool 配 SynchronousQueue(无缓冲,来一个建一个线程);newScheduledThreadPool 配 DelayedWorkQueue(DelayQueue 的堆特化实现,按触发时间排序)实现定时;LinkedBlockingQueue 用作生产-消费者解耦。

### 3. 线程状态

> 线程:new, runnable, block, wait, timedwait, terminal

**答**：六种状态:NEW(已建未启)、RUNNABLE(就绪/运行,Java 不区分)、BLOCKED(等 synchronized 监视器锁)、WAITING(无限等待)、TIMED_WAITING(限时等待)、TERMINATED(终止)。synchronized 阻塞为 BLOCKED,wait/Lock.park 为 WAITING。

**深入**：

- BLOCKED 只因 monitor 锁等待;WAITING 可被 notify/unpack 唤醒或被中断。
- wait 会释放锁并阻塞(Object 的方法,需在 synchronized 块内);sleep 不释放锁(Thread 静态方法,任意位置可调)。

### 4. synchronized 锁升级

> 锁:偏向锁,轻量锁(自旋),重量锁

**答**：synchronized 锁升级:无锁 → 偏向锁(单线程访问,记线程 ID)→ 轻量级锁(多线程交替无竞争,CAS + 自旋)→ 重量级锁(竞争激烈,Monitor 阻塞排队)。升级不可逆,JDK15 起默认禁用偏向锁。

**深入**：锁信息存于对象头 Mark Word;偏向锁适合单线程,轻量级锁适合交替执行,重量级锁适合长临界区;升级不可降,但允许批量撤销/重偏向。

### 5. Lock 与 AQS

> Lock:aqs,可中断,可公平,读共享 ReentrantReadWriteLock

**答**：ReentrantLock 基于 AQS 实现,相比 synchronized 多出:可中断(lockInterruptibly)、可公平(按 FIFO 等待)、可超时(tryLock)、可绑定多个 Condition。ReentrantReadWriteLock 读共享写独占,适合读多写少。

**深入**：synchronized 是 JVM 内置(monitorenter/exit 指令)、自动释放、不可中断;Lock 是 API 级别、需手动 finally 释放、功能更灵活。

### 6. 同步工具(信号量/栅栏/CountDownLatch)

> 信号量 / 栅栏 / CountDownLatch

**答**：Semaphore 控制同时访问的许可数(限流);CyclicBarrier 让一组线程相互等待到屏障点后一起继续(可重复使用);CountDownLatch 等待 N 个事件完成(countDown 到 0 后主线程继续,不可重置)。

**深入**：CyclicBarrier vs CountDownLatch:前者可 reset 重用、基于 ReentrantLock 的 Condition 实现、强调"互相等待";后者不可重用、基于 AQS 共享模式、强调"主线程等多个事件"。

### 7. Atomic 与 CAS

> AtomicInteger:volatile + cas, aba:版本号

**答**：AtomicInteger 基于 volatile(保证可见性)+ CAS(保证原子性)实现无锁原子操作。CAS 的 ABA 问题(值从 A→B→A 看似未变)用版本号解决(AtomicStampedReference)。

**深入**：

- CAS 底层依赖 CPU 的 cmpxchg 指令(单核原子,多核加 lock 前缀),由 Unsafe.compareAndSwapXxx 调用。
- CAS 自旋在高竞争下开销大,Java 8 引入 LongAdder 分段累加(Cell 数组)解决高并发计数性能。

### 8. AQS

> aqs: volatile int state + FIFO,两种模式:独占/共享

**答**：AQS(AbstractQueuedSynchronizer)用 volatile int state 表示同步状态 + FIFO 双向队列管理等待线程。两种模式:独占(ReentrantLock,state 0/1 表示重入数)、共享(Semaphore/CountDownLatch/读锁,state 为许可数/剩余数)。采用模板方法模式,子类重写 tryAcquire/tryRelease 等。

**深入**：state 由 volatile + CAS 保证可见与原子;队列是 CLH 变体,节点基于自旋 + park 阻塞,前驱唤醒后继。ReentrantLock 的公平/非公平区别在于:非公平先尝试 CAS 抢锁(允许插队,吞吐高),公平严格排队。

---

## 六、Spring 框架

### 1. Bean 生命周期

```
BeanNameAware, BeanFactoryAware, ApplicationContextAware
BeanPostProcessor#postProcessBeforeInitialization
InitializingBean#afterPropertiesSet
init-method
BeanPostProcessor#postProcessAfterInitialization
DisposableBean, destroy-method
```

**答**：完整流程:实例化 → 属性注入 → Aware 接口(注入名字/容器) → BeanPostProcessor#before → InitializingBean#afterPropertiesSet → init-method → BeanPostProcessor#after(此处是 AOP 生成代理的关键) → 使用 → 销坏(DisposableBean → destroy-method)。

**深入**：BeanPostProcessor 是 Spring 扩展灵魂——ApplicationContextAwareProcessor 注入容器、AutowiredAnnotationBeanPostProcessor 完成 @Autowired、CommonAnnotationBeanPostProcessor 处理 @PostConstruct;循环依赖靠三级缓存(singletonObjects / earlySingletonObjects / singletonFactories)提前暴露早期引用。

### 2. 依赖注入

> 构造器,set方法注入,字段注入,静态工厂,实例工厂,查找方法注入(@lookup)

**答**：注入方式:构造器(推荐,字段可 final 不可变)、setter(可选依赖)、字段注入(@Autowired/@Resource,简洁但不利于测试)。@Autowired 按类型,@Resource 按名称。

**深入**：@Autowired vs @Resource:前者 Spring 注解 byType(多匹配时用 @Qualifier 指定名)、required 默认 true;后者 JSR-250 标准 byName(无匹配再 fallback byType)。构造器注入能在编译期保证依赖不可变,且循环依赖会在启动期立即暴露错误。

### 3. AOP 代理方式

> Aop代理方式:默认接口则用jdk,否则用cglib

**答**：Spring AOP 动态代理:目标类实现接口默认用 JDK 动态代理(基于接口生成代理),否则用 CGLIB(生成目标类子类)。可设 proxyTargetClass=true 强制 CGLIB;Spring Boot 2.x 起默认 CGLIB。

**深入**：

- JDK 代理基于 Proxy + InvocationHandler 反射调用;CGLIB 基于 ASM 生成子类覆盖方法,final 方法/类无法代理。
- AOP 失效场景:类内部方法自调用(不经过代理对象)、非 public 方法、static/final 方法。解决:注入自身代理或 AopContext.currentProxy()。

### 4. MVC 执行流程

```
1. http请求到dispatcherservlet
2. HandlerMapping
3. controller
4. 返回ModelAndView
5. dispatcherservlet查找viewresolve
6. 返回http响应
```

**答**：核心是前端控制器 DispatcherServlet。请求到达后:HandlerMapping 找到对应 Handler(含拦截器链) → HandlerAdapter 反射执行 Controller → 返回 ModelAndView → ViewResolver 解析视图 → 渲染响应。(@ResponseBody 则由 RequestResponseBodyMethodProcessor 直接写 JSON,跳过视图解析)

**深入**：HandlerMapping 解析 @RequestMapping 的 URL 到 HandlerMethod;HandlerAdapter 适配多种处理器(注解 / HttpRequestHandler / SimpleController),解决 DispatcherServlet 与具体 Controller 的解耦。

### 5. 事务

> 事务

**答**：Spring 声明式事务基于 AOP,通过 @Transactional 生效,默认仅对 RuntimeException 回滚。有七种传播行为(REQUIRED 默认 / REQUIRES_NEW 等)和四种隔离级别。常见失效场景:自调用(不经过代理)、非 public 方法、异常被 catch 吞掉、传播行为错误、抛出非 RuntimeException 默认不回滚。

**深入**：

- 七种传播行为:REQUIRED(有则加入无则新建)、REQUIRES_NEW(总是新开、挂起当前)、NESTED(嵌套子事务,基于 savepoint)、SUPPORTS / NOT_SUPPORTED / NEVER / MANDATORY。
- 本质:TransactionInterceptor 拦截方法,通过 PlatformTransactionManager 在方法前后开启/提交/回滚事务。

---

## 七、Netty

### Reactor 模式

> netty reactor模式:单线程,多线程,主从

**答**：Reactor 三种模式:单 Reactor 单线程(一个线程处理连接与 IO,简单但无法扛压)、单 Reactor 多线程(一个线程接连接,一组线程处理 IO)、主从 Reactor(MainReactor 处理连接,SubReactor 处理读写)。Netty 默认主从(BossGroup 接连接,WorkerGroup 处理 IO)。

**深入**：Netty 核心组件:EventLoop(绑定一个线程,串行无锁)、ChannelPipeline(责任链处理 IO 事件)、ByteBuf(引用计数 + 池化缓冲)。BossGroup 通常 1 个线程,WorkerGroup 默认 2×CPU 核数;EventLoop 的串行化设计天然规避了多线程并发问题。

---

## 八、计算机网络(OSI 七层)

### 1. OSI 七层 与 TCP/IP 四层

> OSI 七层 / tcp 四层

**答**：OSI 七层从下到上:物理层、数据链路层、网络层、传输层、会话层、表示层、应用层。实际工程用 TCP/IP 四层:网络接口层(对应 OSI 1-2)、网际层 IP(对应 3)、传输层 TCP/UDP(对应 4)、应用层 HTTP/FTP/DNS(对应 OSI 5-7)。

**深入**：每层对应的设备与协议:物理层(中继器/集线器)→ 链路层(交换机,MAC 地址,ARP)→ 网络层(路由器,IP,ICMP)→ 传输层(TCP/UDP,端口)→ 应用层(HTTP/HTTPS/DNS)。数据在各层封装时加报文头,链路层加 MAC 头 + 尾(FCS)。

### 2. TCP vs UDP

**答**：TCP 面向连接、可靠(序号/确认/重传/流量/拥塞控制)、字节流、全双工、一对一,代价是开销大、慢,用于文件传输/HTTP。UDP 无连接、不可靠、数据报文、支持广播/多播、一对多,开销小、快,用于 DNS/视频/游戏。

**深入**：TCP"面向字节流"导致应用层数据无边界(产生粘包问题);UDP"面向报文"保留边界不粘包。TCP 头部 20 字节起,UDP 头部固定 8 字节。QUIC(HTTP/3 底层)基于 UDP 在用户态实现可靠传输,规避了 TCP 内核态升级困难。

### 3. TCP三次握手

**答**：三次握手建立 TCP 连接:客户端发 SYN(seq=x)→ 服务端回 SYN+ACK(seq=y,ack=x+1)→ 客户端发 ACK(ack=y+1),连接建立。状态变化:客户端 CLOSED→SYN_SENT→ESTABLISHED;服务端 LISTEN→SYN_RCVD→ESTABLISHED。

**深入**：

- **为什么三次而非两次**：两次握手下,服务端回 SYN+ACK 后即建连,若该包是历史延迟报文,服务端会白白占用资源等待。三次握手让客户端用最终 ACK 决定连接是否真正建立,防止历史失效连接初始化服务端资源。同时三次握手能让双方都确认彼此收发能力正常。
- SYN 队列与 accept 队列:服务端收到 SYN 进半连接队列(SYN_RCVD),收到三次握手的 ACK 后从半连接队列移入全连接队列,accept() 从全连接队列取。

### 4. 四次挥手

**答**：四次挥手释放连接:主动方发 FIN → 被动方回 ACK(进入半关闭,被动方还能发数据)→ 被动方数据发完后发 FIN → 主动方回 ACK,被动方直接关闭。主动方进入 **TIME_WAIT**,等待 **2MSL** 后才真正 CLOSED。

**深入**：

- **为什么四次**:TCP 全双工,每个方向的关闭需独立 FIN+ACK。被动方收到 FIN 时可能还有数据没发完,所以 ACK 和 FIN 要分开发(中间是 CLOSE_WAIT)。
- **为什么 TIME_WAIT 等 2MSL**:① 保证主动方最后那个 ACK 能到达被动方(若丢失,被动方会重发 FIN,主动方重发 ACK,一来一回最多 2MSL);② 让本次连接的旧报文在网络中消亡,防止干扰同四元组的新连接。
- **TIME_WAIT 过多问题**:主动关闭方会堆积大量 TIME_WAIT(默认 60 秒),耗尽端口。解决:服务端主动关闭+客户端处理;调小 tcp_max_tw_buckets;开启 tcp_tw_reuse(客户端复用,依赖时间戳);长连接/连接池减少建连。

### 5. TCP 可靠性机制

**答**：TCP 通过以下机制保证可靠:校验和(检测数据完整)、序号与确认号(保证有序与到达)、超时重传(发送方超时未收 ACK 重发)、流量控制(滑动窗口,接收方限制发送方)、拥塞控制(慢启动/拥塞避免/快重传/快恢复)。

**深入**：

- 超时时间 RTO 动态计算:基于 RTT 的加权平均(SRTT)动态调整,而非固定值。
- 快重传:收到 3 个重复 ACK 立即重传丢失段,不等超时;配合快恢复把拥塞窗口减半而非回到慢启动,提升恢复速度。

### 6. 流量控制 vs 拥塞控制

**答**：两者都限速但对象不同。**流量控制**是端到端,接收方通过通告窗口大小(rwnd)限制发送方,防止发太快淹没接收方。**拥塞控制**是全局,发送方感知网络拥塞动态调整拥塞窗口(cwnd),防止网络瘫痪。

**深入**：发送窗口实际 = min(rwnd, cwnd)。拥塞控制四算法:慢启动(cwnd 指数增长到 ssthresh)→ 拥塞避免(cwnd 线性增长)→ 快重传(3 个重复 ACK)→ 快恢复(cwnd 减半,不回慢启动)。

### 7. 粘包问题

**答**：TCP 是字节流无边界,Nagle 算法合并小包、接收方应用未及时读取缓冲区,都会导致多个应用层消息"粘"在一起。本质不是 bug 而是流式特性。解决在应用层定边界:固定长度、特殊分隔符(如 \n)、长度字段(Header 定长,如 TLV/Netty LengthFieldBasedFrameDecoder)。

**深入**：UDP 面向报文保留边界,不会粘包。Nagle 算法(发的小包未确认前先攒着)会与延迟 ACK 互相作用产生额外延迟,实时性敏感场景可关 TCP_NODELAY。

### 8. HTTP 基础

**答**：HTTP 是无状态、基于请求-响应的应用层协议,基于 TCP(默认端口 80)。请求报文 = 请求行(方法 URL 版本)+ 头 + 空行 + 体;响应报文 = 状态行 + 头 + 空行 + 体。

**深入**：常见状态码:200 OK / 301 永久重定向 / 302 临时重定向 / 304 协商缓存命中 / 400 参数错 / 401 未认证 / 403 禁止访问 / 404 不存在 / 500 服务端错误 / 502 网关错误 / 503 不可用 / 504 网关超时。常见方法:GET(幂等,语义获取)、POST(非幂等,语义创建)、PUT(幂等,更新)、DELETE(幂等)。

### 9. HTTP 版本演进

**答**：HTTP/1.0 每次请求新建 TCP 连接(短连接)。HTTP/1.1 默认长连接(Connection: keep-alive 复用连接)、管道化(响应必须按序,有线头阻塞)、Host 头支持虚拟主机。HTTP/2 二进制分帧、多路复用(一个 TCP 上并发多请求,解决应用层线头阻塞)、头部压缩(HPACK)、服务端推送。HTTP/3 基于 QUIC(UDP),解决 TCP 层的线头阻塞。

**深入**：HTTP/2 多路复用虽解决应用层阻塞,但底层仍是单条 TCP——一旦丢包,所有流都要等重传,这是 TCP 层线头阻塞。QUIC 用 UDP 在用户态独立维护每流的可靠传输,单流丢包不阻塞其他流;还合并了三次握手 + TLS 握手(0-RTT/1-RTT),建连更快。

### 10. HTTPS 与 TLS

**答**：HTTPS = HTTP + TLS/SSL(默认端口 443),提供加密(防窃听)、认证(防伪装)、完整性(防篡改)。流程:TCP 三次握手 → TLS 握手(协商加密套件、验证证书、交换密钥) → 对称加密传输业务数据。

**深入**：

- **为什么混合加密**:TLS 握手用非对称加密(RSA/ECDHE)安全交换会话密钥(解决密钥分发问题),传输用对称加密(AES,性能高)——结合两者优势。
- 证书链:服务端证书由 CA 签名,客户端用本地 CA 根证书逐级验证到受信根,确认服务端身份。
- TLS 1.3 把握手从 2-RTT 减到 1-RTT(支持 0-RTT 恢复),废弃 RSA 密钥交换(只保留前向安全的 ECDHE),提升安全与速度。

### 11. DNS

**答**：DNS 将域名解析为 IP(UDP 53 端口)。解析过程:浏览器/OS 缓存 → 本地域名服务器(递归查询)→ 根域名服务器 → 顶级域名服务器(.com)→ 权威域名服务器(返回最终 IP)。本地服务器拿到结果后缓存并返回客户端。

**深入**：递归查询(客户端→本地服务器,由它替你跑完链路)与迭代查询(本地服务器逐级向根/顶级/权威查询,每次得到下一级地址)。基于 UDP 是因为查询报文小、要求快;区域传送(主从同步)用 TCP 保证大数据可靠。

### 12. Cookie / Session / Token

**答**：Cookie 是客户端存储的键值对,每次请求自动带给服务端,有大小限制(4KB)。Session 是服务端存储、用 Cookie 中的 sessionId 关联。Token(如 JWT)是无状态的、客户端持有,服务端不存储仅验签,天然适合分布式。

**深入**：JWT = Header.Payload.Signature,服务端用密钥验签确认未被篡改,无状态无需查库。缺点:签发后无法主动吊销(需配合黑名单)、Payload 不加密(勿放敏感数据)。Cookie 安全属性:HttpOnly(防 XSS 窃取)、Secure(仅 HTTPS)、SameSite(防 CSRF)。

---

## 九、分布式与数据库

### 1. 事务 ACID

> 事务 acid

**答**：事务四特性:原子性(Atomicity,操作全做或全不做,靠 undo log 回滚)、一致性(Consistency,执行前后数据正确)、隔离性(Isolation,并发事务互不干扰,靠锁/MVCC)、持久性(Durability,提交后永久,靠 redo log)。一致性是目标,其余三者是手段。

**深入**：

- MySQL 实现:原子性靠 undo log(回滚段记录旧值)、持久性靠 redo log(WAL,先写日志再刷盘,崩溃恢复)、隔离性靠锁 + MVCC(快照读)。
- 并发问题与隔离级别:读未提交→脏读;读已提交→不可重复读;可重复读(RR,MySQL 默认)→幻读(MySQL 用间隙锁 + MVCC 基本解决);串行化→无问题但并发最低。

### 2. CAP 定理

> 分布式一致性 cap

**答**：CAP 指分布式系统三选二:一致性(Consistency,所有节点同一时刻数据一致)、可用性(Availability,每次请求都有响应)、分区容错性(Partition tolerance,网络分区时仍能工作)。网络分区不可避免,所以实际是 CP 或 AP 二选一。

**深入**：

- CP(牺牲可用):ZooKeeper、etcd,分区时为保证一致会拒绝服务或阻塞。AP(牺牲一致):Eureka、Cassandra,分区时各分区继续服务,允许数据暂时不一致。
- BASE 是 AP 的实践:基本可用、软状态、最终一致。多数互联网系统选 AP + 最终一致,通过消息队列/异步复制对齐数据。

### 3. Raft 协议

**答**：Raft 是易于理解的强一致共识算法。通过**选主**(Leader Election)和**日志复制**(Log Replication)保证集群一致:客户端所有写请求只发给 Leader,Leader 写本地日志后并行复制给 Follower,过半确认即提交并响应客户端。三个状态:Leader、Follower、Candidate。

**深入**：

- **选主**:节点随机超时(150-300ms)最先超时成为 Candidate,向其他节点拉票,获过半票成为 Leader,任期(term)递增防止脑裂。
- **过半机制**:Raft 的过半确认隐含——任何已提交的日志必然存在于过半节点,新 Leader 必然包含所有已提交日志,从而保证安全性(safety)。
- 对比 Paxos:Paxos 理论更强但难理解难工程化;Raft 通过"强 Leader"简化模型,etcd/Consul 都用它。

### 4. ZooKeeper 分布式锁

**答**：基于 ZooKeeper 的临时顺序节点实现公平锁。加锁:在锁节点下创建临时顺序节点 /lock00001,判断自己是否最小节点,是则获锁,否则监听前一个节点的删除事件排队。释放:客户端断开会话,临时节点自动删除,唤醒后继者。

**深入**：

- **"羊群效应"与顺序节点**:若所有等待者都监听同一节点,删除时会唤醒所有人竞争(惊群)。顺序节点让每个等待者只监听前一个,避免惊群,实现公平排队。
- 临时节点保证客户端宕机时会话失效,锁自动释放,避免死锁;对比 Redis 的 SETNX 需自己处理过期/续期(Redisson 看门狗),ZK 方案在一致性上更强(CP),但写入性能低于 Redis。

---

## 十、设计模式

### 创建型/结构型/行为型 23 种

> 设计模式

**答**：GoF 23 种设计模式分三类。创建型关注对象创建:单例、工厂、抽象工厂、建造者、原型。结构型关注类与对象组合:适配器、装饰器、代理、外观、桥接、组合、享元。行为型关注对象交互:策略、模板方法、观察者、责任链、状态、命令、迭代器、中介者、备忘录、访问者、解释器。

**深入**：JDK/Spring 中的典型应用:

- 单例:Spring Bean 默认单例、Runtime.getRuntime()。

- 工厂:Spring BeanFactory、Calendar.getInstance()。

- 代理:Spring AOP 的 JDK 动态代理/CGLIB、MyBatis Mapper。

- 责任链:Servlet FilterChain、Spring Interceptor、Netty Pipeline。

- 观察者:Spring ApplicationEvent、Guava EventBus。

- 模板方法:Spring JdbcTemplate、AbstractApplicationContext#refresh。

- 适配器:Spring MVC 的 HandlerAdapter。

  面试常考 6 个核心:单例(双重检查/DCL)、工厂、代理(动态代理)、策略(消除 if-else)、模板方法、责任链——结合实际场景能讲清"为什么用"比背诵定义更重要。