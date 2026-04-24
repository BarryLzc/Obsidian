核心思想：使用池化的思想。区别于net，不是每一个http链接一个协程，而是将链接池化复用。

## 一、项目定位
fasthttp 是一个专注于 [[1.1]] 极致性能 的 Go 网络库，核心设计哲学：能复用的绝不分配，能避免 GC 的绝不制造垃圾。

## 二、池化技术
fasthttp 几乎对所有可复用对象都做了池化，这是它性能优于 net/http 的核心原因：

| **对象**                  | **池类型**               | **作用域**        | **重置策略**      |
| ----------------------- | --------------------- | -------------- | ------------- |
| **Request**             | `sync.Pool`           | 全局             | `Reset()`     |
| **Response**            | `sync.Pool`           | 全局             | `Reset()`     |
| **RequestCtx**          | `sync.Pool`           | per-Server     | `ctx.reset()` |
| **bufio.Reader/Writer** | `sync.Pool`           | per-Server     | `Reset(conn)` |
| **Args / URI**          | `sync.Pool`           | 全局             | `Reset()`     |
| **Body Buffer**         | `bytebufferpool.Pool` | 全局             | 有条件归还         |
| **GzipReader/Writer**   | `sync.Pool`           | 全局/per-level   | `Reset()`     |
| **WorkerChan**          | `sync.Pool`           | per-WorkerPool | 复用 channel    |
| **clientConn**          | 数组                    | per-HostClient | 重新初始化         |
| **HexInt Buffer**       | `sync.Pool`           | 全局             | 直接归还          |
关键设计： 每个对象获取后使用，用完 Reset() 清空状态再归还池中，避免了 new 操作和 GC 压力。

## 三、核心性能优化手段

### 1. 零拷贝 string/[]byte 转换（unsafe）
```
  // b2s.go - 零分配：[]byte → string
  func b2s(b []byte) string {
      return unsafe.String(unsafe.SliceData(b), len(b))
  }
  
   // s2b.go - 零分配：string → []byte
  func s2b(s string) []byte {
      return unsafe.Slice(unsafe.StringData(s), len(s))
  }

```
Go 标准库中 string(bytes) 会拷贝数据，fasthttp 直接复用底层内存，单次节省一次堆分配，高并发下效果显著。

### 2. 预分配全局变量
```
// strings.go - 所有 HTTP 常用字符串预分配为 []byte
  var (
      strCRLF     = []byte("\r\n")
      strHTTP11   = []byte("HTTP/1.1")
      strColon    = []byte(":")
      // ... 数十个常量
  )

```
避免每次请求都为 "HTTP/1.1" 等字符串分配内存。
### 3. FILO Worker Pool（CPU 缓存友好）
```

  // workerpool.go - 取最近使用的 worker（FILO 而非 FIFO）
  ch = ready[n]  // 从尾部取，最近归还的 worker

  为什么 FILO？ 最近使用的 goroutine 的栈和相关数据还在 CPU L1/L2 缓存中，复用它们比唤醒冷 goroutine 更快。

```
### 4. GOMAXPROCS 自适应 Channel
```
var workerChanCap = func() int {
      if runtime.GOMAXPROCS(0) == 1 { return 0 }  // 单核：阻塞式
      return 1                                      // 多核：带缓冲
  }()

```
### 5. Body 大小感知的池化策略
```
 // 超大 body 不归还池（避免池中驻留大块内存）
  if requestBodyPoolSizeLimit >= 0 && req.body != nil {
      req.ReleaseBody(requestBodyPoolSizeLimit)
  }

```
### 6. Keep-Alive 连接上复用 RequestCtx
同一连接的多次请求共用一个 RequestCtx，只做 reset() 而不释放/重新获取。

## 四、server架构
```
Accept Loop
      │
      ▼
  WorkerPool.Serve(conn)
      │
      ▼
  workerFunc(ch)          ← goroutine-per-connection
      │
      ▼
  serveConn(conn)
      │  ┌──── acquireCtx()     ← 从 sync.Pool 获取
      │  │     acquireReader()   ← 从 sync.Pool 获取
      │  │
      │  │  for { // keep-alive loop
      │  │      ReadRequest()
      │  │      Handler(ctx)     ← 用户处理
      │  │      WriteResponse()
      │  │      ctx.reset()      ← 重置，不释放
      │  │  }
      │  │
      │  └──── releaseCtx()      ← 归还 sync.Pool
      ▼
  Worker 归还 WorkerPool（FILO）

```

## 五、为什么适合做高性能网关

> [!SUCCESS] 核心优势：极致的资源复用
> 
> `fasthttp` 的设计哲学是：**能不分配就不分配，能复用就复用。**

|**特性**|**对网关的意义**|
|---|---|
|**极低的 GC 压力**|网关是纯转发，几乎所有对象都可池化复用，P99 延迟稳定|
|**零拷贝转换**|网关需大量读取/比较 Header，零拷贝避免每次请求数百次分配|
|**Keep-Alive 优化**|网关与上下游都是长连接，`RequestCtx` 在连接上复用最大化收益|
|**Worker Pool + FILO**|网关并发极高，FILO 策略保持 CPU 缓存热度，降低上下文切换开销|
|**连接池**|`HostClient` 内置连接池，天然适合反向代理场景|
|**Body 流式处理**|`StreamBody` 支持大文件转发而不缓存整个 body|
|**内存可控**|`ReduceMemoryUsage` 模式 + body 大小限制，避免网关 OOM|
|**HTTP/1.1 专注**|网关内部通信 HTTP/1.1 足够，对外可用 nginx 做 HTTP/2 终结|


> [!WARNING] 不适合的场景（避坑指南）
> 
> - **需要 HTTP/2**：fasthttp 原生不支持 HTTP/2。
>     
> - **需要标准库兼容**：API 与 `net/http` 完全不同，生态集成（如某些中间件）成本高。
>     
> - **低并发场景**：池化的收益在低 QPS 下不明显，标准库更简单。
>     
> - **WebSocket 密集型**：虽有支持，但并非其核心性能优化点。
>     

WebSocket：**高度依赖长连接、需要实时、双向数据传输**的系统架构或应用场景

---

## 六、vs net/http 量化对比

|**维度**|**fasthttp**|**net/http**|
|---|---|---|
|**每请求分配**|**~0 (池化)**|~10+ 次堆分配|
|**string/[]byte 转换**|**零拷贝 (unsafe)**|每次拷贝|
|**连接复用**|**ctx + buffer 全复用**|仅连接复用|
|**Worker 管理**|**FILO + 自动清理**|无限 goroutine|
|**压缩器复用**|**池化**|每次新建|
|**吞吐量**|**~10x net/http**|基准线|
|**GC 停顿**|**极低**|高并发下显著|

---

## 总结

**fasthttp** 通过 **全链路池化 + 零拷贝 + CPU 缓存友好的调度** 实现了极致性能，特别适合作为高性能 API 网关、反向代理等纯转发场景的底层框架。

> [!INFO] 权衡 (Trade-off)
> 
> 性能的代价是**牺牲了 HTTP/2 支持**和**标准库兼容性**。在选择时需评估业务是否对极致延迟（P99）有强需求。