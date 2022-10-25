---
title: WebAssembly简介
date: 2019-12-20 17:00:00
categories:
- 前端技术
tags: 
- JavaScript
- WASM
---

WebAssembly是一个可移植、体积小、加载快并且兼容 Web 的全新二进制格式，其文件后缀名为wasm，是由主流浏览器厂商组成的 W3C 社区团体 制定的一个新的规范。


![wasm示意图](https://g.asyncoder.com/images/2019{mon}2014644-wasm.png)



主流浏览器已经支持WebAssembly

![wasm浏览器支持情况](https://g.asyncoder.com/images/2019{mon}201483-wasm_browsers.png)

![wasm浏览器支持情况](https://g.asyncoder.com/images/2019{mon}2014955-wasm_browsers_caniuse.jpg)


## 环境安装

wasm是一种二进制格式，主流的C/C++、Go以及Rust都可以编译wasm，[Emscripten](https://emscripten.org/)就是一个用于将C/C++编译为wasm的编译器工具集。

安装方式可以参考 [Emscripten安装指南](https://emscripten.org/docs/getting_started/downloads.html)，如果是Mac用户并安装了brew，可以直接`brew install emscripten`进行安装，更为便捷。

```bash
~ emcc -v
emcc (Emscripten gcc/clang-like replacement + linker emulating GNU ld) 1.38.44
clang version 6.0.1  (emscripten 1.38.44 : 1.38.44)
Target: x86_64-apple-darwin19.0.0
Thread model: posix
InstalledDir: /usr/local/Cellar/emscripten/1.38.44/libexec/llvm/bin
shared:INFO: (Emscripten: Running sanity checks)

```

如果运行`emcc -v`可以正常，表示`Emscripten`已经正常安装成功


## 样例

### 永远的hello world

```c
// hello.c
#include <stdio.h>
int main(int argc, char **argv) {
    printf("hello world\n");
    return 0;
}

```

运行`emcc hello.c -o hello.html` 进行编译，编译后会生成三个文件。

```bash
├── hello.c
├── hello.html
├── hello.js
└── hello.wasm
```

运行 `emrun --port 8080 .`可以自动打开浏览器

![hell world](https://g.asyncoder.com/images/2019{mon}20142048-wasm-helloworld.jpg)

也可以运行 `emcc hello.c -o hello.js`，这样不生成`hello.html`文件，可以通过nodejs直接运行

```bash
~ emcc hello.c -o hello.js
~ node hello.js
hello world
```

### 函数调用

```c
// fib.c
#include <stdio.h>
#include <emscripten.h> 

void print();

// EMSCRIPTEN_KEEPALIVE是一个宏，用于标识函数名不被修改
int EMSCRIPTEN_KEEPALIVE fib(int n){
    if(n == 0 || n == 1)
        return 1;
    else
        return fib(n - 1) + fib(n - 2);
}

void greet() {
    printf("hello world\n");
}

```

运行`emcc fib.c -o fib.js -s EXPORTED_FUNCTIONS='["_greet"]' -s EXTRA_EXPORTED_RUNTIME_METHODS='["cwrap", "ccall"]'`进行编译

接下来，我们来调用`fib.js`

```html
   <script src="fib.js"></script>
   <script>
       Module.onRuntimeInitialized = () => {
           // 通过Module.cwrap调用
           const fib = Module.cwrap('fib', 'number', ['number'])
           console.log(`fib(10)=${fib(4)}`)

           // 通过Module.ccall调用
           const result = Module.ccall('fib', 'number', ['number'], [4])
           console.log(`fib(10)=${result}`)

           // 调用greet
           const showGreet = Module.cwrap('greet', null,  null)
           showGreet()

       }
   </script>
```
`cwrap`和`ccall`都是调用wasm里面的原生函数，cwrap的参数为`cwrap(functionName, returnType, argsType)`, `ccall`的参数为`ccall(functionName, returnType, argsType, parameters)`。前者返回一个函数，后者直接 返回执行结果


### 传递指针

```c
// pointer.c

#include <stdio.h>
#include <emscripten.h> // note we added the emscripten header

// 交换数字
void EMSCRIPTEN_KEEPALIVE swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

// 数组里面的每个数字加1
void EMSCRIPTEN_KEEPALIVE addOne(int* input_ptr, int* output_ptr, int len){
	int i;
	for(i = 0; i < len; i++) {
    	output_ptr[i] = input_ptr[i] + 1;
    }
}

// 统计字符串里面某个数字的出现次数
int EMSCRIPTEN_KEEPALIVE countOccurrences(const char * str, int len, char target) {
	int i, count = 0;
	for(i = 0; i < len; i++){
    	if(str[i] == target){
        	count++;
    	}
	}
	return count;
}

```

运行`emcc pointer.c -o pointer.js -s EXTRA_EXPORTED_RUNTIME_METHODS='["cwrap", "ccall", "getValue", "setValue"]'`进行编译。这次我们用Node.js来执行`pointer.js`

```javascript
// pointer_demo.js
const m = require('./pointer.js')

const mallocByteBuffer = len => {
    const ptr = m._malloc(len)
    const heapBytes = new Uint8Array(m.HEAPU8.buffer, ptr, len)
    return heapBytes
}

const mallocUInt32Buffer = len =>  {
    const ptr = m._malloc(len * 4)
    const heapBytes = new Uint32Array(m.HEAPU8.buffer, ptr, len)
    return heapBytes
}

const free  = nativeBuffer =>  {
    if (typeof nativeBuffer === 'number') {
        m._free(nativeBuffer)
    } else if (nativeBuffer && nativeBuffer.buffer) {
        nativeBuffer.buffer instanceof ArrayBuffer && m._free(nativeBuffer.byteOffset)
    }
}

// 交换数字
const swapNumbers = ({first = 0, second  = 0} = {}) => {
    var aPtr = m._malloc(4)
    var bPtr = m._malloc(4)
    m.setValue(aPtr, first, 'i32')
    m.setValue(bPtr, second, 'i32')
    m.ccall('swap', null, ['number', 'number'], [aPtr, bPtr])
    return {
        first: m.getValue(aPtr, 'i32'),
        second: m.getValue(bPtr, 'i32')
    }
}

// 数组数字加1
const plusArrays = (numbers = []) => {
    // 传递数组
    const length = numbers.length
    const inputBuffer = mallocUInt32Buffer(length)
    const outputBuffer = mallocUInt32Buffer(length)
    // 填充数据
    inputBuffer.set(numbers)
    m.ccall('addOne', null, ['number', 'number', 'number'], [inputBuffer.byteOffset, outputBuffer.byteOffset, length])
    const outputArray = new Int32Array(outputBuffer.buffer, outputBuffer.byteOffset, length)
    free(inputBuffer)
    free(outputBuffer)
    return Array.from(outputArray)
}

// 检测字符串中某个字符出现的次数
const countOccurrences = (words, target = '') => {
    // 传递字符串
    const countOccurrences = m.cwrap("countOccurrences", "number", ["number", "number", "number"]) 
    const array = words.split('').map(v => v.charCodeAt(0))
    const length = array.length
    const inputBuffer = mallocByteBuffer(length)
    inputBuffer.set(array)
    const counts = countOccurrences(inputBuffer.byteOffset, length, target.charCodeAt(0))
    free(inputBuffer)
    return counts
}

m.onRuntimeInitialized = () => {
    const numbersObj = swapNumbers({ first: 4, second: 5 })
    console.log(numbersObj)

    const plusNumbers = plusArrays([1, 2, 3, 4, 5])
    console.log(plusNumbers)

    const count = countOccurrences('WebAssembly', 's')
    console.log(`count=${count}`)
}
```

运行结果：

```bash
~ node pointer_demo.js
{ first: 5, second: 4 }
[ 2, 3, 4, 5, 6 ]
count=2
```

### 调用C++

```cpp
// binding.cpp
#include <emscripten.h>
#include <emscripten/bind.h>
#include <iostream>

using namespace emscripten;


class Person {
public:
  Person(std::string name, int age) : name(name), age(age) {}

  void plusAge() { ++age; }

  int getAge() const { return age; }

  void setAge(int age_) { age = age_; }

  void toString() {
    std::cout << "(name=" << name << ", age=" << age << ")" << std::endl;
  }

  static std::string getStringFromInstance(const Person &instance) {
    return instance.name;
  }
private:
  int age;
  std::string name;
};

// Binding code
EMSCRIPTEN_BINDINGS(person) {
  class_<Person>("Person")
      .constructor<std::string, int>()
      .function("plusAge", &Person::plusAge)
      .function("toString", &Person::toString)
      .property("age", &Person::getAge, &Person::setAge)
      .class_function("getStringFromInstance", &Person::getStringFromInstance);
}
```

运行 `emcc --bind binding.cpp -o binding.js`。这里利用了`Emscripten`的[embind](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#embind)功能，它可以直接把C++里面的类或者函数映射为Javascript里面的对象。

```javascript
//binding_demo.js
const m = require('./binding.js')

m.onRuntimeInitialized = () => {
    const Person = m.Person
    const person = new Person('larry', 20)
    person.plusAge()
    person.plusAge()
    console.log(`person.age = ${person.age}`)
    console.log(Person.getStringFromInstance(person))
    person.toString()
}
```

执行结果
```bash
~  node binding_demo.js
person.age = 22
larry
(name=larry, age=22)
```

# 其它

更多的内容请参考[Emscripten](https://emscripten.org/docs/getting_started/index.html)官网。本文涉及到的样例请点击[下载](https://g.asyncoder.com/images/202012193556-wasm-demo.zip)。