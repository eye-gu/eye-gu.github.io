---
title: Jvm参数
subtitle:
date: 2025-07-15T20:00:09+08:00
slug: 709a52d
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



## error

| 参数          | 解释                                                  |
| ------------- | ----------------------------------------------------- |
| -XX:ErrorFile | 指定JVM发生致命错误（如崩溃）时生成的错误日志文件路径 |
| -XX:OnError   | 当JVM发生致命错误时，执行指定的脚本                   |

## 内存

| 参数                          | 解释                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| -Xms                          | 设置JVM堆内存的初始大小（即最小堆内存）                      |
| -Xmx                          | 设置JVM堆内存的最大大小. 一般和最小相等. 容器中运行时要小于容器的内存限制, 也要给堆外内存留空间 |
| -XX:NewRatio=2                | 设置老年代（Old Generation）与新生代（Young Generation）的比例,默认值为 2，即老年代占堆的 2/3，新生代占 1/3 |
| -XX:SurvivorRatio=8           | 设置 Eden 区与 Survivor 区的比例（默认值为 8，即 Eden 占新生代的 8/10，每个 Survivor 占 1/10） |
| -XX:InitialRAMPercentage=70.0 | 设置堆内存初始值（`-Xms`）占物理内存的百分比                 |
| -XX:MaxRAMPercentage=70.0     | 动态设置 Java 堆内存的最大值占物理内存的百分比               |

## oom

| 参数                            | 解释                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| -XX:+HeapDumpOnOutOfMemoryError | 当发生 `OutOfMemoryError` 时，自动生成堆转储文件（heap dump） |
| -XX:HeapDumpPath=xxx.hprof      | 指定 `OutOfMemoryError` 发生时生成的堆转储文件路径           |
|                                 |                                                              |

## gc

| 参数                       | 解释                                                         |
| -------------------------- | ------------------------------------------------------------ |
| -verbose:gc                | 基础的 GC 信息                                               |
| -XX:+PrintGCDetails        | 详细的GC信息                                                 |
| -Xlog:gc*:file=gc.log:time | 将 GC 日志输出到文件.gc*匹配所有gc标签, file=gc.log将日志写入文件, time每行日志前添加时间戳. jdk9+推荐使用该配置 |



## g1

| 参数                                   | 解释                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| -XX:+UseG1GC                           | 使用g1                                                       |
| -XX:MaxGCPauseMillis=200               | 设置期望的最大 GC 停顿时间目标（默认值为 200 毫秒），G1 会根据此目标调整堆内存布局和回收策略 |
| -XX:G1HeapRegionSize=16M               | 设置每个 Region 的大小（范围 1~32MB，且必须是 2 的幂次），默认值由 JVM 启发式推断 |
| -XX:G1NewSizePercent=5                 | 设置新生代最小占比（默认值为堆内存的 5%）                    |
| -XX:G1MaxNewSizePercent=60             | 设置新生代最大占比（默认值为堆内存的 60%）                   |
| -XX:G1MixedGCCountTarget=8             | 设置混合回收阶段的 GC 次数目标（默认值为 8 次），用于控制老年代 Region 的回收频率 |
| -XX:G1OldCSetRegionThresholdPercent=10 | 设置混合回收时老年代 Region 的最大比例（默认值为 10%）       |
| -XX:ConcGCThreads=4                    | 设置并发标记阶段的线程数（默认值由 JVM 自动计算），通常建议与 CPU 核心数相关 |
| -XX:G1HeapWastePercent=5               | 设置可容忍的堆内存浪费比例（默认值为 5%），影响并发标记后的回收决策 |



## 其他

| 参数                                    | 解释                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| -XX:-OmitStackTraceInFastThrow          | 禁用JVM对频繁抛出的异常的优化（即不省略堆栈信息）.异常频繁抛出的话会省略堆栈信息 |
| -Dfile.encoding=UTF-8                   | 设置JVM默认的文件编码为 UTF-8。避免因系统默认编码不同而导致的乱码问题 |
| -Djava.awt.headless=true                | 启用无头模式（Headless Mode） .表示程序不需要图形界面（GUI）支持，适合服务器端应用。 避免因缺少图形环境而抛出异常 |
| -Djava.security.egd=file:/dev/./urandom | 设置熵源（Entropy Source）为 `/dev/./urandom`。 说明 ：  在Linux系统中，`/dev/random` 提供高质量的随机数，但可能阻塞。 使用 `/dev/./urandom` 可避免阻塞，加快随机数生成速度，适用于对安全要求不高的场景（如Web应用） |
| -XX:+UseContainerSupport                | 启用 JVM 对容器环境的支持, 让 JVM 正确识别容器（如 Docker、Kubernetes Pod 等）的资源限制（如内存和 CPU），从而合理分配资源，避免因资源超限导致应用被容器管理器强制终止。jdk10+默认生效 |
