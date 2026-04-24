## 概述

一个高性能 API 网关，需要处理大量请求转发，对吞吐量和延迟有极高要求。fasthttp 在这类场景下比 gin（基于 net/http）有显著优势（虽然fast-http 不是最快的，也有比fast-http更快的网络框架gnet，但是没有考虑大报文的场景

---

## 1. 内存分配更少

fasthttp 通过对象复用减少 GC 压力。代码中大量使用了这种模式：
- Context 池化（core/app_new.go）：用 sync.Pool 复用请求上下文，避免每个请求都分配新对象 
- Request / Response 复用：使用 fasthttp.AcquireRequest / ReleaseRequest 模式
- 禁用 Header 规范化：ctx.Response.Header.DisableNormalizing() 跳过不必要的开销
---

##  2. 更高的连接吞吐

HTTP 客户端配置了高性能连接池（infrastructure/httpclient/fasthttp.go）：
- MaxConnsPerHost: 4096 — 每个后端服务维护大量连接
- StreamResponseBody: true — 支持流式响应（SSE/长连接）
- MaxIdleConnDuration: 59s — 长时间复用连接
---

## 3. gin 的功能对网关来说是多余的

- 网关有自定义路由逻辑（基于租户/命名空间解析路径），不需要 gin 的路由树
- 网关使用 Sonic JSON（字节跳动的高性能 JSON 库），不依赖 gin 的默认序列化
- gin 的中间件/绑定/验证等便利功能对网关场景价值不大，反而增加开销
---

## 4. fast-http vs net/http 的本质差异

| 特性             |    fasthttp    |      net/http/gin |
| :------------- | :------------: | ----------------: |
| 每个连接 goroutine | 复用 worker pool | 每个连接一个新 goroutine |
| 内存分配           |   大量对象复用，零拷贝   |         每次请求都有新分配 |
| Header 处理      |  []byte 直接操作   |         string 拷贝 |
| 适用场景           |    高吞吐代理/网关    |           通用web应用 |

---

## 5. fast-http vs gnet

### 5.1 内存拷贝与分配开销

`gnet` 的核心优势在于处理海量并发的小报文（如心跳包、简单的请求响应），因为它通过循环利用一小块 Buffer 来减少内存申请。

- **挑战**：当单个报文达到数 MB 甚至更大时，`gnet` 必须分配足够大的连续内存来承载。如果频繁处理大报文，`sync.Pool` 内部的缓冲区可能会频繁扩容或触发 GC，导致性能优势丧失。
- **对策**：需要手动调优 `gnet.WithReadBufferCap()` 等参数，或者配合自定义的内存池管理（如 `bytebufferpool`）来规避分配开销。
---

### 5.2. Reactor 模型的“阻塞”风险

这是最关键的一点。`gnet` 默认在 **Event-Loop (Reactor) 线程** 中处理业务逻辑。

- **风险**：大报文意味着较长的**编解码（Codec）时间**。如果在 Event-Loop 线程中进行耗时的大报文反序列化（例如解析一个巨大的 JSON 或 Protobuf），会直接阻塞该线程上的所有其他连接。
- **现象**：你会发现长尾延迟（P99）陡增，即便并发量并不大，整体吞吐量也会因为某个连接的“大包”而卡住。
---

### 5.3 I/O 效率对比

在 Linux 内核层面，对于大报文的传输：

- **标准库 (`net`)**：使用多协程模式。当一个 Goroutine 在等待大块数据 I/O 时，Go 调度器（GMP）可以很轻松地切换到其他 Goroutine，利用率较高。
- **gnet**：虽然 epoll 本身很快，但在用户态和内核态之间搬运大量数据时，非阻塞 I/O 并不能比阻塞 I/O（配合协程）带来质的飞跃。
---
### 5.4 小节
如果场景中**平均包体超过 1MB**，且连接数没有达到数万级别，建议使用 **标准库 net** 或者 **fasthttp**。Go 协程的抢占式调度在处理长耗时任务（大报文处理）时，比 Reactor 事件驱动模型更稳健。像网关会出现几十m的大报文，使用gnet性能反而没有那么好。