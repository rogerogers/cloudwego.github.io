---
title: "StreamX 流式细粒度事件追踪"
linkTitle: "流式细粒度事件追踪"
weight: 6
date: 2026-05-08
keywords: ["流式监控", "StreamEventHandler", "流式细粒度追踪"]
description: "Kitex StreamX 流式细粒度事件追踪机制 StreamEventHandler，支持感知流式协议层面的核心 Event，便于定制细粒度的流式监控。"
---

## 背景

为了更方便地对流式接口做**细粒度**监控，从协议层面掌控一个流的**全生命周期**，Kitex 从 **v0.16.0** 开始提供全新的流式细粒度监控扩展机制 **StreamEventHandler**，支持感知流式协议层面的核心 Event（Stream 开始、Recv、Send、Recv Header、Stream 结束）。

与之前只能感知 Send/Recv 的 `StreamEventReporter` 相比，StreamEventHandler 提供了更完整的生命周期事件，便于实现监控、打点、元信息解析等扩展能力。

## 什么是 StreamEventHandler？

`StreamEventHandler` 是按 Client / Server 视角提供的一组事件回调，业务在感兴趣的事件上挂载回调函数，即可在流式 RPC 的协议层完成监控、打点、元信息解析等工作。

### 支持的 Event

| Event | 含义 | Client | Server |
| --- | --- | --- | --- |
| `StreamStartEvent` | 建流（Client：写出 Header Frame；Server：接收并解析 Header Frame） | ✅ | ✅ |
| `StreamRecvHeaderEvent` | 收到对端 Header Frame | ✅ | ❌ |
| `StreamRecvEvent` | 每次 `Stream.Recv` | ✅ | ✅ |
| `StreamSendEvent` | 每次 `Stream.Send` | ✅ | ✅ |
| `StreamFinishEvent` | 流结束 | ✅ | ✅ |

### Handler 类型定义

定义在 `github.com/cloudwego/kitex/pkg/rpcinfo` 包：

```go
// Client 侧
type ClientStreamEventHandler struct {
    HandleStreamStartEvent      func(ctx context.Context, ri RPCInfo, evt StreamStartEvent)
    HandleStreamRecvHeaderEvent func(ctx context.Context, ri RPCInfo, evt StreamRecvHeaderEvent)
    HandleStreamRecvEvent       func(ctx context.Context, ri RPCInfo, evt StreamRecvEvent)
    HandleStreamSendEvent       func(ctx context.Context, ri RPCInfo, evt StreamSendEvent)
    HandleStreamFinishEvent     func(ctx context.Context, ri RPCInfo, evt StreamFinishEvent)
}

// Server 侧（无 RecvHeader）
type ServerStreamEventHandler struct {
    HandleStreamStartEvent  func(ctx context.Context, ri RPCInfo, evt StreamStartEvent)
    HandleStreamRecvEvent   func(ctx context.Context, ri RPCInfo, evt StreamRecvEvent)
    HandleStreamSendEvent   func(ctx context.Context, ri RPCInfo, evt StreamSendEvent)
    HandleStreamFinishEvent func(ctx context.Context, ri RPCInfo, evt StreamFinishEvent)
}
```

业务**只需要填关心的事件字段**，其余留 `nil`，框架会自动跳过未注册的事件。

## 使用方式

### Server 侧

1. 实现 `ServerStreamEventHandler`，只填关心的事件字段：

```go
import (
    "context"

    "github.com/cloudwego/kitex/pkg/rpcinfo"
)

var serverHandler = rpcinfo.ServerStreamEventHandler{
    HandleStreamStartEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamStartEvent) {
        // 流建立时回调
    },
    HandleStreamRecvEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamRecvEvent) {
        // 每次 Recv 回调
    },
    HandleStreamSendEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamSendEvent) {
        // 每次 Send 回调
    },
    HandleStreamFinishEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamFinishEvent) {
        // 流结束时回调
    },
}
```

2. 通过 `server.WithStreamEventHandler` 注入：

```go
import (
    "github.com/cloudwego/kitex/server"
)

svr := testservice.NewServer(hdl,
    server.WithStreamOptions(server.WithStreamEventHandler(serverHandler)),
)
err := svr.Run()
```

### Client 侧

1. 实现 `ClientStreamEventHandler`：

```go
import (
    "context"

    "github.com/cloudwego/kitex/pkg/rpcinfo"
)

var clientHandler = rpcinfo.ClientStreamEventHandler{
    HandleStreamStartEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamStartEvent) {
        // 流建立时回调
    },
    HandleStreamRecvHeaderEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamRecvHeaderEvent) {
        // 收到对端 Header Frame
    },
    HandleStreamRecvEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamRecvEvent) {
        // 每次 Recv 回调
    },
    HandleStreamSendEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamSendEvent) {
        // 每次 Send 回调
    },
    HandleStreamFinishEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamFinishEvent) {
        // 流结束时回调
    },
}
```

2. 通过 `client.WithStreamEventHandler` 注入：

```go
import (
    "github.com/cloudwego/kitex/client"
)

// streamx Client
cli, err := testservice.NewClient("service-name",
    client.WithStreamOptions(client.WithStreamEventHandler(clientHandler)),
)
```

> 注：使用 gRPC 协议时，记得通过 `client.WithTransportProtocol(transport.GRPC)` 指定。

## 与旧机制 `StreamEventReporter` 的关系

旧的 `StreamEventReporter` 仍然兼容（不需要改动旧代码），但只能感知到 Send / Recv 两类事件。若需要感知 Start / Finish / RecvHeader 等更完整生命周期事件，请使用 StreamEventHandler。
