---
title: "Kitex Release v0.16.2"
linkTitle: "Release v0.16.2"
projects: ["Kitex"]
date: 2026-05-08
description: >
---

> 本次 release notes 汇总了 v0.16.0、v0.16.1、v0.16.2 三个版本的变更，对外发布版本为 v0.16.2。

## **重要变更介绍**

### **公告**
1. **netpollmux 不再维护**：`pkg/remote/trans/netpollmux` 包以及对应的 `client.WithMuxConnection` / `server.WithMuxTransport` 选项整体标记为废弃，netpollmux 模块不再维护。详见 [连接多路复用](/zh/docs/kitex/tutorials/basic-feature/connection_type#连接多路复用)。
2. **部分接口调整**：对普通使用的用户无影响，但如果有扩展或特殊依赖会有影响，详见 [**特殊变更**] 部分。

### **新特性**
1. **二进制泛化调用：请求粒度指定 IDL Service Name，大幅减少维护的泛化 Client 数量**

   旧版二进制泛化 Client 与 IDL Service 绑定，需要维护大量 Client。本版本支持请求时动态指定 IDL Service Name，通过新增的 `callopt.WithBinaryGenericIDLService(svcName)` / `streamcall.WithBinaryGenericIDLService(svcName)` 调用选项指定，且调用维度的配置优先级更高。Ping-Pong 和流式调用均支持，详见 [动态指定 IDL Service Name](/zh/docs/kitex/tutorials/advanced-feature/generic-call/basic_usage#动态指定-idl-service-name)。

2. **流式接口 Recv 超时控制**

   新增 `streaming.TimeoutConfig`，支持对流式接口 Recv 进行细粒度超时控制，可单独配置超时时间，并通过 `DisableCancelRemote` 控制超时后是否级联取消远端流。提供两种配置入口：
   - Client 维度：`client.WithStreamRecvTimeoutConfig(streaming.TimeoutConfig)`
   - 单次调用维度：`streamcall.WithRecvTimeoutConfig(streaming.TimeoutConfig)`

   Recv 超时返回新错误码 `codes.RecvDeadlineExceeded`（值为 `17`）以及哨兵错误 `kerrors.ErrStreamingTimeout`，便于区分超时类型。

3. **流式细粒度事件追踪 - StreamEventHandler**

   新增独立于 Tracer 的流式事件回调机制，可以感知流式协议层面的核心 Event（Stream 开始、Recv、Send、Recv Header、Stream 结束），便于定制细粒度的流式监控：
   - Client：`client.WithStreamEventHandler(rpcinfo.ClientStreamEventHandler)`
   - Server：`server.WithStreamEventHandler(rpcinfo.ServerStreamEventHandler)`

   同时新增 `stats.StreamStart`、`stats.StreamRecvHeader`、`stats.StreamFinish` 事件类型。详见 [StreamX 流式细粒度事件追踪](/zh/docs/kitex/tutorials/basic-feature/streamx/streamx_event_handler)。

### **功能/性能优化**
1. **Kitex gRPC：内存优化与连接泄漏修复**
   - **HTTP/2 write buffer 复用与 framer 池化**：支持连接维度的 write buffer 池化复用，降低空闲连接内存占用，适用于直接承接大量 gRPC 连接的场景。通过 `client.WithGRPCReuseWriteBuffer` / `server.WithGRPCReuseWriteBuffer` 启用，并在 `ReuseWriteBufferConfig.EnableReuseHTTP2FramerBuffer` 中进一步启用 framer 层的池化。
   - **Client 侧 cancel 对象分配优化**：减少 gRPC client 在统一 cancel 场景下的对象分配，避免网关等需要频繁 cancel 的场景分配大量对象。
   - **连接池泄漏修复**：修复连接关闭且无后续调用时 gRPC 连接未正确回收的问题。

2. **TTHeader Streaming：内存优化**

   发送端直接复用底层 TCP 的流控能力，避免 OOM。

3. **服务发现：实例变更事件对象分配优化**

   减少因下游实例变化频繁而导致的实例变更事件对象分配，降低 GC 压力。同时新增 `event.GetDefaultEventNum()` / `event.SetDefaultEventNum(num int)`，支持调整默认事件队列容量（默认 200）。

4. **RPCInfo 字段内联**

   新增 `rpcinfo.NewRPCInfoWithInlineFields() RPCInfo`，返回带内联子对象的 RPCInfo，减少每次请求的 pool get 与对象分配；server 热点路径采用该内联方式，提升请求处理性能。

### **问题修复**
1. **流式相关问题修复**
   - **流式 Server 侧 Recv/Send panic**：流式 RPC 不再复用 rpcinfo，彻底避免因 handler 提前退出、但仍在异步使用 Stream 导致的 rpcinfo 并发读写 panic。
   - **流式旧 Server Middleware 扩展不生效**：修复旧 Server Middleware 中扩展的 Stream 对象被意外丢弃的问题。

2. **其他问题修复**
   - **rpcTimeout ticker 泄漏修复**（极小概率）：关闭 rpcTimeout pool 中的 ticker，防止资源泄漏。线上绝大部分场景无问题，只在 QPS 极低且接口处理时间较短的场景会感知到。
   - **泛化调用容器字段写入不同类型元素 panic**（小概率）：当某个容器字段被写入了不同类型的元素时会 panic（例如字段本身是 `[]uint8`，第一个传入 `uint8`，第二个却传入 `string`）。修复后返回错误而不是直接 panic。

### **特殊变更 - 少数服务可能会有影响**
> 主要为 Breaking Changes 与接口废弃，对绝大部分用户无影响，请有特殊依赖的用户关注。

#### Breaking Changes

1. **`pkg/remote/trans/ttstream/container` 路径迁移**（#1952）

   迁移至 `pkg/remote/trans/ttstream/internal/container`。该包本就不是对外 API，外部如有依赖请将所需实现复制到自己的 module 中。

2. **`grpc.NewClientTransport` 回调时序变化**（#1945）

   `onClose` / `onGoAway` 回调改为在 transport 状态转换为 closing/draining 并释放 `http2Client.mu` 之后触发。依赖旧时序的调用方请迁移至 `grpc.NewClientTransportWithConfig`。

3. **`consistBalancer` 一致性哈希算法替换**（#1924）

   `consistBalancer` 的 consistent-hash key/node 哈希将 `github.com/bytedance/gopkg/util/xxhash3` 替换为 `hash/maphash`。`hash/maphash` 在每个进程使用随机种子，hash 值会在**不同 client 副本和重启之间不同**。

#### 接口废弃
> 当前版本仅标记废弃，仍保留可用，请及时迁移到新接口。

1. **netpollmux 整体废弃**（#1933）
   - `client.WithMuxConnection(connNum int)` 废弃
   - `server.WithMuxTransport()` 废弃
   - `pkg/remote/trans/netpollmux` 包整体标记为废弃，不再维护

   废弃原因详见 [连接多路复用](/zh/docs/kitex/tutorials/basic-feature/connection_type#连接多路复用)。

2. **`kerrors.ErrRPCFinish` 恢复为废弃符号**（#1953）

   该 API 在 v0.15.0 被移除，v0.16.2 重新加回作为废弃符号，便于 pre-v0.15 代码继续编译。

3. **流式相关接口废弃**
   - `client.WithStreamRecvTimeout` 废弃，使用 `client.WithStreamRecvTimeoutConfig` 替代（#1911）
   - `streamcall.WithRecvTimeout` 废弃，使用 `streamcall.WithRecvTimeoutConfig` 替代（#1911）

## **详细变更**

### Feature
* feat(generic): support specifying IDL Service Name per call for Binary Generic by @DMwangnima in [#1928](https://github.com/cloudwego/kitex/pull/1928)
> 特性：二进制泛化调用支持请求粒度指定 IDL Service Name
* feat(server): add in-process LocalCaller for unary calls by @xiaost in [#1930](https://github.com/cloudwego/kitex/pull/1930)
> 特性：新增进程内 LocalCaller，支持 unary 本地调用
* feat: gRPC supports reusing write buffer for each connection by @DMwangnima in [#1918](https://github.com/cloudwego/kitex/pull/1918)
> 特性：gRPC 支持连接维度的 write buffer 复用，降低空闲连接内存占用
* feat(streaming): add Recv timeout config and adjust gRPC error/log by @DMwangnima in [#1911](https://github.com/cloudwego/kitex/pull/1911)
> 特性：流式 Recv 超时配置，并调整 gRPC 错误信息与日志级别
* feat(streaming): support detailed tracing events by @DMwangnima in [#1905](https://github.com/cloudwego/kitex/pull/1905)
> 特性：流式调用支持细粒度事件追踪（StreamEventHandler）
* feat(streaming): remove rpcinfo reuse for streaming by @DMwangnima in [#1909](https://github.com/cloudwego/kitex/pull/1909)
> 特性：流式 RPC 不再复用 rpcinfo

### Fix
* fix(ttstream): ttstream should not recycle connection when Recv timeout with DisableCancelRemote=true by @DMwangnima in [#1952](https://github.com/cloudwego/kitex/pull/1952)
> 修复：开启 `DisableCancelRemote=true` 时，Recv 超时后不再回收长连接，避免丢包
* fix(gRPC): connection pool leak when connection is closed and there are no more subsequent calls by @DMwangnima in [#1945](https://github.com/cloudwego/kitex/pull/1945)
> 修复：连接关闭且无后续调用时 gRPC 连接池泄漏的问题
* fix(codec): frugalAvailable for void func result structs by @xiaost in [#1938](https://github.com/cloudwego/kitex/pull/1938)
> 修复：void 函数 result 结构体的 `frugalAvailable` 判断
* fix(timeout): close ticker in rpctimeout pool to prevent resource leak by @DMwangnima in [#1931](https://github.com/cloudwego/kitex/pull/1931)
> 修复：关闭 rpctimeout pool 中的 ticker，避免资源泄漏
* fix(streaming): server-side old Stream wrapped in server MW should not be discarded by @DMwangnima in [#1929](https://github.com/cloudwego/kitex/pull/1929)
> 修复：Server 侧旧 Stream 被 Server Middleware 包装后不应被丢弃
* fix(generic): panic when generic writing different elem types of container by @DMwangnima in [#1926](https://github.com/cloudwego/kitex/pull/1926)
> 修复：泛化调用容器字段写入不同类型元素导致的 panic
* fix: remove streaming rpcstats Reset by @DMwangnima in [#1922](https://github.com/cloudwego/kitex/pull/1922)
> 修复：移除流式 `rpcStats.Reset()`
* fix(ttstream): add server-side information in ttstream errBizHandlerReturnCancel exception by @DMwangnima in [#1921](https://github.com/cloudwego/kitex/pull/1921)
> 修复：ttstream 在 `errBizHandlerReturnCancel` 异常中补充服务端信息

### Optimize
* perf(gRPC): reduce object allocations on the gRPC client side for unified cancel scenarios by @DMwangnima in [#1950](https://github.com/cloudwego/kitex/pull/1950)
> 优化：减少 gRPC client 统一 cancel 场景下的对象分配
* optimize(gRPC): support pooling HTTP2 framer write buffer to reduce idle connection memory by @DMwangnima in [#1944](https://github.com/cloudwego/kitex/pull/1944)
> 优化：支持 HTTP/2 framer write buffer 池化，降低空闲连接内存占用
* perf(server): use inline RPCInfo fields in LocalCaller by @xiaost in [#1940](https://github.com/cloudwego/kitex/pull/1940)
> 优化：LocalCaller 使用内联 RPCInfo 字段
* optimize(discovery): reduce object allocation in discovery event queue and support changing default capacity of queue by @DMwangnima in [#1939](https://github.com/cloudwego/kitex/pull/1939)
> 优化：减少服务发现事件队列对象分配，并支持修改队列默认容量
* perf: rpcinfo inline fields by @xiaost in [#1935](https://github.com/cloudwego/kitex/pull/1935)
> 优化：内联 rpcinfo 字段
* optimize: remove ttstream connection write goroutine to avoid Sender OOM by @DMwangnima in [#1917](https://github.com/cloudwego/kitex/pull/1917)
> 优化：移除 ttstream 连接独立 write goroutine，避免 Sender OOM

### Refactor
* refactor: use maphash instead of xxhash3 by @xiaost in [#1924](https://github.com/cloudwego/kitex/pull/1924)
> 重构：consistent-hash 使用 `hash/maphash` 替换 `xxhash3`

### Chore
* chore: update dependencies and add ErrRPCFinish back for compatibility by @DMwangnima in [#1953](https://github.com/cloudwego/kitex/pull/1953)
> chore：更新依赖，并恢复 `ErrRPCFinish` 以保持兼容性
* chore(queue): remove unused field tailVersion of Queue by @DMwangnima in [#1947](https://github.com/cloudwego/kitex/pull/1947)
> chore：移除 Queue 中未使用的 tailVersion 字段
* chore: improve bug report issue template by @xiaost in [#1941](https://github.com/cloudwego/kitex/pull/1941)
> chore：优化 bug report issue 模板
* chore(codec): log data type before errDecodeMismatchMsgType by @xiaost in [#1937](https://github.com/cloudwego/kitex/pull/1937)
> chore：在 `errDecodeMismatchMsgType` 之前记录数据类型
* chore(mux): deprecate thrift mux transport by @DMwangnima in [#1933](https://github.com/cloudwego/kitex/pull/1933)
> chore：标记 thrift mux transport 为废弃
* chore: change tests workflow on go 1.21-1.26 by @GuangmingLuo in [#1923](https://github.com/cloudwego/kitex/pull/1923)
> chore：更新测试 workflow 至 Go 1.21-1.26
* chore: update dependencies by @DMwangnima in [#1912](https://github.com/cloudwego/kitex/pull/1912)
> chore：更新依赖（sonic、dynamicgo、frugal）
* chore: release version v0.16.2 by @DMwangnima in [#1954](https://github.com/cloudwego/kitex/pull/1954)
> chore：发布 v0.16.2 版本
* chore: release version v0.16.1 by @DMwangnima in [#1919](https://github.com/cloudwego/kitex/pull/1919)
> chore：发布 v0.16.1 版本
* chore: release version v0.16.0 by @DMwangnima in [#1913](https://github.com/cloudwego/kitex/pull/1913)
> chore：发布 v0.16.0 版本

### Docs
* docs: add changelog by @xiaost in [#1946](https://github.com/cloudwego/kitex/pull/1946)
> docs：增加 changelog 文件
