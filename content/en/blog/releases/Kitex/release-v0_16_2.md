---
title: "Kitex Release v0.16.2"
linkTitle: "Release v0.16.2"
projects: ["Kitex"]
date: 2026-05-08
description: >
---

> These release notes consolidate the changes from v0.16.0, v0.16.1, and v0.16.2. The externally announced version is v0.16.2.

## **Introduction to Key Changes**

### **Announcements**
1. **netpollmux is no longer maintained**: The `pkg/remote/trans/netpollmux` package and the corresponding `client.WithMuxConnection` / `server.WithMuxTransport` options are all marked as deprecated. netpollmux is no longer maintained. For details, see [Connection Multiplexing](/docs/kitex/tutorials/basic-feature/connection_type#connection-multiplexing).
2. **Adjustments to some interfaces**: No impact on regular users, but may affect those with extensions or special dependencies. For details, see the [**Special Changes**] section.

### **New Features**
1. **Binary Generic Call: Per-Request IDL Service Name, Greatly Reducing the Number of Generic Clients to Maintain**

   In previous versions, binary generic clients were bound to an IDL Service, requiring maintenance of a large number of clients. This version supports dynamically specifying the IDL Service Name per call, via the newly added `callopt.WithBinaryGenericIDLService(svcName)` / `streamcall.WithBinaryGenericIDLService(svcName)` call options, and the per-call configuration takes precedence over the configuration set at client initialization. Both Ping-Pong and streaming calls are supported. For details, see [Per-Call IDL Service Name](/docs/kitex/tutorials/advanced-feature/generic-call/basic_usage#per-call-idl-service-name).

2. **Recv Timeout Control for Streaming**

   Added `streaming.TimeoutConfig` for fine-grained Recv timeout control on streaming APIs. A dedicated timeout can be configured, and the new `DisableCancelRemote` flag controls whether the remote stream is cascaded-cancelled after timeout. Two configuration entry points are provided:
   - Per-client: `client.WithStreamRecvTimeoutConfig(streaming.TimeoutConfig)`
   - Per-call: `streamcall.WithRecvTimeoutConfig(streaming.TimeoutConfig)`

   When a Recv times out, a new error code `codes.RecvDeadlineExceeded` (value `17`) is returned together with the sentinel error `kerrors.ErrStreamingTimeout`, making timeout classification easier.

3. **Fine-Grained Streaming Event Tracing - StreamEventHandler**

   A new event-callback mechanism, independent of the Tracer, can observe the core events of the streaming protocol layer (Stream start, Recv, Send, Recv Header, Stream finish), making it easy to build custom fine-grained streaming monitoring:
   - Client: `client.WithStreamEventHandler(rpcinfo.ClientStreamEventHandler)`
   - Server: `server.WithStreamEventHandler(rpcinfo.ServerStreamEventHandler)`

   New event types `stats.StreamStart`, `stats.StreamRecvHeader`, and `stats.StreamFinish` are also added. For details, see [StreamX Detailed Stream Event Tracing](/docs/kitex/tutorials/basic-feature/streamx/streamx_event_handler).

### **Feature/Performance Optimization**
1. **Kitex gRPC: Memory Optimization and Connection Leak Fixes**
   - **HTTP/2 Write Buffer Reuse and Framer-Level Pooling**: Supports per-connection pooling and reuse of write buffers, reducing memory usage for idle connections, suitable for scenarios where the service needs to directly handle a large number of gRPC connections. Use `client.WithGRPCReuseWriteBuffer` / `server.WithGRPCReuseWriteBuffer` to enable it, and further enable framer-level pooling via `ReuseWriteBufferConfig.EnableReuseHTTP2FramerBuffer`.
   - **Client-Side Cancel Object Allocation Optimization**: Reduces object allocations for unified cancel on the gRPC client side, avoiding excessive allocations in gateway scenarios where cancel operations are frequent.
   - **Connection Pool Leak Fix**: Fixed an issue where gRPC connections were not properly recycled after being closed with no subsequent calls.

2. **TTHeader Streaming: Memory Optimization**

   The sender now directly reuses the underlying TCP flow control, avoiding OOM.

3. **Service Discovery: Instance Change Event Allocation Optimization**

   Reduces object allocations caused by frequent downstream instance changes, lowering GC pressure. Also added `event.GetDefaultEventNum()` / `event.SetDefaultEventNum(num int)` for tuning the default event queue capacity (default 200).

4. **RPCInfo Field Inlining**

   Added `rpcinfo.NewRPCInfoWithInlineFields() RPCInfo`, which returns an RPCInfo with inlined sub-objects, reducing per-request pool gets and allocations. Server hot paths adopt this inlining approach, improving request handling performance.

### **Bug Fixes**
1. **Streaming-Related Fixes**
   - **Recv/Send Panic on the Streaming Server Side**: Streaming RPCs no longer reuse `rpcinfo`, completely avoiding concurrent read/write panics on `rpcinfo` caused by the handler exiting early while the Stream is still being used asynchronously.
   - **Legacy Server Middleware Extension Not Taking Effect**: Fixed an issue where the extended Stream object in legacy Server Middleware was unexpectedly discarded.

2. **Other Fixes**
   - **rpcTimeout Ticker Leak Fix** (extremely rare): Closed the ticker in the `rpcTimeout` pool to prevent resource leaks. Most online scenarios are unaffected; this issue is only noticeable in scenarios with extremely low QPS and short processing time for the API.
   - **Panic Caused by Writing Elements of Different Types into Container Fields in Generic Calls** (very rare): Writing elements of different types into the same container field could cause a panic. For example, if the field itself is `[]uint8`, the first input may be `uint8` while the second input may be `string`. After the fix, an error is returned instead of panic.

### **Special Changes - May Affect a Small Number of Services**
> Mainly Breaking Changes and API deprecations. No impact on the vast majority of users; users with special dependencies should pay attention.

#### Breaking Changes

1. **`pkg/remote/trans/ttstream/container` Path Migration** (#1952)

   Moved to `pkg/remote/trans/ttstream/internal/container`. The package was not intended as a public API; copy the needed implementations into your own module if you depended on them.

2. **`grpc.NewClientTransport` Callback Timing Change** (#1945)

   The `onClose` / `onGoAway` callbacks now fire after the transport transitions to closing/draining and after `http2Client.mu` is released. Callers relying on the old ordering should migrate to `grpc.NewClientTransportWithConfig`.

3. **`consistBalancer` Consistent-Hash Algorithm Switched** (#1924)

   The consistent-hash key/node hashing in `consistBalancer` replaces `github.com/bytedance/gopkg/util/xxhash3` with `hash/maphash`. Because `hash/maphash` uses a per-process random seed, hash values **differ across client replicas and restarts**.

#### Deprecations
> APIs are only marked as deprecated in this version; they still work, but please migrate to the new APIs at your earliest convenience.

1. **netpollmux Fully Deprecated** (#1933)
   - `client.WithMuxConnection(connNum int)` is deprecated
   - `server.WithMuxTransport()` is deprecated
   - The `pkg/remote/trans/netpollmux` package itself is marked as deprecated and no longer maintained

   See [Connection Multiplexing](/docs/kitex/tutorials/basic-feature/connection_type#connection-multiplexing) for the rationale.

2. **`kerrors.ErrRPCFinish` Restored as Deprecated Symbol** (#1953)

   This API was removed in v0.15.0. v0.16.2 restores it as a deprecated symbol for backward compatibility with pre-v0.15 code.

3. **Streaming-Related API Deprecations**
   - `client.WithStreamRecvTimeout` is deprecated; use `client.WithStreamRecvTimeoutConfig` instead (#1911)
   - `streamcall.WithRecvTimeout` is deprecated; use `streamcall.WithRecvTimeoutConfig` instead (#1911)

## **Full Change**

### Feature
* feat(generic): support specifying IDL Service Name per call for Binary Generic by @DMwangnima in [#1928](https://github.com/cloudwego/kitex/pull/1928)
* feat(server): add in-process LocalCaller for unary calls by @xiaost in [#1930](https://github.com/cloudwego/kitex/pull/1930)
* feat: gRPC supports reusing write buffer for each connection by @DMwangnima in [#1918](https://github.com/cloudwego/kitex/pull/1918)
* feat(streaming): add Recv timeout config and adjust gRPC error/log by @DMwangnima in [#1911](https://github.com/cloudwego/kitex/pull/1911)
* feat(streaming): support detailed tracing events by @DMwangnima in [#1905](https://github.com/cloudwego/kitex/pull/1905)
* feat(streaming): remove rpcinfo reuse for streaming by @DMwangnima in [#1909](https://github.com/cloudwego/kitex/pull/1909)

### Fix
* fix(ttstream): ttstream should not recycle connection when Recv timeout with DisableCancelRemote=true by @DMwangnima in [#1952](https://github.com/cloudwego/kitex/pull/1952)
* fix(gRPC): connection pool leak when connection is closed and there are no more subsequent calls by @DMwangnima in [#1945](https://github.com/cloudwego/kitex/pull/1945)
* fix(codec): frugalAvailable for void func result structs by @xiaost in [#1938](https://github.com/cloudwego/kitex/pull/1938)
* fix(timeout): close ticker in rpctimeout pool to prevent resource leak by @DMwangnima in [#1931](https://github.com/cloudwego/kitex/pull/1931)
* fix(streaming): server-side old Stream wrapped in server MW should not be discarded by @DMwangnima in [#1929](https://github.com/cloudwego/kitex/pull/1929)
* fix(generic): panic when generic writing different elem types of container by @DMwangnima in [#1926](https://github.com/cloudwego/kitex/pull/1926)
* fix: remove streaming rpcstats Reset by @DMwangnima in [#1922](https://github.com/cloudwego/kitex/pull/1922)
* fix(ttstream): add server-side information in ttstream errBizHandlerReturnCancel exception by @DMwangnima in [#1921](https://github.com/cloudwego/kitex/pull/1921)

### Optimize
* perf(gRPC): reduce object allocations on the gRPC client side for unified cancel scenarios by @DMwangnima in [#1950](https://github.com/cloudwego/kitex/pull/1950)
* optimize(gRPC): support pooling HTTP2 framer write buffer to reduce idle connection memory by @DMwangnima in [#1944](https://github.com/cloudwego/kitex/pull/1944)
* perf(server): use inline RPCInfo fields in LocalCaller by @xiaost in [#1940](https://github.com/cloudwego/kitex/pull/1940)
* optimize(discovery): reduce object allocation in discovery event queue and support changing default capacity of queue by @DMwangnima in [#1939](https://github.com/cloudwego/kitex/pull/1939)
* perf: rpcinfo inline fields by @xiaost in [#1935](https://github.com/cloudwego/kitex/pull/1935)
* optimize: remove ttstream connection write goroutine to avoid Sender OOM by @DMwangnima in [#1917](https://github.com/cloudwego/kitex/pull/1917)

### Refactor
* refactor: use maphash instead of xxhash3 by @xiaost in [#1924](https://github.com/cloudwego/kitex/pull/1924)

### Chore
* chore: update dependencies and add ErrRPCFinish back for compatibility by @DMwangnima in [#1953](https://github.com/cloudwego/kitex/pull/1953)
* chore(queue): remove unused field tailVersion of Queue by @DMwangnima in [#1947](https://github.com/cloudwego/kitex/pull/1947)
* chore: improve bug report issue template by @xiaost in [#1941](https://github.com/cloudwego/kitex/pull/1941)
* chore(codec): log data type before errDecodeMismatchMsgType by @xiaost in [#1937](https://github.com/cloudwego/kitex/pull/1937)
* chore(mux): deprecate thrift mux transport by @DMwangnima in [#1933](https://github.com/cloudwego/kitex/pull/1933)
* chore: change tests workflow on go 1.21-1.26 by @GuangmingLuo in [#1923](https://github.com/cloudwego/kitex/pull/1923)
* chore: update dependencies by @DMwangnima in [#1912](https://github.com/cloudwego/kitex/pull/1912)
* chore: release version v0.16.2 by @DMwangnima in [#1954](https://github.com/cloudwego/kitex/pull/1954)
* chore: release version v0.16.1 by @DMwangnima in [#1919](https://github.com/cloudwego/kitex/pull/1919)
* chore: release version v0.16.0 by @DMwangnima in [#1913](https://github.com/cloudwego/kitex/pull/1913)

### Docs
* docs: add changelog by @xiaost in [#1946](https://github.com/cloudwego/kitex/pull/1946)
