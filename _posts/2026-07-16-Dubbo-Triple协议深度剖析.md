---
layout: post
title: "Dubbo Triple 协议深度剖析 —— 从 HTTP/2 到 HTTP/3 的云原生 RPC 协议"
author: "Inela"
---

你用 curl 调用过 Dubbo 服务吗？

在 Dubbo 2 的世界里，这不可能——因为 Dubbo 2 用的是私有 TCP 协议，只有 Dubbo SDK 才能解析。但在 Dubbo 3 里，一行 curl 就能调通：

```bash
curl -X POST http://localhost:50051/com.example.GreeterService/sayHello \
  -H "Content-Type: application/json" \
  -d '["Dubbo"]'
```

这背后的答案就是 **Triple 协议**——Dubbo 3 重新设计的新一代 RPC 通信协议，基于 HTTP/2，完全兼容 gRPC，一套协议打通 Dubbo、gRPC、HTTP 三种生态。

本文从协议规范、源码实现、通信模式、性能基准到 HTTP/3 演进，全方位拆解 Triple 协议的设计与实现。

---

## 一、为什么需要 Triple 协议？

### 1.1 Dubbo 2 协议的时代局限

Dubbo 2 的默认协议（`dubbo://`）是一种**私有 TCP 二进制协议**，协议头设计如下：

```
┌──────────────────────────────────────────────────────────────────┐
│                     Dubbo 2 协议帧结构（Header）                      │
│                                                                  │
│   0-1   byte: Magic (0xdabb)                                     │
│   2     byte: Req/Res + Seria + Twoway + Event 标志              │
│   3     byte: Status                                             │
│   4-11  long: Request ID                                         │
│   12-15 int : Body Length                                        │
│                                                                  │
│   设计目标：极致性能，最小协议头（16 字节），面向内部数据中心网络        │
└──────────────────────────────────────────────────────────────────┘
```

这个协议是为**内部局域网、纯 Java 生态、高性能点对点调用**设计的。在 2011 年那个年代，这是毫无疑问的正确选择。但在云原生时代，它的局限性逐一暴露：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Dubbo 2 协议的 5 大局限                              │
│                                                                     │
│  ┌─────────────────┐                                                │
│  │ 1. 私有协议       │  无法与生态工具集成（curl/Postman/浏览器不可用）    │
│  │    调试困难       │  抓包出来是二进制，没有可读性，排查问题效率低         │
│  └─────────────────┘                                                │
│  ┌─────────────────┐                                                │
│  │ 2. 网关穿透困难   │  私有协议需要专用网关解析，无法利用 HTTP 网关的       │
│  │                  │  鉴权、限流、路由等成熟能力                         │
│  └─────────────────┘                                                │
│  ┌─────────────────┐                                                │
│  │ 3. 跨语言互调    │  Dubbo 2 协议没有 Go/Node.js/Python 的标准实现，    │
│  │    几乎不可能    │  虽然社区有三方实现，但完整度和兼容性参差不齐          │
│  └─────────────────┘                                                │
│  ┌─────────────────┐                                                │
│  │ 4. Service Mesh  │  Istio/Envoy 基于 HTTP/gRPC，Dubbo 2 协议        │
│  │    不兼容       │  需要额外的协议转换适配层                            │
│  └─────────────────┘                                                │
│  ┌─────────────────┐                                                │
│  │ 5. 无流式通信    │  仅支持 Request-Response，不支持 Server Stream、    │
│  │    能力          │  Client Stream、Bidirectional Stream            │
│  └─────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 为什么不直接使用 gRPC？

你可能会问：既然需要 HTTP/2 + Protobuf + 跨语言，直接用 gRPC 不就行了？Dubbo 官方不是没考虑过这个方案，但最终选择了"重新实现一遍 gRPC"——通过 Triple 协议。理由有六个：

| # | 理由 | 详细说明 |
|---|------|---------|
| 1 | **gRPC 不支持浏览器/HTTP API** | 原生 gRPC 需要 grpc-web 代理才能从浏览器调用，需要 grpc-gateway 才能暴露 REST API。Triple 原生支持 `application/json`，curl 和浏览器直接调 |
| 2 | **gRPC 强制绑定 IDL** | gRPC 必须写 `.proto` 文件 → 编译生成 stub → 才能用。Triple 还支持 **Java Interface + POJO** 直接定义服务（零 IDL 成本），兼容 Dubbo 2 用户的开发习惯 |
| 3 | **gRPC 难以调试** | gRPC 的二进制帧无法用 curl 直接访问。Triple 协议下，`curl -X POST ... -d '["hello"]'` 就能调通，配合 `jq` 格式化输出，体验不输 REST API |
| 4 | **gRPC 代码量巨大** | gRPC-Java 官方库超过 10 万行代码，依赖复杂；Triple 协议实现仅**几千行**，排查问题和维护成本低得多 |
| 5 | **gRPC 自研 HTTP/2 库** | 如 grpc-go 自己实现 HTTP/2，而非使用 Go 官方库。这导致其与语言生态的标准 HTTP 组件不兼容。Dubbo 使用 Netty（Java 官方推荐的网络库）实现 HTTP/2 |
| 6 | **服务治理融合** | gRPC 的服务发现、负载均衡需要在 Dubbo 体系外单独配置。Triple 的源码由 Dubbo 自己掌控，与 Dubbo 的服务发现、负载均衡、流量管控、降级限流无缝融合 |

第七个理由藏在更深的地方：**gRPC 绑定 Protobuf，而 Protobuf 绑定 gRPC**——如果你用 Dubbo 协议 + Protobuf 序列化（不走 gRPC），性能反而比 Triple + Protobuf 更高（见第八节 BenchMark 数据）。所以 Dubbo 需要一种"能用到 Protobuf 的跨语言能力，但不被 gRPC 生态锁死"的方案。

### 1.3 Triple 的设计目标

Triple 协议的设计目标可以总结为三个关键词：

```
┌──────────────────────────────────────────────────────────────────┐
│                    Triple 设计目标的"不可能三角"                      │
│                                                                  │
│               gRPC 兼容 ──────────────────────── 高性能            │
│                    │                            │                │
│                    │         Triple             │                │
│                    │        协议设计              │                │
│                    │                            │                │
│                    └────────────┬───────────────┘                │
│                                 │                                │
│                          人机友好                               │
│                     (curl/浏览器可访问)                            │
└──────────────────────────────────────────────────────────────────┘
```

- **人机友好**：基于 HTTP，curl、浏览器、Postman 直接可调，开发者体验对标 REST API
- **gRPC 完全兼容**：100% 互操作，Dubbo Client ↔ gRPC Server 双向互通，不改一行代码
- **标准 HTTP 依赖**：仅依赖 HTTP 标准特性，使用语言官方 HTTP 库（Java 用 Netty，Go 用官方 HTTP 库），不做私有的 HTTP/2 实现

---

## 二、协议规范：基于 HTTP 的 RPC 协议定义

Triple 协议的完整规范由两部分组成——分别应对"简单 RPC"和"高级流式"两种场景。

### 2.1 协议整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                      Triple 协议分层架构                            │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                  应用层 / IDL 层                              ││
│  │   Java Interface + POJO    │   Protobuf IDL + Stub          ││
│  │   (零 IDL, Wrapper 模式)    │   (标准 IDL, 跨语言)             ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                  Triple 协议层                                ││
│  │   请求/响应编解码、超时控制、流控、序列化方式协商                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────┬──────────────────────────────────┐│
│  │   Part 1: HTTP RPC       │  Part 2: Extended gRPC          ││
│  │   (Unary 模式)            │  (Streaming 模式)                ││
│  │   HTTP/1.1 + HTTP/2      │  HTTP/2 Only                    ││
│  │   application/json       │  application/grpc                ││
│  │   application/proto      │  application/grpc+proto          ││
│  │                          │  application/grpc+json           ││
│  └──────────────────────────┴──────────────────────────────────┘│
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                  HTTP/2 传输层                                ││
│  │   Netty Http2FrameCodec + Http2MultiplexHandler             ││
│  │   (多路复用、流控、帧优先级)                                     ││
│  └─────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Part 1 — HTTP RPC 协议（Unary 模式）

Part 1 是 Triple 对"简单 RPC"的定义，同时支持 HTTP/1.1 和 HTTP/2，核心规则只有几条：

**（1）URL 路径规范**

```
POST /{Service-Name}/{Method-Name}
```

例如：`POST /com.example.GreeterService/sayHello`

**（2）支持的 Content-Type**

| Content-Type | 说明 |
|---|---|
| `application/json` | JSON 格式，请求体为标准 JSON 数组，元素按方法参数顺序排列 |
| `application/proto` | Protobuf 二进制格式，请求体为序列化后的 Protobuf 消息 |

**（3）自定义 Header**

Triple 通过 HTTP Header 传递 Dubbo 的服务治理参数，所有自定义 Header 以 `tri-` 为前缀：

```
┌────────────────────────────┬───────────────────────────────────────┐
│ Header                      │ 说明                                  │
├────────────────────────────┼───────────────────────────────────────┤
│ tri-protocol-version       │ Triple 协议版本，当前为 1.0.0            │
│ tri-service-version        │ Dubbo 服务版本号，如 1.0.0              │
│ tri-service-group          │ Dubbo 服务分组，如 demo                 │
│ tri-service-timeout        │ 超时时间（毫秒），如 3000                 │
│ tri-trace-traceid          │ 链路追踪 Trace ID                      │
│ tri-trace-rpcid            │ 链路追踪 RPC ID                        │
│ tri-unit-info              │ 单元化信息                             │
│ Content-Encoding           │ 压缩算法：gzip / br / zstd             │
└────────────────────────────┴───────────────────────────────────────┘
```

**（4）请求/响应格式**

```
┌──────────────────────────────────────────────────────────────────┐
│  Part 1 (Unary) 请求 = POST /ServiceName/MethodName              │
│                                                                  │
│  HTTP Headers:                                                    │
│    Content-Type: application/json                                │
│    tri-service-version: 1.0.0                                    │
│    tri-service-group: demo                                       │
│    tri-service-timeout: 3000                                     │
│                                                                  │
│  HTTP Body (JSON 模式):                                           │
│    ["arg0_value", "arg1_value", "arg2_value"]                    │
│                                                                  │
│  HTTP Body (Protobuf 模式):                                        │
│    [protobuf_wire_format_bytes...]                               │
└──────────────────────────────────────────────────────────────────┘
```

JSON 请求体是一个**有序数组**，元素按方法参数声明的顺序排列。这意味着你甚至可以用 curl 调试 Protobuf 定义的服务（如果开启了 `application/json` 支持）。

**（5）压缩支持**

Triple 通过标准 HTTP `Content-Encoding` Header 声明压缩算法：

```
客户端发送带压缩的请求:
  POST /GreeterService/sayHello
  Content-Encoding: gzip
  [gzip 压缩后的 body]

服务端返回带压缩的响应:
  HTTP/1.1 200 OK
  Content-Encoding: zstd
  [zstd 压缩后的 body]
```

注意：压缩是针对整个 HTTP Body 的，不是针对单个字段，这与 Dubbo 2 协议在协议头中指定序列化 ID 的设计完全不同——Triple 更依赖标准 HTTP 语义。

### 2.3 Part 2 — 扩展 gRPC 协议（Streaming 模式）

Part 2 是针对**流式通信**的扩展协议，**仅在 HTTP/2 上可用**（因为流式依赖 HTTP/2 的多 Stream 模型）。它完全兼容 gRPC 协议规范，并新增了 Dubbo 的扩展能力。

**（1）Content-Type 体系**

| Content-Type | 说明 |
|---|---|
| `application/grpc` | 标准 gRPC 协议，Protobuf 序列化 |
| `application/grpc+proto` | 同 `application/grpc`，明确声明 Protobuf |
| `application/grpc+json` | gRPC JSON 模式 |
| `application/triple+wrapper` | **Dubbo 扩展**，Triple Wrapper 模式（兼容 Java POJO） |

其中 `application/triple+wrapper` 是 Dubbo 的独有扩展——用于传递 `TripleRequestWrapper` / `TripleResponseWrapper` 包装的非 Protobuf 数据。

**（2）HTTP/2 帧级别的消息流**

gRPC 协议将一次 RPC 调用映射为一个 HTTP/2 Stream：

```
┌──────────────────────────────────────────────────────────────────┐
│              一次 gRPC/Streaming 调用的 HTTP/2 帧序列              │
│                                                                  │
│  Client                                    Server               │
│    │                                         │                   │
│    │ ──── HEADERS Frame (Stream ID=N) ────► │ 请求头              │
│    │     :method = POST                     │                   │
│    │     :path = /ServiceName/MethodName    │                   │
│    │     content-type = application/grpc    │                   │
│    │     grpc-timeout = 3000m               │                   │
│    │                                         │                   │
│    │ ──── DATA Frame ────────────────────► │ ┌─ 压缩标记:1 byte  │
│    │     Length-Prefixed-Message            │ │  消息长度: 4 bytes│
│    │     (5字节头 + Payload)               │ └─ Payload         │
│    │                                         │                   │
│    │ ──── DATA Frame (多次) ──────────►    │ 多条消息            │
│    │                                         │                   │
│    │ ──── DATA Frame (EndStream=true) ──►  │ 客户端发送完毕       │
│    │                                         │                   │
│    │ ◄─── HEADERS Frame ────────────────    │ 响应头              │
│    │     :status = 200                      │                   │
│    │     content-type = application/grpc    │                   │
│    │                                         │                   │
│    │ ◄─── DATA Frame ──────────────────    │ 响应数据            │
│    │     Length-Prefixed-Message            │                   │
│    │                                         │                   │
│    │ ◄─── HEADERS Frame (Trailers) ────    │                     │
│    │     grpc-status = 0                    │ 调用成功            │
│    │     grpc-message = "OK"               │                   │
│    │     EndStream=true                     │ 流结束              │
└──────────────────────────────────────────────────────────────────┘
```

每个 DATA 帧中的消息格式为（gRPC 标准）：

```
┌────────────────────────────────────────┐
│ 压缩标记 (1 byte) │ 长度 (4 bytes, BE) │  → Payload (序列化后的消息体)
│                    ──────────────────  │
│                    Length-Prefixed     │
│                    Message Header      │
│                    (共 5 bytes)        │
└────────────────────────────────────────┘
```

- **压缩标记**：`0` = 未压缩，`1` = 已压缩
- **长度**：4 字节大端无符号整数，表示 Payload 的字节数

### 2.4 与 gRPC 的 100% 兼容性——互操作矩阵

Triple 协议完整实现了 gRPC 的核心规范，包括：

- Protocol Buffers 序列化
- HTTP/2 传输语义
- gRPC 帧格式（含 `grpc-timeout` Header 编解码）
- gRPC Health Checking Protocol（`HealthStatusManager`）
- Streaming / Trailers / Error Details 全部特性

互操作矩阵如下：

```
┌────────────────────────────────────────────────────────────────────┐
│                    Triple 协议互操作矩阵                              │
│                                                                    │
│  ┌─────────────────┐          ┌─────────────────┐                  │
│  │  Dubbo Client   │ ──────► │  gRPC Server    │  ✅ 正常调用      │
│  │  (triple协议)    │          │  (标准gRPC服务)   │                  │
│  └─────────────────┘          └─────────────────┘                  │
│                                                                    │
│  ┌─────────────────┐          ┌─────────────────┐                  │
│  │  gRPC Client    │ ──────► │  Dubbo Server   │  ✅ 正常调用      │
│  │  (标准gRPC客户端) │          │  (triple协议)    │                  │
│  └─────────────────┘          └─────────────────┘                  │
│                                                                    │
│  ┌─────────────────┐          ┌─────────────────┐                  │
│  │  curl / 浏览器   │ ──────► │  Dubbo Server   │  ✅ 正常调用      │
│  │  (HTTP/1.1)     │          │  (triple协议)    │                  │
│  └─────────────────┘          └─────────────────┘                  │
│                                                                    │
│  ┌─────────────────┐          ┌─────────────────┐                  │
│  │  Dubbo Client   │ ──────► │  Dubbo Server   │  ✅ 正常调用      │
│  │  (triple协议)    │          │  (triple协议)    │  (原生)          │
│  └─────────────────┘          └─────────────────┘                  │
└────────────────────────────────────────────────────────────────────┘
```

这意味着一个 Dubbo Server **同一端口同时支持 Triple、gRPC、HTTP/1.1 三种协议**，所有请求最终路由到相同的业务逻辑实现。

---

## 三、核心实现：源码级拆解

下面以 Dubbo Java SDK 实现为参考，拆解 Triple 协议的核心类、ChannelPipeline 和编解码链路。

### 3.1 TripleProtocol：服务暴露与服务引用

Triple 协议的入口是 `TripleProtocol`（继承自 `AbstractProtocol`），围绕 `export()` 和 `refer()` 两个核心方法展开。

**服务暴露（export）**

```
TripleProtocol#export()
  │
  ├─► 1. 封装 Invoker → Exporter（管理服务生命周期）
  │
  ├─► 2. 注册路径映射到 PathResolver
  │      类似于 Spring MVC 的 RequestMappingHandlerMapping
  │      key = "/ServiceName/MethodName"
  │      value = Invoker
  │      这样收到 HTTP 请求后，解析 :path 即可定位到目标方法
  │
  ├─► 3. 设置服务状态：HealthStatusManager 更新为 "SERVING"
  │      遵循 gRPC Health Checking Protocol
  │
  ├─► 4. 初始化服务端线程池（per port，默认 200 核心线程）
  │
  └─► 5. PortUnificationExchanger#bind() 开启 Netty 服务端
         同一端口同时监听 HTTP/1.1 和 HTTP/2（通过 ALPN 协商）
```

**服务引用（refer）**

```
TripleProtocol#refer()
  │
  ├─► 1. 创建 TripleInvoker 对象
  │
  ├─► 2. Connection#create() 构建 Netty Bootstrap
  │      │
  │      └─► TripleHttp2Protocol#configClientPipeline()
  │           配置 HTTP/2 的 ChannelPipeline
  │
  └─► 3. Connection#connect() 建立物理连接（Lazy，首次调用时建立）
```

一个 `TripleInvoker` 对应一个 `Connection` 对象（一个 Socket 连接），而 `ConnectionManager` 按服务地址缓存连接——**同一 IP:Port 的不同接口共享同一个 TCP 连接**。

### 3.2 ChannelPipeline 配置

Triple 基于 Netty 的 HTTP/2 实现，Pipeline 设计为**两层架构**：

**父 Channel（TCP 连接级别）**

```
┌──────────────────────────────────────────────────────────┐
│  父 Channel Pipeline (每个 TCP 连接一个)                    │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │ SslClientTlsHandler (可选，TLS 加密)             │     │
│  │ 根据配置决定使用 TLS 还是 Plaintext               │     │
│  ├────────────────────────────────────────────────┤     │
│  │ Http2FrameCodec                                │     │
│  │ HTTP/2 Frame 编解码器                          │     │
│  │ 负责: ByteBuf ↔ Http2Frame                     │     │
│  │ 屏蔽 Frame 细节（HEADERS/DATA/PRIORITY/RST_STREAM│     │
│  ├────────────────────────────────────────────────┤     │
│  │ Http2MultiplexHandler                         │     │
│  │ HTTP/2 多路复用支持                             │     │
│  │ 为每个 Stream 创建子 Channel                    │     │
│  │ 将父 Channel 事件传播到对应子 Channel            │     │
│  ├────────────────────────────────────────────────┤     │
│  │ TripleClientHandler                           │     │
│  │ 连接级别的客户端处理                            │     │
│  ├────────────────────────────────────────────────┤     │
│  │ TripleTailHandler                             │     │
│  │ 末尾 Handler，防止内存泄漏                      │     │
│  └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**子 Channel（Stream 级别）**

```
┌──────────────────────────────────────────────────────────┐
│  子 Channel Pipeline (每个 HTTP/2 Stream 一个)              │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │ TripleCommandOutBoundHandler                   │     │
│  │ 命令输出：将响应封装为 HTTP/2 Frame 写出          │     │
│  ├────────────────────────────────────────────────┤     │
│  │ TripleHttp2ClientResponseHandler               │     │
│  │ 客户端响应处理：读取 Frame → 反序列化 → 回调业务   │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  服务端对应的 Handler:                                    │
│  ┌────────────────────────────────────────────────┐     │
│  │ TripleHttp2FrameServerHandler#channelRead()    │     │
│  │ → onHeadersRead(): 解析 Header，定位 Invoker    │     │
│  │ → onDataRead(): 解析 Data，反序列化请求体       │     │
│  └────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

这里的关键设计是 `Http2MultiplexHandler`——它为每个 HTTP/2 Stream 创建一个虚拟的子 Channel，这样每个 RPC 调用都有独立的消息处理器，天然支持**多路复用**。

```
┌──────────────────────────────────────────────────────────────────┐
│                HTTP/2 多路复用示意                                  │
│                                                                  │
│  ┌──────────────────┐    一个 TCP 连接                             │
│  │ 父 Channel        │                                            │
│  │ (192.168.1.1:50051)                                           │
│  │                  │                                            │
│  │  ┌────────────────┐  Stream ID=1  →  子 Channel-1  (sayHello) │
│  │  │Http2Multiplex  │  Stream ID=3  →  子 Channel-3  (getOrder) │
│  │  │    Handler     │  Stream ID=5  →  子 Channel-5  (listUser) │
│  │  └────────────────┘  Stream ID=7  →  子 Channel-7  (queryLog) │
│  │                                                                 │
│  │  同一连接上，不同接口、不同方法、不同请求共享一条 TCP 连接            │
│  │  每个 Stream 独立收发，互不阻塞                                    │
│  └──────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### 3.3 编解码流程

编解码由两个核心组件负责：

| 组件 | 作用 |
|------|------|
| **TriDecoder** | 协议层解码器：解析 HTTP/2 Data 帧中的 5 字节 gRPC 头（1 byte 压缩标记 + 4 bytes 消息长度），提取 Payload |
| **PackableMethod** | 序列化层：负责消息的打包/解包，根据序列化方式选择不同的实现 |

**PackableMethod 的两种实现**

```
┌──────────────────────────────────────────────────────────────────┐
│                    PackableMethod 实现体系                          │
│                                                                  │
│  ┌─────────────────────┐    ┌─────────────────────────────┐     │
│  │ Protobuf 模式        │    │ Wrapper 模式                  │     │
│  │ (IDL 定义)           │    │ (Java Interface + POJO)      │     │
│  ├─────────────────────┤    ├─────────────────────────────┤     │
│  │ PbUnpack            │    │ WrapRequestUnpack            │     │
│  │ PbArrayPacker       │    │ WrapResponsePack             │     │
│  │                     │    │                             │     │
│  │ 直接操作 Protobuf   │    │ 先序列化为 byte[],           │     │
│  │ Message 对象        │    │ 再包装到 TripleRequestWrapper  │     │
│  │                     │    │ /TripleResponseWrapper       │     │
│  │ 一次序列化          │    │ 两次序列化 (双重序列化问题)     │     │
│  └─────────────────────┘    └─────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

**完整编解码时序**

```
┌──────────────────────────────────────────────────────────────────┐
│             Triple Unary RPC 完整请求-响应时序                      │
│                                                                  │
│  Consumer (客户端)                      Provider (服务端)         │
│  ─────────────────                      ─────────────────        │
│                                                                  │
│  TripleInvoker#doInvoke()                                        │
│    │                                                              │
│    ├─► 1. 从 ConsumerModel 获取 ServiceDescriptor                 │
│    │      MethodDescriptor                                          │
│    │                                                              │
│    ├─► 2. 创建 TripleClientCall                                   │
│    │      (封装一次 Stream 调用)                                    │
│    │                                                              │
│    ├─► 3. 创建 DeadlineFuture                                     │
│    │      (异步等待结果的 Future)                                  │
│    │                                                              │
│    ├─► 4. call.start(request, callListener)                      │
│    │      │                                                       │
│    │      ├─► stream.sendHeader(headers)                          │
│    │      │   ──── HEADERS Frame ─────►                           │
│    │      │   :path=/Service/Method                              │
│    │      │   content-type=application/grpc+proto               │
│    │      │   tri-service-version=1.0.0                         │
│    │      │                                                       │
│    │      ├─► PackableMethod#packRequest()                       │
│    │      │   序列化请求参数                                         │
│    │      │                                                       │
│    │      ├─► stream.sendMessage(data)                            │
│    │      │   ──── DATA Frame ─────►                             │
│    │      │   [压缩标记][长度][Payload]                             │
│    │      │                                                       │
│    │      └─► stream.halfClose()                                  │
│    │          ──── DATA Frame (EndStream=true) ──►              │
│    │                                                              │
│    │                                        TripleHttp2Frame     │
│    │                                        ServerHandler         │
│    │                                           │                  │
│    │                                        ├─► onHeadersRead()   │
│    │                                        │   解析 path →       │
│    │                                        │   PathResolver      │
│    │                                        │   定位 Invoker      │
│    │                                        │   构建 RpcInvocation│
│    │                                        │                    │
│    │                                        ├─► onDataRead()      │
│    │                                        │   TriDecoder 解帧   │
│    │                                        │   PackableMethod    │
│    │                                        │   #parseRequest()   │
│    │                                        │   反序列化参数      │
│    │                                        │                    │
│    │                                        ├─► EndStream 到达   │
│    │                                        │   → invoke() 执行业务│
│    │                                        │                    │
│    │                                        ├─► onReturn()        │
│    │                                        │   PackableMethod    │
│    │                                        │   #packResponse()   │
│    │                                        │   序列化返回值      │
│    │                                        │                    │
│    │      ◄─── HEADERS Frame ────────                           │
│    │      :status=200                                            │
│    │      content-type=application/grpc+proto                  │
│    │                                                              │
│    │      ◄─── DATA Frame ────────────                           │
│    │      [压缩标记][长度][Payload]                                │
│    │                                                              │
│    │      ◄─── HEADERS Frame (Trailers, EndStream=true) ─      │
│    │      grpc-status=0                                          │
│    │                                                              │
│    ├─► 5. TripleHttp2ClientResponseHandler                       │
│    │      TriDecoder#deframe()                                    │
│    │      → PackableMethod#parseResponse()                       │
│    │      → DeadlineFuture#received()                             │
│    │                                                              │
│    └─► 6. 业务线程拿到结果，返回                                    │
└──────────────────────────────────────────────────────────────────┘
```

### 3.4 TripleInvoker#doInvoke()：RPC 类型分发

`TripleInvoker` 是客户端调用的核心类，它的 `doInvoke()` 方法是所有 RPC 调用的分发入口：

```java
protected Result doInvoke(final Invocation invocation) {
    // 1. 检查连接可用性
    // 2. 从 ConsumerModel 获取 ServiceDescriptor、MethodDescriptor
    // 3. 实例化 TripleClientCall（封装一次 Stream）
    // 4. 根据 RPC 类型分发：
    switch (methodDescriptor.getRpcType()) {
        case UNARY:
            result = invokeUnary(methodDescriptor, invocation, call);
            break;
        case SERVER_STREAM:
            result = invokeServerStream(methodDescriptor, invocation, call);
            break;
        case CLIENT_STREAM:
        case BI_STREAM:
            result = invokeBiOrClientStream(methodDescriptor, invocation, call);
            break;
    }
}
```

`RpcType` 的判断依据是服务定义方式：
- 使用 **Protobuf IDL** 时，`RpcType` 由 `.proto` 文件中 `stream` 关键字决定
- 使用 **Java Interface** 时，根据方法参数中的 `StreamObserver` 位置和返回值类型自动推断

### 3.5 连接复用与性能优化

Triple 在连接管理上做了两个关键优化：

**（1）连接复用**

同一 IP:Port 上的多个服务共享一个物理连接——`ConnectionManager` 按地址缓存 `Connection` 对象。这消除了"每个接口建一条 TCP 连接"的资源浪费。

```
┌──────────────────────────────────────────────────────────────────┐
│                     连接复用 vs 独立连接                            │
│                                                                  │
│  Dubbo 2 默认（dubbo协议）:                                        │
│  Consumer ──── TCP Conn-1 ────► Provider  (接口 OrderService)     │
│  Consumer ──── TCP Conn-2 ────► Provider  (接口 UserService)      │
│  Consumer ──── TCP Conn-3 ────► Provider  (接口 LogService)       │
│  缺点: 3 个 TCP 连接 = 3 次握手 + 3 份连接维护开销                  │
│                                                                  │
│  Triple 协议:                                                     │
│  Consumer ──── TCP Conn-1 ────► Provider                          │
│                  ├── Stream-1  →  OrderService                   │
│                  ├── Stream-2  →  UserService                    │
│                  └── Stream-3  →  LogService                     │
│  优点: 1 个 TCP 连接，HTTP/2 多路复用，连接开销降低 ~90%            │
└──────────────────────────────────────────────────────────────────┘
```

**（2）TripleWriteQueue 批量发送**

```java
// TripleWriteQueue 将多个响应帧批量提交到 Netty IO 线程
// 减少用户线程（业务线程）与 IO 线程之间的上下文切换

// 伪代码示意
writeQueue.enqueue(frame1);  // 不立即 flush
writeQueue.enqueue(frame2);  // 排队
writeQueue.enqueue(frame3);  // 排队
writeQueue.flush();          // 批量一次性提交到 Netty
```

---

## 四、四种通信模式

Triple 协议原生支持四种通信模式，这是 Dubbo 2 协议无法做到的：

| 模式 | 请求方式 | 响应方式 | HTTP 版本要求 |
|------|---------|---------|-------------|
| **UNARY** | 一次请求 | 一次响应 | HTTP/1.1 或 HTTP/2 |
| **SERVER_STREAM** | 一次请求 | 多次响应（流） | HTTP/2 Only |
| **CLIENT_STREAM** | 多次请求（流） | 一次响应 | HTTP/2 Only |
| **BIDIRECTIONAL_STREAM** | 多次请求（流） | 多次响应（流） | HTTP/2 Only |

### 4.1 Unary（一元调用）

最常规的 RPC 调用——客户端发送一次请求，服务端返回一次响应。两种定义方式：

**Protobuf IDL 定义:**
```protobuf
service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply);
}
```

**Java Interface 定义:**
```java
public interface Greeter {
    String sayHello(String name);
}
```

### 4.2 Server Stream（服务端流）

客户端发送一次请求，服务端通过 `StreamObserver` 多次推送响应：

**Protobuf 定义:**
```protobuf
service Greeter {
    rpc SayHelloServerStream (HelloRequest) returns (stream HelloReply);
}
```

**Java Interface 定义:**
```java
public interface Greeter {
    void sayHelloServerStream(String name, StreamObserver<String> responseObserver);
    // 参数: 第0个是请求参数，第1个是响应流 StreamObserver
    // 服务端通过 responseObserver.onNext() 多次推送数据
}
```

**服务端实现示例:**
```java
@Override
public void sayHelloServerStream(String name, StreamObserver<String> responseObserver) {
    for (int i = 0; i < 10; i++) {
        responseObserver.onNext("Hello " + name + ", message #" + i);
    }
    responseObserver.onCompleted();  // 流结束
}
```

**与 Unary 的关键区别**：服务端在收到 EndStream 后不会一次性返回全部结果，而是在处理过程中**持续调用** `responseObserver.onNext()` 逐条推送数据，最后调用 `onCompleted()` 结束流。

### 4.3 Client Stream & Bi Stream（客户端流 & 双向流）

由于 Java 语言的限制（没有多返回值语法），CLIENT_STREAM 和 BIDIRECTIONAL_STREAM 在实现上采用相同的模式。

**Protobuf 定义:**
```protobuf
service Greeter {
    rpc SayHelloBiStream (stream HelloRequest) returns (stream HelloReply);
}
```

**Java Interface 定义:**
```java
public interface Greeter {
    StreamObserver<String> sayHelloBiStream(StreamObserver<String> responseObserver);
    // 参数: StreamObserver 用于接收服务端推送的响应
    // 返回值: StreamObserver 用于向服务端发送请求
}
```

这是 Triple 协议中最独特的 API 设计：

```
┌──────────────────────────────────────────────────────────────────┐
│              Bi Stream 中 StreamObserver 的角色分配                 │
│                                                                  │
│  Consumer 端:                                                    │
│    StreamObserver<String> responseObserver = new StreamObserver<>() {│
│        onNext(data)  → 收到服务端推送的响应                          │
│        onError(t)    → 处理异常                                     │
│        onCompleted() → 服务端流结束                                  │
│    };                                                            │
│                                                                  │
│    StreamObserver<String> requestObserver =                     │
│        greeter.sayHelloBiStream(responseObserver);                │
│       // ↑ 返回值 = 用于向服务端发送数据的 StreamObserver              │
│                                                                  │
│    requestObserver.onNext("msg-1");  // 发送数据                  │
│    requestObserver.onNext("msg-2");  // 继续发送                  │
│    requestObserver.onCompleted();    // 客户端发送完毕              │
│                                                                  │
│  Provider 端:                                                    │
│    返回值 = 用于监听客户端流推送                                    │
│    参数中的 responseObserver = 用于向客户端推送消息                  │
└──────────────────────────────────────────────────────────────────┘
```

**四种模式在 HTTP/2 帧级别的区别**

```
┌──────────────────────────────────────────────────────────────────────┐
│ 模式         │  客户端 DATA 帧  │  服务端 DATA 帧  │ EndStream 标记    │
├──────────────┼─────────────────┼─────────────────┼──────────────────┤
│ UNARY        │ 1 个（含请求数据） │ 1 个（含响应数据）  │ 客户端发完即标记    │
│ SERVER_STREAM│ 1 个             │ N 个（流式推送）   │ 客户端发完标记      │
│ CLIENT_STREAM│ N 个（流式发送）   │ 1 个              │ 客户端发完所有后标记 │
│ BI_STREAM    │ N 个（流式发送）   │ N 个（流式推送）   │ 各自独立标记       │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.4 流控反压机制（Backpressure）

流式通信的一个核心问题是——如果 Producer 的生产速度远大于 Consumer 的消费速度，内存会被撑爆。Triple 基于 HTTP/2 实现了两层流控：

```
┌──────────────────────────────────────────────────────────────────┐
│                    Triple 流控反压机制                              │
│                                                                  │
│  Inbound 流控（Netty DefaultHttpLocalFlowController）:             │
│    ● 接收端维护接收窗口（默认 8MB）                                  │
│    ● 窗口耗尽时发送 WINDOW_UPDATE 帧扩大窗口                         │
│    ● 窗口为 0 时，发送端暂停发送                                     │
│                                                                  │
│  Outbound 流控（Triple 自定义 TriHttpRemoteFlowController）:        │
│    ● 发送端检查远程接收窗口                                          │
│    ● 窗口为 0 时，抛出自定义异常                                    │
│    ● 异常透传到客户端业务层 → 用户感知到背压 → 停止发送数据            │
│                                                                  │
│  ┌─────────────────────┐                                         │
│  │ Consumer 发送速度过快 │                                         │
│  │   requestObserver   │                                         │
│  │   .onNext(msg-1)    │──► 服务端接收窗口充裕，正常处理             │
│  │   .onNext(msg-2)    │──► 正常                                  │
│  │   .onNext(msg-3)    │──► 正常                                  │
│  │   .onNext(msg-4)    │──► ❌ 窗口为 0，抛出 FlowControlException │
│  │   .onNext(msg-5)    │──► ❌ 异常透传到业务代码                    │
│  │                     │     业务收到异常 → 暂停发送 → 等待窗口恢复   │
│  └─────────────────────┘                                         │
└──────────────────────────────────────────────────────────────────┘
```

默认流控窗口大小为 **8MB**（`DEFAULT_WINDOW_INIT_SIZE = MIB_8`），可通过配置调整。

---

## 五、Wrapper 模式 vs IDL 模式

这是 Triple 协议使用中最重要的决策点——它决定了你的序列化性能和开发体验。

### 5.1 IDL 模式：Protobuf 原生

使用 `.proto` 文件定义服务，编译生成 Stub，直接使用 Protobuf 序列化传输数据。

```protobuf
syntax = "proto3";
package com.example;
option java_multiple_files = true;

message HelloRequest {
    string name = 1;
    int32 age = 2;
}

message HelloReply {
    string message = 1;
}

service GreeterService {
    rpc greet (HelloRequest) returns (HelloReply);
    rpc greetServerStream (HelloRequest) returns (stream HelloReply);
    rpc greetBiStream (stream HelloRequest) returns (stream HelloReply);
}
```

优点：序列化紧凑、跨语言、一次序列化、性能高。
缺点：必须写 `.proto` 文件，必须编译生成 Stub，不能直接使用已有的 Java POJO。

### 5.2 Wrapper 模式：Java Interface 兼容

不需要 IDL，直接用 Java Interface + POJO 定义服务：

```java
public interface GreeterService {
    String greet(String name);
    void greetServerStream(String name, StreamObserver<String> response);
    StreamObserver<String> greetBiStream(StreamObserver<String> response);
}
```

内部实现原理：框架检测到参数不是 Protobuf Message 时，自动使用 Wrapper 包装。

```protobuf
// TripleRequestWrapper — Dubbo 内部使用的 Protobuf 消息
message TripleRequestWrapper {
    string serializeType = 1;    // 标记实际序列化类型，如 "hessian2"、"json"、"kryo"
    repeated bytes args = 2;     // 每个参数先用自定义序列化器序列化为 byte[]
    repeated string argTypes = 3; // 参数类型名，便于 Consumer 端反序列化
}

message TripleResponseWrapper {
    string serializeType = 1;
    bytes data = 2;             // 返回值序列化后的 byte[]
    string type = 3;            // 返回值类型名
}
```

**双重序列化问题**

```
┌──────────────────────────────────────────────────────────────────┐
│                   Wrapper 模式的双重序列化                           │
│                                                                  │
│  第一次序列化（自定义序列化器）:                                      │
│    Java POJO 对象 → Hessian2/Kryo/JSON 序列化 → byte[]           │
│                                                                  │
│  第二次序列化（Protobuf）:                                          │
│    byte[] → 包装到 TripleRequestWrapper.args → Protobuf 序列化 →   │
│    wire format bytes                                             │
│                                                                  │
│  网络传输:                                                        │
│    Protobuf wire format → HTTP/2 DATA Frame → 发送               │
│                                                                  │
│  结果: 每次 RPC 调用经历两次序列化 + 两次反序列化                       │
│        序列化开销翻倍，CPU 和内存占用增加                              │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 性能影响

根据 Apache Dubbo Issue #10776，双重序列化会导致**显著的性能退化**。这是社区确认的已知问题，官方正在探索绕过 Protobuf 二次序列化的方案（通过 Content-Type 协商序列化方式）。

### 5.4 选择建议

```
┌──────────────────────────────────────────────────────────────────┐
│              IDL 模式 vs Wrapper 模式 决策树                        │
│                                                                  │
│  需要跨语言互通吗？                                                  │
│    ├─ 是 → IDL 模式 (Protobuf)                                     │
│    └─ 否 → 现有服务是否已经用了 Java Interface + POJO？              │
│              ├─ 是 → 是追求性能还是追求迁移成本？                      │
│              │       ├─ 性能优先 → 改造为 IDL 模式                   │
│              │       └─ 迁移成本优先 → Wrapper 模式 (接受性能折损)      │
│              └─ 否 → IDL 模式 (Protobuf)                           │
│                                                                  │
│  核心原则: 使用 Triple 协议时，尽量采用 IDL 模式。                     │
│            Wrapper 模式是兼容手段，不是推荐方案。                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 六、异常回传设计

### 6.1 问题场景

在 Dubbo 2 中，Provider 抛出的自定义异常可以被 Consumer 精确捕获：

```java
// Dubbo 2: 正常工作
try {
    orderService.createOrder(request);
} catch (OrderDuplicateException e) {
    // ✅ 能精确捕获自定义异常
}
```

但升级到 Dubbo 3 Triple 协议后，所有异常都被统一包装成 `RpcException`，用户无法区分具体的异常类型：

```java
// Dubbo 3 Triple: 升级后问题
try {
    orderService.createOrder(request);
} catch (RpcException e) {
    // ❌ 丢失了原始异常类型和信息
}
```

### 6.2 方案演进：从 Trailer 到 Body

社区对该问题进行了多轮讨论，经历了两个方案的演进：

```
┌──────────────────────────────────────────────────────────────────┐
│                    方案一: Trailer 携带异常 (被舍弃)                 │
│                                                                  │
│  Provider                                         Consumer      │
│  捕获异常 → 序列化异常对象 →                             │
│  放入 HTTP Trailer →  ──── HEADERS Frame (Trailer) ──►        │
│                       tri-exception-data: [base64...]           │
│                                                                  │
│  舍弃原因:                                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 1. Header 大小限制: gRPC Header ~8KB, 异常含堆栈信息可能超限  │  │
│  │ 2. 挤占用户 Header 空间: 每个自定义 Header 都减少用户可用空间  │  │
│  │ 3. 性能退化: 过大的 Header 导致 HTTP/2 帧处理性能下降         │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                  方案二: Body 携带异常 (最终方案)                     │
│                                                                  │
│  Provider                                         Consumer      │
│  捕获异常 → TripleWrapper + Protobuf 序列化 →              │
│  放入 HTTP Body ──── DATA Frame ──────────────►             │
│  HTTP 状态码: 200 (异常被视作带有业务属性的数据)                    │
│                                                                  │
│  Consumer 端:                                                    │
│  接收 DATA Frame → 解析 TripleResponseWrapper →                  │
│  反序列化异常对象 → throw 原始异常 → 业务代码 instanceof 精准捕获      │
│                                                                  │
│  关键决策:                                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ HTTP 状态码用 200 而非 5xx: 因为异常信息带有业务属性，          │  │
│  │ 不应被网关/代理当作网络错误而误处理。                           │  │
│  │ 与 gRPC 的设计一致: gRPC 也是通过 grpc-status trailer          │  │
│  │ 传递错误状态，而非 HTTP 状态码。                               │  │
│  │                                                              │  │
│  │ 跨语言场景: Java 异常回传是 Dubbo 特有增强，Go/Node.js 等      │  │
│  │ 语言没有 Java 的 "try-catch 按类型捕获" 概念，跨语言调用时    │  │
│  │ 使用标准 gRPC grpc-status + grpc-message trailer 传递错误。  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 七、Triple X：HTTP/3 时代的协议演进

### 7.1 HTTP/3 (QUIC) 的核心原理

从 Dubbo 3.3.0 开始，Triple 协议升级为 **Triple X**，新增对 HTTP/3 的支持。HTTP/3 是一套全新的传输协议，基于 Google 的 **QUIC 协议**，使用 **UDP** 替代 TCP：

```
┌──────────────────────────────────────────────────────────────────┐
│              HTTP 协议演进路线                                      │
│                                                                  │
│  HTTP/1.1 (1997)        HTTP/2 (2015)         HTTP/3 (2022)      │
│  ┌──────────────┐     ┌──────────────┐      ┌──────────────┐    │
│  │ TCP           │     │ TCP           │      │ UDP + QUIC   │    │
│  │ 文本协议       │     │ 二进制帧协议   │      │ 二进制帧协议    │    │
│  │ 无多路复用     │     │ 多路复用       │      │ 多路复用       │    │
│  │ (队头阻塞)     │     │ (队头阻塞)     │      │ (无队头阻塞)   │    │
│  │ 明文/可选TLS   │     │ 可选TLS        │      │ 强制TLS 1.3   │    │
│  │ 域名:端口连接  │     │ 域名:端口连接   │      │ Connection ID │    │
│  │               │     │               │      │ (连接迁移)     │    │
│  └──────────────┘     └──────────────┘      └──────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

HTTP/3 的五个核心突破：

| 特性 | 原理 | 对 RPC 的意义 |
|------|------|-------------|
| **0-RTT 连接建立** | QUIC 合并了 TCP 握手 + TLS 握手，缓存会话密钥后可直接发送数据 | 冷启动调用延迟大幅缩短 |
| **无队头阻塞** | 多路复用在 QUIC Stream 层面实现，丢包只影响单个 Stream | HTTP/2 的"一个丢包阻塞所有 Stream"问题被彻底解决 |
| **连接迁移** | 使用 Connection ID 而非 IP:Port 标识连接，网络切换时连接不中断 | 移动端、跨网络场景连接保持 |
| **强制 TLS 1.3** | 加密是 QUIC 的强制部分，非可选 | 比 HTTP/2 更安全的默认通信 |
| **改进的拥塞控制** | QUIC 运行在用户态，拥塞控制算法可热更新 | 弱网环境下吞吐量显著提升 |

### 7.2 Triple 集成 HTTP/3 的实现

Triple 通过**协议协商 + 自适应切换**的方式集成 HTTP/3：

```
┌──────────────────────────────────────────────────────────────────┐
│                 Triple HTTP/3 协议协商流程                           │
│                                                                  │
│  Consumer                                Provider                │
│       │                                     │                    │
│       │ ──── HTTP/2 Connection ──────────► │                    │
│       │     (首次连接: HTTP/2)               │                    │
│       │                                     │                    │
│       │ ◄─── HTTP Response ──────────────   │                    │
│       │     Alt-Svc: h3=":50051"            │  告知支持 HTTP/3   │
│       │                                     │                    │
│       ├─► AutoSwitchConnectionClient        │                    │
│       │     检测到 Alt-Svc                   │                    │
│       │     发起 QUIC 连接 (UDP)             │                    │
│       │                                     │                    │
│       │ ──── QUIC Connection ────────────► │                    │
│       │     (后续请求: HTTP/3)               │                    │
│       │                                     │                    │
│       │  如果网络不支持 QUIC (防火墙拦截 UDP):  │                    │
│       │     → 回退 HTTP/2                    │                    │
│       │     → 保持正常通信                    │                    │
└──────────────────────────────────────────────────────────────────┘
```

**配置开启:**

```yaml
dubbo:
  protocol:
    name: tri
    triple:
      http3:
        enabled: true
```

需要额外引入依赖：

```xml
<dependency>
    <groupId>io.netty.incubator</groupId>
    <artifactId>netty-incubator-codec-http3</artifactId>
    <version>0.0.28.Final</version>
</dependency>
```

**Triple X 的协议体系全景：**

```
┌──────────────────────────────────────────────────────────────────┐
│                    Triple X 协议体系全景                            │
│                                                                  │
│                        业务代码                                   │
│               (Java Interface / Protobuf)                         │
│                           │                                      │
│              ┌────────────┴────────────┐                         │
│              │     Triple Protocol     │                         │
│              │  (协议层: 编解码/流控/超时)│                         │
│              └────────────┬────────────┘                         │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                    │
│         │                 │                 │                    │
│    ┌────┴────┐       ┌────┴────┐       ┌────┴────┐              │
│    │ HTTP/1.1 │       │ HTTP/2  │       │ HTTP/3  │              │
│    │ (TCP)   │       │ (TCP)   │       │ (QUIC)  │              │
│    │ Unary   │       │ Unary + │       │ Unary + │              │
│    │ Only    │       │ Stream  │       │ Stream  │              │
│    └─────────┘       └─────────┘       └─────────┘              │
│                                                                  │
│  同一业务代码，自适应选择最佳传输协议                                 │
└──────────────────────────────────────────────────────────────────┘
```

### 7.3 弱网效率提升 6 倍

根据 Dubbo 官方在 2024 年 11 月发布的数据，HTTP/3 在弱网络条件下效率提升达到 **6 倍**。这主要归功于：

- QUIC 的 0-RTT 握手减少了连接建立时间
- 无队头阻塞避免了 TCP 层面的"一个丢包卡住所有请求"
- 用户态拥塞控制算法可针对 RPC 场景优化

适用场景：
- 跨地域/跨云调用（延迟高、丢包率高）
- 移动网络环境（网络切换频繁）
- 对请求延迟有严苛要求的金融/交易系统

---

## 八、性能基准与协议选型

### 8.1 官方 Benchmark 数据

以下数据来自 Apache Dubbo 官方 Benchmark（4C8G Linux JDK8，32 并发，单链路场景）：

```
┌──────────────────────────────────────────────────────────────────────┐
│ 场景            Dubbo+Hessian2  Dubbo+Protobuf  Triple+Protobuf  Triple+Wrapper │
│                   (Dubbo 2.7)     (Dubbo 3.0)     (Dubbo 3.0)    (Dubbo 3.0)│
├──────────────────────────────────────────────────────────────────────┤
│ 无参方法         30,333 ops/s    24,123 ops/s     7,016 ops/s     6,635 ops/s │
│ (P99 延迟)        2.5ms           3.2ms            8.7ms           9.1ms    │
├──────────────────────────────────────────────────────────────────────┤
│ POJO 返回值       8,984 ops/s    21,479 ops/s     6,255 ops/s     6,491 ops/s │
│ (P99 延迟)        6.1ms           3.0ms            8.9ms          10ms      │
├──────────────────────────────────────────────────────────────────────┤
│ POJO 列表返回值    1,916 ops/s    12,722 ops/s     6,920 ops/s     2,833 ops/s │
│ (P99 延迟)       34ms             7.7ms            9.6ms          27ms      │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.2 数据解读

从上面的数据可以得出几个关键结论：

**（1）点对点场景：Dubbo 协议 > Triple 协议**

无论是在 Hessian2 还是 Protobuf 序列化下，Dubbo 协议的点对点性能都优于 Triple 协议。这在 POJO 列表场景最为明显——Dubbo+Protobuf 达到 12,722 ops/s，是 Triple+Protobuf 的近 2 倍。

根本原因：Triple 基于 HTTP/2，而 Dubbo 协议直接基于 TCP。**"基于 HTTP/2 的 RPC 协议"相比"基于 TCP 的 RPC 协议"存在固有的协议头开销**——HTTP/2 帧头、gRPC 帧头、多路复用管理等都会增加固定的 CPU 开销。

**（2）Wrapper 模式在复杂对象下退化严重**

Triple+Wrapper 在 POJO 列表场景下退化了约 **60%**（从 6,920 降至 2,833 ops/s），P99 延迟从 9.6ms 飙升至 27ms。这是双重序列化的直接后果。

**（3）Protobuf 序列化本身有优势**

注意 Dubbo+Protobuf 在 POJO 返回值场景下比 Dubbo+Hessian2 快了 **139%**（8,984 → 21,479 ops/s）。Protobuf 的紧凑编码在复杂对象场景下优势明显。

### 8.3 协议选型决策树

```
┌──────────────────────────────────────────────────────────────────┐
│                     RPC 协议选型决策树                               │
│                                                                  │
│  场景一: 纯 Dubbo 内部调用，追求极致性能                              │
│    → dubbo:// + Protobuf 或 Hessian2                             │
│                                                                  │
│  场景二: 需要跨语言互调 (Java ↔ Go / Python / Node.js)              │
│    → tri:// + Protobuf IDL                                       │
│                                                                  │
│  场景三: 需要流式通信 (大数据传输 / 推送 / 实时交互)                    │
│    → tri:// (唯一选择，dubbo 协议不支持流式)                          │
│                                                                  │
│  场景四: 需要通过网关代理 (API Gateway / Service Mesh)                │
│    → tri:// + HTTP/2 (网关原生支持，无需协议转换)                     │
│                                                                  │
│  场景五: Dubbo 2 老用户平滑迁移                                      │
│    → tri:// + Java Interface (Wrapper 模式，零代码改造)              │
│    → 后续逐步改造为 Protobuf IDL 模式                                │
│                                                                  │
│  场景六: 跨地域 / 弱网环境 / 移动端                                    │
│    → tri:// + HTTP/3 (Triple X，弱网优化显著)                       │
│                                                                  │
│  场景七: 需要 curl/浏览器直接调用调试                                  │
│    → tri:// + application/json                                   │
└──────────────────────────────────────────────────────────────────┘
```

**核心认知**：Triple 协议的真正价值不在于"比 Dubbo 协议快"——它确实在点对点场景下更慢。它的价值在于**打破了 Dubbo 2 协议的信息孤岛**，让 Dubbo 服务能够融入 HTTP/gRPC 的云原生生态，同时保证了服务治理能力的完整继承。

---

## 九、总结

### 9.1 Triple 协议核心优势一览

```
┌──────────────────────────────────────────────────────────────────┐
│                   Triple 协议核心优势总结                            │
│                                                                  │
│  ┌─────────────────┐                                             │
│  │ gRPC 100% 兼容   │  Dubbo Client ↔ gRPC Server 双向互通         │
│  │                 │  不改一行代码接入 gRPC 生态                     │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 多协议同端口     │  同一端口同时支持 Triple、gRPC、HTTP/1.1       │
│  │                 │  所有请求路由到相同业务逻辑                      │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 流式通信         │  Unary / Server Stream / Client Stream /      │
│  │                 │  Bi Stream 四种模式，HTTP/2 背压流控           │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 开发模式灵活      │  IDL 模式 (跨语言+高性能) 与 Wrapper 模式       │
│  │                 │  (Java Interface 零学习成本) 自由选择           │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 调试体验好       │  curl + jq 直接调试，Postman/浏览器也可用       │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 服务治理融合      │ 与 Dubbo 服务发现/路由/限流/降级无缝融合       │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 代码轻量         │  仅几千行核心代码，远小于 gRPC 的 10 万+ 行      │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ HTTP/3 演进     │  Triple X 支持 HTTP/3 (QUIC)，弱网提升 6 倍    │
│  │                 │  0-RTT、无队头阻塞、连接迁移                     │
│  └─────────────────┘                                             │
│  ┌─────────────────┐                                             │
│  │ 生产级验证       │  阿里巴巴数十万容器使用，经大规模生产环境验证      │
│  └─────────────────┘                                             │
└──────────────────────────────────────────────────────────────────┘
```

### 9.2 Triple 协议的设计哲学

Triple 协议的本质不是"又一个 RPC 协议"，而是 Dubb​​o 对云原生时代 RPC 通信范式的回答：

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Dubbo 2 时代: "RPC 就是 TCP 上的二进制私有协议"                    │
│                                                                  │
│        ──────────────────► 云原生时代 ◄──────────────────          │
│                                                                  │
│   Dubbo 3 时代: "RPC 应该基于标准 HTTP，融入云原生生态"              │
│                                                                  │
│   这不仅是协议的变化，更是理念的转变：                                  │
│                                                                  │
│   从封闭到开放:  私有协议 → HTTP 标准，curl 就能调试                   │
│   从单体到生态:  Java 独享 → gRPC 生态完全兼容                        │
│   从静态到动态:  单一传输协议 → HTTP/1.1 → HTTP/2 → HTTP/3 自动协商   │
│   从一维到多维:  Unary 单模式 → 四种通信模式自由组合                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Triple 协议用几千行代码，在 HTTP 基础设施之上实现了 gRPC 的全部能力，同时保留了 Dubbo 独有的服务治理优势。它不是要取代 gRPC，而是要让 Dubbo 生态中数以万计的服务能够在保持治理能力的前提下，无缝融入云原生世界。

---

**参考资料**：
- [Apache Dubbo — Triple 协议规范](https://cn.dubbo.apache.org/zh-cn/overview/reference/protocols/triple/)
- [Apache Dubbo — Triple HTTP/3 支持](https://github.com/apache/dubbo/issues/14296)
- [Apache Dubbo — RPC 协议基准测试](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/reference-manual/performance/rpc-benchmarking/)
- [Apache Dubbo — Triple 协议 Java 异常回传设计与实现](https://cn.dubbo.apache.org/zh/blog/2022/12/19/triple-%E5%8D%8F%E8%AE%AE%E6%94%AF%E6%8C%81-java-%E5%BC%82%E5%B8%B8%E5%9B%9E%E4%BC%A0%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/)
- [Apache Dubbo — 流式通信](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/tasks/protocols/triple/streaming/)
- [Apache Dubbo 正式发布 HTTP/3 版本 RPC 协议，弱网效率提升 6 倍](https://developer.aliyun.com/article/1641569)
- [Dubbo3 and the Triple Protocol — Why It Is the Logical Choice (Alibaba Cloud)](https://www.alibabacloud.com/blog/dubbo3-and-the-triple-protocol-why-it-is-the-logical-choice_598953)
