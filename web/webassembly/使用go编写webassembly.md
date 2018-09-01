# 使用go编写webassembly

## webassembly是什么？
> WebAssembly (abbreviated Wasm) is a binary instruction format for a stack-based virtual machine. Wasm is designed as a portable target for compilation of high-level languages like C/C++/Rust, enabling deployment on the web for client and server applications.

[webassembly](https://webassembly.org/)是可以支持在web浏览器或者v8等环境下的二进制格式，想具体了解可以查看[这个回答](https://www.zhihu.com/question/31415286/answer/58022648)  
本文只介绍如何使用golang生成wasm文件，并在浏览器上执行。

## 开始
需要先升级go到1.11版本

### 编写需要编译成wasm文件的go文件

```go
// main.go
package main

func main() {
	println("Hello, WebAssembly!")
}
```

### 执行`build`命令
```
GOARCH=wasm GOOS=js go build -o test.wasm main.go
```
注意这个是在mac或者linux操作系统下执行的命令，在windows下应该设置环境变量再执行编译命令
```powershell
$env:GOARCH="wasm";$env:GOOS="js";
go build -o test.wasm main.go
```
命令执行完后，后生成`test.wasm`文件，这个就是可以在浏览器上运行的二进制文件

### 添加其他依赖
复制`$(go env GOROOT)/misc/wasm/`下的`wasm_exec.html`和`wasm_exec.js`两个文件到当前目录


### 搭建web服务器
```go
// test.go
package main

import (
	"flag"
	"log"
	"net/http"
	"strings"
)

var (
	listen = flag.String("listen", ":8080", "listen address")
	dir    = flag.String("dir", ".", "directory to serve")
)

func main() {
	flag.Parse()
	log.Printf("listening on %q...", *listen)
	log.Fatal(http.ListenAndServe(*listen, http.HandlerFunc(func(resp http.ResponseWriter, req *http.Request) {
		if strings.HasSuffix(req.URL.Path, ".wasm") {
			resp.Header().Set("content-type", "application/wasm")
		}

		http.FileServer(http.Dir(*dir)).ServeHTTP(resp, req)
	})))
}
```
运行web服务
```
go run test.go
```

### 测试
在浏览器中打开`http://localhost:8080/wasm_exec.html`，点击页面中的run按钮，即可看到控制台打印`Hello, WebAssembly!`

这样我们就已经可以使用go编写一个可以运行在浏览器的程序了

## 如何使用js
使用go的js库`syscall/js`
```go
// main.go
package main

import "syscall/js"

func sum(args []js.Value) {
	var sum int
	for _, val := range args {
		sum += val.Int()
	}
	println(sum)
}
func registerCallbacks() {
 	js.Global().Set("sum", js.NewCallback(sum))
}
func main() {
	c := make(chan struct{}, 0)
	println("Hello, WebAssembly!")
	registerCallbacks()
	<-c
}
```
重新编译后，刷新页面，点击`run`按钮，就会为`window`对象挂载一个`sum`函数，在控制台可以调用
![image](https://note.youdao.com/yws/public/resource/47ed86ad94482e345a33baf0e9fb89eb/xmlnote/86ADB5B191C141C38442F7DB54257CD1/3794)

### 操作dom
在go中可以获取window对象来达到操作dom的效果，做一个计算器  
![image](https://note.youdao.com/yws/public/resource/47ed86ad94482e345a33baf0e9fb89eb/xmlnote/5C677199C5144A6CAC3D8A2AC9CF378B/3797)  
此示例可以在[wasm js示例](https://github.com/linjiajian999/anything/tree/master/go/js)中参考

具体代码可以参考
- [wasm示例](https://github.com/linjiajian999/anything/tree/master/go/webassembly)
- [wasm js示例](https://github.com/linjiajian999/anything/tree/master/go/js)

## 参考链接
- [webassembly home page](https://webassembly.org/)
- [go WebAssembly wiki page](https://github.com/golang/go/wiki/WebAssembly)
- [syscall/js](https://tip.golang.org/pkg/syscall/js/)
