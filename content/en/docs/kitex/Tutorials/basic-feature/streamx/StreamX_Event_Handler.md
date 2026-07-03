---
title: "StreamX Detailed Stream Event Tracing"
linkTitle: "Detailed Stream Event Tracing"
weight: 6
date: 2026-05-08
keywords: ["Stream Monitoring", "StreamEventHandler", "Detailed Stream Tracing"]
description: "Kitex StreamX detailed stream event tracing via StreamEventHandler, allowing observation of core stream protocol events for fine-grained streaming monitoring."
---

## Background

To make fine-grained monitoring of streaming APIs easier and gain control over the **full lifecycle** of a stream at the protocol layer, Kitex provides a new fine-grained streaming monitoring extension mechanism **StreamEventHandler** starting from **v0.16.0**. It supports observing core events at the streaming protocol layer (Stream start, Recv, Send, Recv Header, Stream finish).

Compared to the previous `StreamEventReporter` which could only observe Send/Recv events, StreamEventHandler offers more complete lifecycle events, making it easier to implement monitoring, telemetry, and meta-info parsing extensions.

## What is StreamEventHandler?

`StreamEventHandler` is a set of event callbacks provided from both Client and Server perspectives. Business code can register callbacks on events of interest to complete monitoring, telemetry, and meta-info parsing at the streaming RPC protocol layer.

### Supported Events

| Event | Description | Client | Server |
| --- | --- | --- | --- |
| `StreamStartEvent` | Stream start (Client: writing Header Frame; Server: receiving and parsing Header Frame) | ✅ | ✅ |
| `StreamRecvHeaderEvent` | Header Frame received from peer | ✅ | ❌ |
| `StreamRecvEvent` | Each `Stream.Recv` | ✅ | ✅ |
| `StreamSendEvent` | Each `Stream.Send` | ✅ | ✅ |
| `StreamFinishEvent` | Stream finish | ✅ | ✅ |

### Handler Type Definitions

Defined in `github.com/cloudwego/kitex/pkg/rpcinfo`:

```go
// Client side
type ClientStreamEventHandler struct {
    HandleStreamStartEvent      func(ctx context.Context, ri RPCInfo, evt StreamStartEvent)
    HandleStreamRecvHeaderEvent func(ctx context.Context, ri RPCInfo, evt StreamRecvHeaderEvent)
    HandleStreamRecvEvent       func(ctx context.Context, ri RPCInfo, evt StreamRecvEvent)
    HandleStreamSendEvent       func(ctx context.Context, ri RPCInfo, evt StreamSendEvent)
    HandleStreamFinishEvent     func(ctx context.Context, ri RPCInfo, evt StreamFinishEvent)
}

// Server side (no RecvHeader)
type ServerStreamEventHandler struct {
    HandleStreamStartEvent  func(ctx context.Context, ri RPCInfo, evt StreamStartEvent)
    HandleStreamRecvEvent   func(ctx context.Context, ri RPCInfo, evt StreamRecvEvent)
    HandleStreamSendEvent   func(ctx context.Context, ri RPCInfo, evt StreamSendEvent)
    HandleStreamFinishEvent func(ctx context.Context, ri RPCInfo, evt StreamFinishEvent)
}
```

You **only need to set the event fields you care about**; leave the rest as `nil` and the framework will skip unregistered events automatically.

## Usage

### Server Side

1. Implement `ServerStreamEventHandler` with only the events you care about:

```go
import (
    "context"

    "github.com/cloudwego/kitex/pkg/rpcinfo"
)

var serverHandler = rpcinfo.ServerStreamEventHandler{
    HandleStreamStartEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamStartEvent) {
        // Called when a stream is established
    },
    HandleStreamRecvEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamRecvEvent) {
        // Called on every Recv
    },
    HandleStreamSendEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamSendEvent) {
        // Called on every Send
    },
    HandleStreamFinishEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamFinishEvent) {
        // Called when the stream ends
    },
}
```

2. Inject it via `server.WithStreamEventHandler`:

```go
import (
    "github.com/cloudwego/kitex/server"
)

svr := testservice.NewServer(hdl,
    server.WithStreamOptions(server.WithStreamEventHandler(serverHandler)),
)
err := svr.Run()
```

### Client Side

1. Implement `ClientStreamEventHandler`:

```go
import (
    "context"

    "github.com/cloudwego/kitex/pkg/rpcinfo"
)

var clientHandler = rpcinfo.ClientStreamEventHandler{
    HandleStreamStartEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamStartEvent) {
        // Called when a stream is established
    },
    HandleStreamRecvHeaderEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamRecvHeaderEvent) {
        // Header Frame received from peer
    },
    HandleStreamRecvEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamRecvEvent) {
        // Called on every Recv
    },
    HandleStreamSendEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamSendEvent) {
        // Called on every Send
    },
    HandleStreamFinishEvent: func(ctx context.Context, ri rpcinfo.RPCInfo, evt rpcinfo.StreamFinishEvent) {
        // Called when the stream ends
    },
}
```

2. Inject it via `client.WithStreamEventHandler`:

```go
import (
    "github.com/cloudwego/kitex/client"
)

// streamx Client
cli, err := testservice.NewClient("service-name",
    client.WithStreamOptions(client.WithStreamEventHandler(clientHandler)),
)
```

> Note: when using the gRPC protocol, remember to specify it via `client.WithTransportProtocol(transport.GRPC)`.

## Relationship with Legacy `StreamEventReporter`

The legacy `StreamEventReporter` is still compatible (no changes required to existing code), but it can only observe Send / Recv events. To observe more complete lifecycle events such as Start / Finish / RecvHeader, please use StreamEventHandler.
