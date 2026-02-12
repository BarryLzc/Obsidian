1. 单个源文件的 init 执行顺序
```
package main 

func init() {
	println("init a") 
} 
func init() { 
	println("init b") 
} 
func init() { 
	println("init c") 
} 
func main() { 
	println("main") 
}

$ go run main.go 
init a 
init b 
init c 
main
```
**结论：** 同一个源文件的 init 函数执行顺序与其定义顺序一致，从上到下。

2. 单个包的 init 执行顺序
```
// a.go 
package main 
func init() { 
	println("init a") 
} 
// b.go 
package main 
func init() { 
	println("init b")
} 

// c.go 
package main 
func init() { 
	println("init c") 
} 

// main.go 
package main 
func init() { 
	println("init main") 
} 
func main() { 
	println("main") 
}
$ go build && ./main 
init a 
init b 
init c 
init main 
main
```
**结论：** 同一个包中不同源文件 init 函数的执行顺序，是根据文件名的字典序来确定。

https://cloud.tencent.com/developer/article/2138066
