---
title: Wasm
subtitle:
date: 2026-06-07T15:05:26+08:00
slug: c8c912a
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

# WASM 运行时替换：Wasmtime-Java → Chicory

> Issue: [#6353](https://github.com/apache/shenyu/issues/6353)
> Branch: `fix-6353`
> 类型: 重构 (refactor)

---

## 背景

ShenYu 的 WASM 插件原先依赖 `wasmtime-java`（基于 JNI 的 WASM 运行时），存在以下问题：

| 问题 | 影响 |
|------|------|
| **项目不活跃** | `wasmtime-java` 长期无维护，无新版本发布，存在安全与功能风险 |
| **平台绑定** | 通过 JNI 绑定 `.so`/`.dylib`/`.dll` 原生库，部署环境必须精确匹配 OS + Arch |
| **CI 无法真实测试** | GitHub CI 环境中无法加载原生 wasmtime 库，测试实际是用 Java mock 实现绕过了 WASM 执行——**没有真正的 WASM 调用被测试过** |

## 方案

替换为 [Chicory](https://chicory.dev/) — 纯 Java 实现的 WebAssembly 运行时（Apache 2.0 协议）。

### 选择 Chicory 的理由

- **纯 Java，零原生依赖** — 任何能跑 JVM 的地方都能跑
- **活跃维护** — 定期发版，社区响应及时
- **运行时编译** — `compiler` 模块将 WASM 字节码翻译为 JVM 字节码，执行速度远快于解释模式，也支持 AOT 编译
- **内置 WASI 支持** — 提供 WASI preview1 完整实现

## 改动范围

### 核心类重构：`WasmLoader`

`WasmLoader` 是整个 WASM 插件体系的核心，负责加载 `.wasm` 文件并提供 Java ↔ WASM 互操作能力。

**主要变更**：

| 方面 | wasmtime-java (旧) | Chicory (新) |
|------|-------------------|-------------|
| 实例化 | `new WasmStore()` → `new WasmModule()` → `WasmInstance` | `Parser.parse()` → `Instance.builder().build()` |
| Host Function 注册 | wasmtime 自有 API | `Store.addFunction(HostFunction)` |
| WASI | 手动 mock | Chicory 内置 `WasiPreview1` |
| 内存访问 | `Memory buffer` 操作 | `Memory.readBytes()` / `Memory.write()` |
| 编译优化 | 无 | `MachineFactoryCompiler.compile()` WASM→JVM 字节码，失败则回退解释模式 |
| 关闭 | 需释放原生资源 | 纯 Java，GC 管理，无需显式清理 |

**初始化流程**：

```
1. 定位 .wasm 资源文件
2. 创建 Store
3. 注册 WASI preview1（Chicory 内置 + 覆盖 proc_exit 为 no-op）
4. 注册自定义 Host Function（get_args / put_result）
5. 解析 WASM 模块
6. 尝试运行时编译（compiler），失败则回退解释器
7. 调用 _initialize（TinyGo 兼容）
8. 注册 JVM 关闭钩子
```

**Host Function 协议**：

WASM 端通过 `shenyu` 模块名导入两个函数：

- `get_args(argId: i64, addr: i64, len: i32) -> i32`：Java → WASM 传递参数（Java 写入 WASM 线性内存）
- `put_result(argId: i64, addr: i64, len: i32) -> i32`：WASM → Java 返回结果（WASM 写入，Java 读取）

### TinyGo 编译工具链

本次改动引入 [TinyGo](https://tinygo.org/) 作为 Go → WASM 的编译器，用于构建测试用的 `.wasm` 模块。

**为什么选 TinyGo 而非标准 Go**：

| 对比 | 标准 Go | TinyGo |
|------|---------|--------|
| WASM 输出体积 | ~2MB+（包含完整 Go runtime） | ~4KB（精简 runtime） |
| `-target` 支持 | `wasip1`（需要 WASI runtime） | `wasm-unknown`（纯 WASM，最小依赖） |
| `//go:wasmexport` | 支持 | 支持 |
| 适合场景 | 生产级 Go WASM 应用 | 轻量级插件、测试用例 |

**编译命令**（所有 Go 测试模块统一使用）：

```bash
tinygo build -target wasm-unknown -opt=2 -no-debug -panic=trap -o plugin.wasm main.go
```

参数说明：
- `-target wasm-unknown`：生成最小化 WASM 模块，不绑定特定 WASI 实现，由宿主（Chicory）提供 WASI 支持
- `-opt=2`：启用优化（中等级别）
- `-no-debug`：去除调试信息，减小体积
- `-panic=trap`：panic 时直接触发 WASM trap 而非展开栈（减小 runtime 开销）

**`_initialize` 兼容**：TinyGo `wasm-unknown` target 生成的模块导出 `_initialize` 函数。`WasmLoader` 在实例化后显式调用它以确保 Go runtime 正确初始化：

```java
ExportFunction initFn = instance.export("_initialize");
if (initFn != null) {
    initFn.apply();
}
```

**每个模块的构建流程**：各 Go 测试模块目录下均包含 `Makefile`，支持一键构建并复制到测试资源目录：

```bash
cd shenyu-plugin/shenyu-plugin-wasm-api/src/test/go-wasm-plugin
make build
# 输出: ../resources/org.apache.shenyu.plugin.wasm.api.AbstractWasmPluginTest$GoWasmPlugin.wasm
```

**WASM 文件命名规则**：编译产物按 `Java测试类名$内部类名.wasm` 命名，放在 test resources 对应包路径下，`WasmLoader` 通过 `ClassLoader.getResource()` 自动定位。

### Go 端 ABI 辅助库：`wasmabi`

新增 `pkg/wasmabi/abi.go`，封装了底层 `//go:wasmimport` 调用，为所有 Go 测试 WASM 模块提供统一的与 Java 宿主通信 API：

```go
//go:wasmimport shenyu get_args
func getArgsRaw(argId int64, addr int64, len int32) int32

//go:wasmimport shenyu put_result
func putResultRaw(argId int64, addr int64, len int32) int32
```

上层封装：
```go
// 从 Java 主机读取参数（Java 通过 onGetArgs 写入 WASM 线性内存）
input := wasmabi.GetArgs(argId, buf)

// 向 Java 主机写回结果（WASM 写入线性内存，Java 通过 onPutResult 读取）
wasmabi.PutResult(argId, []byte("result"))
```

`wasmabi` 还提供了 `Eprintln` 用于通过 WASI `fd_write` 向 stderr 输出调试信息，方便在测试中追踪 WASM 侧执行流程。

### 测试变更

这是本次改动的关键价值——**从 mock 测试变为真实 WASM 执行测试**。

**旧测试方式**：
- Java 子类直接覆写 WASM 函数，用 Java 逻辑模拟 WASM 行为
- 实际从未加载和执行 `.wasm` 文件

**新测试方式**：
- 为每个 Handler/Plugin 编写对应的 Go WASM 模块（`go-*-handler/`、`go-*-plugin/`）
- 使用 TinyGo 编译为 `.wasm` 文件
- 测试中真正加载并执行 WASM 模块，验证 Java ↔ WASM 双向通信

**新增 Go 测试模块**：

| 目录 | 用途 | 导出函数 |
|------|------|---------|
| `go-wasm-plugin/` | AbstractWasmPlugin 测试 | execute, before, after |
| `go-shenyu-wasm-plugin/` | AbstractShenyuWasmPlugin 测试 | doExecute, before, after |
| `go-plugin-data-handler/` | AbstractWasmPluginDataHandler 测试 | handlerPlugin, removePlugin, handlerSelector, removeSelector, handlerRule, removeRule |
| `go-meta-data-handler/` | AbstractWasmMetaDataHandler 测试 | handleMetaData, removeMetaData, refresh |
| `go-discovery-handler/` | AbstractWasmDiscoveryHandler 测试 | handlerDiscoveryUpstreamData |

所有模块通过 `go.mod` 的 `replace` 指令共享 `wasmabi` 包：

```go
// go.mod
module shenyu/go-wasm-plugin
go 1.21
require shenyu/wasmabi v0.0.0
replace shenyu/wasmabi => ../../../../shenyu-plugin-wasm-base/src/test/pkg/wasmabi
```

**测试中的 Java ↔ Go WASM 交互流程**：

```
Java 测试                          Go WASM 模块
   │                                   │
   │  execute.apply(argumentId)        │
   │──────────────────────────────────>│
   │                                   │  wasmabi.GetArgs(argId, buf)
   │<──────────────────────────────────│
   │  onGetArgs() → 写入线性内存        │
   │──────────────────────────────────>│
   │                                   │  处理数据
   │                                   │  wasmabi.PutResult(argId, data)
   │<──────────────────────────────────│
   │  onPutResult() → 读取并验证结果    │
   │                                   │
```

**WASM 文件体积对比**：

| 文件 | 旧大小 (Rust+wasmtime) | 新大小 (Rust+chicory) | 新大小 (Go+TinyGo) |
|------|----------------------|---------------------|-------------------|
| PluginDataHandler | 1,680,550 bytes | 61,199 bytes | ~3,650 bytes |
| MetaDataHandler | 1,679,924 bytes | 60,800 bytes | ~4,775 bytes |
| DiscoveryHandler | 1,984,166 bytes | 60,991 bytes | ~4,327 bytes |
| ShenyuWasmPlugin | 1,679,615 bytes | 60,937 bytes | ~4,346 bytes |

Rust 版本体积缩减 96%+（得益于新的编译配置），Go 版本（TinyGo `wasm-unknown`）更是仅 3~5KB。

## 受影响的模块

```
shenyu-plugin/shenyu-plugin-wasm-api/        ← 核心重构 (WasmLoader, AbstractWasmPlugin)
shenyu-plugin/shenyu-plugin-wasm-base/       ← Handler 适配 + 测试重写
shenyu-dist/shenyu-bootstrap-dist/           ← License 文件更新
pom.xml                                      ← 依赖版本管理
```

## 向后兼容性

- **Java API 不变**：`AbstractWasmPlugin`、`AbstractShenyuWasmPlugin`、各 Handler 的公共接口保持一致
- **WASM 导出函数名不变**：`execute`、`doExecute`、`handlerPlugin` 等函数签名未变
- **Host Function 协议不变**：`shenyu.get_args` / `shenyu.put_result` 签名未变
- **用户影响**：使用 WASM 插件的用户只需确保 `.wasm` 文件兼容 WASI preview1，无需修改 Java 代码

## 总结

本次重构将 ShenYu 的 WASM 运行时从基于 JNI 的 `wasmtime-java` 替换为纯 Java 的 `Chicory`，解决了平台兼容性和项目维护性风险。更重要的是，使测试从 mock 模式升级为**真实 WASM 执行验证**，显著提升了 WASM 插件体系的可靠性。通过运行时编译（WASM → JVM 字节码）保证了执行性能。
