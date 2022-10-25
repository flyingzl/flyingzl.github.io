---
title: 利用Golang玩转WebAssembly
date: 2018-08-27 14:07:51
categories:
- 后端技术
tags: 
- WASM
- Go
---

WebAssembly，也叫WAMS，是一个可移植、体积小、加载快并且兼容 Web 的全新二进制格式，可以将C/C++/Rust等语言编写的代码编译为wasm文件然后运行在Web上。

用C/C++写wasm有点麻烦，需要安装各种编译器。话说最新[Go 1.11](https://blog.golang.org/go1.11)发布了，其中有个新特性就是对WebAssembly的支持，今天我们就来玩玩，如何将Go代码编译为wasm文件。

*提示：本文不会介绍Go的相关语法以及WASM的相关基础知识。如果阅读有困难，请先参考相关的基础介绍或阅读相关文章。*




## 环境安装

环境安装就不多说了，直接参考[官方安装文档](https://golang.org/doc/install.html)就行，然后配置好对应的环境变量即可。例如在windows上，运行如下命令OK，就说明都正常了：

```bash
C:\Users\Administrator>go version

go version go1.11 windows/amd64
```

## 来个栗子🌰

先建立一个`main.go`的文件，内容如下：

```go
package main

func main() {
	println("Hello, WebAssembly!")
}
```

代码很简单，就是程序运行后直接在控制台输出`Hello, WebAssembly!`

然后进入控制台（windows上需要先进入git bash），输入如下命令：

```bash
GOARCH=wasm GOOS=js go build -o test.wasm main.go
```

这句话的意思是，切换Go的编译架构为wasm，运行的的对象为Javascript引擎，然后源码为`main.go`，最后输出为`test.wasm`这个二进制文件

生成的`test.wasm`需要加载到浏览器中才可以使用，为了方便使用，Go已经提供了默认的加载样例和脚本，可以通过把脚本和样例复制一下：

```bash
cp $(go env GOROOT)/misc/wasm/wasm_exec.{html,js} .
```

上面的意思是把Go安装目录下的`wasm_exec.html`和`wasm_exec.js`复制到当前文件夹。一切就绪后，我们用`http-server`（可以通过npm install http-server来安装）启动一个Web服务器：

```bash
http-server -c 0 -p 2000
```

访问 [http://localhost:2000/wasm_exec.html](http://localhost:2000/wasm_exec.html)并打开控制台，会看到如下内容：

```
Hello, WebAssembly!
```

## 深入样例

直接显示Hello world之类的提示不好玩，我们能否调用下Go里面的方法呢？ 答案是可以的。我们首先修改下`wasm_exec.html`，改成如下内容：

```html
<!doctype html>
<html>

<head>
	<meta charset="utf-8">
	<title>Go wasm</title>
</head>

<body>
	<script src="wasm_exec.js"></script>
	<script>
		if (!WebAssembly.instantiateStreaming) { // polyfill
			WebAssembly.instantiateStreaming = async (resp, importObject) => {
				const source = await (await resp).arrayBuffer()
				return await WebAssembly.instantiate(source, importObject)
			}
		}

		const go = new Go()
		let mod, inst
		WebAssembly.instantiateStreaming(fetch("test.wasm"), go.importObject).then( async result=> {
			mod = result.module
			inst = result.instance
			await go.run(inst)
        })

		function wamsCallback(value) {
			console.log(`wasm output: ${value}`)
		}
	</script>

	<p>Hello</p>
	<input id="name" />
</body>

</html>
```

接下来修改下`main.go`，加入如下代码：

```go
package main

import (
	"syscall/js"
)

func calFib(n int) int {
	a := 0
	b := 1

	for i := 0; i < n; i++ {
		a, b = b, a+b
	}
	return b
}

func fib(params []js.Value) {
	value := params[0].Int()
	value = calFib(value)
    // 当前Go和wasm交互，wasm没法直接获得函数的返回值，调用window.wamsCallback(value)或者直接window.output获取
    // window.wamsCallback为用户在Javascript中自定义的函数，也就是一个回调函数
	js.Global().Set("output", value)
	js.Global().Call("wamsCallback", value)
}
 
// 将Go里面的方法注入到window.fibNative里面
func registerCallbacks() {
	js.Global().Set("fibNative", js.NewCallback(fib))
}

func main() {
	registerCallbacks()
	select {}
}
```

上面的代码中，我们在Go中增加了一个计算斐波那契数列的方法，并在Go中通过`registerCallbacks`这个方法注入到浏览器里面。`js.Global()`表示获取宿主环境的window（浏览器）或者global（Node.js）。`Call`表示调用对应的方法。所以wasm成功加载到浏览器中后，我们可以通过`window.fibNative`这个函数来访问Go中的`fib`方法。

Go现在对wasm的支持属于试验阶段，相关API还不完善，我们现在还没法直接获得返回值。在代码中，我们可以将返回值通过`js.Global().Set("output", value)`将计算的返回值直接写到`window.output`里面，或者调用我们页面中已有的函数。

在Chrome控制台，我们可以得到这样的结果：

```bash
Hello, WebAssembly!

> window.fibNative(3)
wasm_exec.html:28 wasm output: 3
> window.output
3

> window.fibNative(30)
wasm_exec.html:28 wasm output: 1346269
> window.output
1346269
```

在Go中，通过`syscall/js`这个官方提供的开发库还是可以调用页面中的DOM并操作，例如：

```go
package main

import (
	"syscall/js"
)

func changeBodyColor(color []js.Value) {
	// document.body.style.color = color
	js.Global().Get("document").Get("body").Set("style", "color:"+color[0].String())
}

func setInputValue(val []js.Value) {
	id := val[0].String()
	// document.getElementById(id).value = "value from Go"
	js.Global().Get("document").Call("getElementById", id).Set("value", "value from Go")
}

// 将Go里面的方法注入到window.fibNative里面
func registerCallbacks() {
	js.Global().Set("changeBodyColor", js.NewCallback(changeBodyColor))
	js.Global().Set("setInputValue", js.NewCallback(setInputValue))
}

func main() {
	registerCallbacks()
	select {}
}

```

重新编译为wasm后，可以通过`window.changeBodyColor("red")` 和 `setInputValue("name")`来切换页面颜色以及给文本框一个默认值。


再来一个计算数组和的样例：

```go
package main

import (
	"syscall/js"
)

func sum(params []js.Value) {
	result := 0
	for _, value := range params {
		result += value.Int()
	}
	js.Global().Call("wamsCallback", result)
}

func registerCallbacks() {
	js.Global().Set("sumNative", js.NewCallback(sum))
}

func main() {
	println("Hello, WebAssembly!")
	registerCallbacks()
	select {}
}
```

这样，我们可以通过`window.sumNative(1, 2,3)`来调用Go的计算和的方法


## 总结

以上是在Go中使用wasm的基本方法。当前Go对wasm的支持还属于实验阶段，官方也提到现在提供的API也不多，重要用于测试和验证。还有就是，当前的wasm二进制文件大小大约为1.2Mb左右，比于C生成的wasm要大不少，主要是里面包含的Go的一些运行时，所以会大点。以后也许会优化。

## 参考资料

- [Go中的WebAssembly](https://github.com/golang/go/wiki/WebAssembly)
- [Go WebAssembly Tutorial - Building a Calculator](https://www.youtube.com/watch?v=4kBvvk2Bzis)
- [golang-wasm-example](https://github.com/Chyroc/golang-wasm-example)

