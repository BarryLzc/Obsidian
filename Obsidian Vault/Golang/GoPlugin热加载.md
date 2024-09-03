底层原理：通过go build -buildmode=plugin -o xxx.so xxx.go将go文件编译为so文件（go-plugin），使用golang的"plugin"包解析So文件。

编译So文件示例
1. 普通编译：go build -buildmode=plugin -o xxx.so
2. debug模式编译：go build -gcflags "all=-N -l" -o xxx.so -buildmode=plugin xxx.go     
3. linux交叉编译plugin：CC=x86_64-linux-musl-gcc CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -ldflags "-s -w" -buildmode=plugin -o signlinux.so（注意需要先在安装工具链 `brew install FiloSottile/musl-cross/musl-cross`）
```
package main

func Add(x,y int) int {
	return x+y
}

func Subtract(x,y int) int {
	return x-y
}
```

So文件解析示例
```
package main

import (
	"fmt"
	"plugin"
)

func main() {
	ptr, err := plugin.Open("math.so")
	if err != nil {
		fmt.Println("err != nil")
	}
	Add, _ := ptr.Lookup("Add")
	sum := Add.(func(int,int) int)(5,4)
	fmt.Println(sum)
}
```

go-plugin 踩过的坑
go 的插件系统必须满足依赖  
1. 宿主程序和插件的deps必须一致,包括GOVERSION等  
2. 宿主程序和插件的gcflags 等具有部分强制性，例如debug模式下 all="-l -N"  
3. [[GOPATH]]一致，不同的环境构建时候可能带有path,导致符号表找不到，因此需要 trimpath（同一个环境下运行宿主和程序，不需要考虑这点）  
4. 同名的so文件只会加载一次，第二次直接用cache,因此如果第二次加载是个空文件，也不会报错（go设计问题）  
5. 不同名的so文件如果go.mod定义的module是一样的，则提示plugin already loaded  
6. 编译为plugin的时候必须开启CGO  
7. 宿主程序在debug模式下加载的插件，插件必须也禁止内联优化，否则提示ABI不同  

**plugin system**
```
go build --trimpath --buildmode=plugin -> plugin was built with a different version of package internal/goarch"  
go build --trimpath --buildmode=plugin plugin was built with a different version of package gopkg.inshopline.com/gsoul/openmf/stateless/model"
```

场景分析  
1. 当go宿主机器 build命令增加-trimpath时候，so文件如果不包含，则提示plugin was built with a different version of package internal/goarch  
2. package deps 必须一样  
3. 两个插件是同一个文件 执行plugin.Open("p1.so")和p.Lookup("MySymbol")-- OK  
4. 两个插件源代码是一个，生成p1.so和p2.so,同时查询X此时提示 “plugin already loaded”
原因是，p1.so和p2.so 属于同一个plugin(pluginpath是go.mod定义的package ，两者一样)，因此会提示已经loaded  
但是当都加载p1.so的时候，会缓存，导致直接使用同一个。 即便是此时p1.so的内容已经改变了。  

```
1. 通过 `nm` 命令查看symbol ,发现两者的符号一样

2. -`strings myprogram | grep '/home/user/myproject'` 可以查看symbol
```

[[linux系统]]