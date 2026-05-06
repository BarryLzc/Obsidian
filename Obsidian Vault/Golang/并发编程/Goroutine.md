一、 Goroutine（协程）相关面试题

1. **什么是Goroutine？它与线程的区别？**
    - Goroutine是Go语言用户态轻量级线程，由Go运行时管理，非操作系统线程。
    - 启动成本极低（KB级栈），创建和切换速度快。
2. **Goroutine 调度模型 (GMP)？**
    - G (Goroutine), M (Machine/内核线程), P (Processor/调度器上下文)。P控制G到M的映射，实现高性能工作窃取。
3. **如何优雅地终止一个Goroutine？**
    - 不能直接强制干掉另一个协程。通常使用`context.Context`或关闭一个专门的`quit` channel来通知其自行退出。
4. **什么是Goroutine泄漏？怎么处理？**
    - 定义：协程已启动但永远无法退出，导致内存持续增长。
    - 原因：读取了未发送数据的channel、向未被读取的channel发送数据。
    - 处理：使用`select`加超时控制，使用工具如`pprof`检测。 
二、 Channel（管道）相关面试题

1. **无缓冲Channel（Unbuffered）和有缓冲Channel（Buffered）的区别？**
    - 无缓冲：发送操作和接收操作必须同时准备好，是同步的，发送后立即阻塞直到被接收。
    - 有缓冲：发送时内部缓冲区未满则不阻塞，接收时缓冲区不空则不阻塞。
2. **对已关闭的Channel进行读写会怎么样？**
    - **读已关闭的chan**：会读出该类型的零值，且不会阻塞。
    - **写已关闭的chan**：会panic。
3. **Channel是线程安全的吗？**
    - 是的，Go运行时通过内部锁机制保证一个channel同时仅允许一个goroutine读写。
4. **如何判断Channel已被关闭？**
    - `v, ok := <-ch`，如果`ok`为`false`，则说明channel已关闭。 

三、 综合实战面试题

1. **使用Channel和Goroutine交替打印1-100之间的奇数和偶数？**
    - [分析与代码] 需要使用两个或多个channel来保证交替顺序。
2. **如何实现一个高性能的协程池？**
    - [设计] 定义`Task`任务，创建一个`JobQueue`（channel），预先启动N个工作协程循环从`JobQueue`读取任务处理。
3. **使用select进行超时控制？**
    - [场景] 发送请求后若3秒内无响应则放弃。 

go

```
select {
case <-ch:
    // 正常处理
case <-time.After(3 * time.Second):
    // 超时处理
}
```

Use code with caution.

4. **如何用Channel实现信号量机制？**
    - 使用带缓冲的channel来限制同一时间运行的协程数量。 

四、 核心注意点

- **[[不要通过共享内存来通信，而要通过通信来共享内存]]**（Do not communicate by sharing memory; instead, share memory by communicating）。
- 注意channel关闭的原则：**谁发送，谁关闭**。不要在接收方关闭channel。
- 理解`select`语句的多路复用能力