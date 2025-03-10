### 1. **`runtime` 包**

Go 的 `runtime` 包提供了一些用于调试和分析 Goroutine 的 API，例如：

- `runtime.NumGoroutine()`: 返回当前运行的 Goroutine 数量，可以帮助检测 Goroutine 泄露。
- `runtime.Stack()`: 获取当前所有 Goroutine 的栈信息，类似 `pprof` 的 `goroutine` 采样。

**示例**：
	package main

	import (
    "fmt"
    "runtime"
    "time")

	func main() {
    go func() {
        time.Sleep(time.Hour)
    }()

    time.Sleep(time.Second)
    fmt.Println("Goroutines:", runtime.NumGoroutine())
	}


**应用场景**：

- 在关键路径插入 `NumGoroutine()` 监控 Goroutine 数量，观察是否持续增加（可能泄露）。
- `runtime.Stack()` 获取所有 Goroutine 的状态，可以在日志或 panic 处理中调用。

---

### 2. **Go `trace` 工具**

`go tool trace` 可以提供更详细的 Go 运行时跟踪，包括 Goroutine 何时被创建、调度、阻塞、恢复等信息。

**使用方法**：

1. 代码中加入：
    import (
    "log"
    "os"
    "runtime/trace")

	func main() {
	    f, err := os.Create("trace.out")
	    if err != nil {
		    log.Fatal(err)
	    }
	    defer f.Close()

	    trace.Start(f)
	    defer trace.Stop()
	}

2. 运行程序后，使用 `go tool trace` 可视化分析：
    `go run main.go go tool trace trace.out`
    
3. 浏览器打开 `http://127.0.0.1:XXXXX/` 可查看详细的 goroutine 执行情况。

**应用场景**：

- 分析 Goroutine 创建、运行、调度、阻塞情况，查找长时间等待的 Goroutine。
- 发现 CPU 运行、I/O 等导致的 Goroutine 调度问题。

---

### 3. **`debug.Stack()`**

`debug.Stack()` 可以打印当前 Goroutine 的调用栈，适用于在关键位置（如 `recover()`）打印信息排查问题。

**示例**：
package main

	import (
    "fmt"
    "runtime/debug"
	)

	func main() {
	    go func() {
	        fmt.Println(string(debug.Stack()))
	    }()
	}

**应用场景**：
- 在 `panic` 发生时打印 Goroutine 的执行栈，帮助分析问题。
- 定位 Goroutine 运行到哪里，特别是在死锁或 Goroutine 阻塞时使用。

---

### 4. **Goroutine Dump（`http/pprof` `goroutine`）**

`net/http/pprof` 提供了 `goroutine` 采样，类似 `runtime.Stack()`，但更方便在运行时获取。

**使用方法**：

1. 导入 `net/http/pprof`：
	import _ "net/http/pprof"
	import "net/http"
	import "log"

	func main() {
	    go func() {
		    log.Println(http.ListenAndServe("localhost:6060", nil))
	    }()
	    // 其他业务逻辑
	}

2. 运行后访问：
    curl http://localhost:6060/debug/pprof/goroutine?debug=2

3. 也可以用 `go tool pprof` 分析：
    go tool pprof http://localhost:6060/debug/pprof/goroutine

**应用场景**：

- 快速查看当前 Goroutine 运行情况，分析死锁、阻塞、泄露等问题。
- 结合 `pprof` 可视化工具（如 `go tool pprof -http=:8080`）进一步分析。

---

### 5. **`-race` 竞态检测**

Go 运行时可以开启 `-race` 选项检测 Goroutine 之间的竞争条件（data race）。

**使用方法**：
go run -race main.go
go test -race ./...
**应用场景**：

- 发现并修复数据竞争问题，避免 Goroutine 之间非预期的读写操作。
- 适用于并发代码的调试和测试。

---

### 6. **`dlv` 调试器**

`delve`（`dlv`）是 Go 的强大调试工具，可以在运行时查看 Goroutine 的详细信息。

**安装**：
`go install github.com/go-delve/delve/cmd/dlv@latest`

**使用**：
`dlv debug ./main.go`

**调试 Goroutine**：
`(dlv) goroutines`

**应用场景**：

- 在死锁或阻塞时查看 Goroutine 的状态，找出问题所在。
- 在代码中设置断点，观察 Goroutine 的执行情况。

---

### 7. **手动 Goroutine 监控**

在代码中手动维护 Goroutine 的创建和销毁情况，帮助排查泄露问题。

**示例**：
	package main

	import (
    "fmt"
    "sync"
    "time"
	)

	var wg sync.WaitGroup

	func worker(id int) {
	    defer wg.Done()
	    fmt.Println("Worker", id, "started")
	    time.Sleep(time.Second)
	    fmt.Println("Worker", id, "done")
	}

	func main() {
	    for i := 0; i < 5; i++ {
	        wg.Add(1)
	        go worker(i)
	    }
	    wg.Wait()
	    fmt.Println("All workers done")
	}


**应用场景**：

- 监控 Goroutine 生命周期，避免泄露。
- 使用 `sync.WaitGroup` 确保所有 Goroutine 都正确完成。

---

## **总结**

|方法|适用场景|
|---|---|
|`runtime.NumGoroutine()`|监控 Goroutine 是否泄露|
|`runtime.Stack()`|快速获取所有 Goroutine 堆栈|
|`go tool trace`|详细跟踪 Goroutine 运行情况|
|`debug.Stack()`|`panic` 发生时获取当前 Goroutine 堆栈|
|`net/http/pprof`|在线分析 Goroutine 运行情况|
|`-race` 竞态检测|查找数据竞争问题|
|`dlv` 调试器|逐步调试 Goroutine|

可以根据问题类型选择合适的工具，如果是 Goroutine 泄露，建议结合 `runtime.NumGoroutine()`、`pprof goroutine` 和 `trace` 进行分析；如果是数据竞争，`-race` 检测非常有帮助。