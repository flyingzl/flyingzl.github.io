---
title: Mac上编译openSSL到WebAssembly
date: 2020-01-02 17:30:00
categories:
- 前端技术
tags: 
- JavaScript
- WASM
---


最近想利用WebAssembly来调用OpenSSL里面的MD5加密库，采用如下命令进行编译：

```bash
~ emcc md5.c -I /usr/local/Cellar/openssl@1.1/1.1.1d/include -L /usr/local/Cellar/openssl@1.1/1.1.1d/lib -lcrypto  -s EXTRA_EXPORTED_RUNTIME_METHODS='["cwrap", "ccall"]' -o md5.js
```

编译时发现了如下问题：

```bash
shared:WARNING: object /var/folders/gk/_j9gq7sn6d19820tz77q5k600000gn/T/emscripten_temp_0ybodbu3_archive_contents/x86_64cpuid.o is not a valid object file for emscripten, cannot link
error: undefined symbol: MD5_Final
warning: To disable errors for undefined symbols use `-s ERROR_ON_UNDEFINED_SYMBOLS=0`
error: undefined symbol: MD5_Init
error: undefined symbol: MD5_Update
Error: Aborting compilation due to previous errors
shared:ERROR: '/usr/local/bin/node /usr/local/Cellar/emscripten/1.38.44/libexec/src/compiler.js /var/folders/gk/_j9gq7sn6d19820tz77q5k600000gn/T/tmpmptz1ty9.txt /usr/local/Cellar/emscripten/1.38.44/libexec/src/library_pthread_stub.js' failed (1)
```


Google和StackOverflow下才发现，Mac系统自带的OpenSSL在WebAssembly中是没法直接是用的；因为WebAssembly和操作系统无关，不能直接使用和操作系统相关的库文件，我们需要利用`Emscripten`重新编译下。

##  编译openSSL

首先下载最新的OpenSSL，这里我下载的是 [https://github.com/openssl/openssl/releases/tag/OpenSSL_1_1_1d](https://github.com/openssl/openssl/releases/tag/OpenSSL_1_1_1d)，现在后解压，进入`openssl-OpenSSL_1_1_1d`文件夹。

```bash
emcmake ./Configure  darwin64-x86_64-cc -no-asm --api=1.1.0
```

### 修改Makefile文件

- 将 `CROSS_COMPILE=/usr/local/Cellar/emscripten/1.38.44/libexec/em` 改为 `CROSS_COMPILE=`
- 将 `CNF_CFLAGS=-arch x86_64` 改为 `CNF_CFLAGS=`


> 这步非常重要，如果不修改，容易出现编译错误!


### 编译OpenSSL

```bash
emmake make -j 12 build_generated libssl.a libcrypto.a
mkdir -p ~/Downloads/openssl/libs
cp -R include ~/Downloads/openssl/include
cp libcrypto.a libssl.a ~/Downloads/openssl/libs
```
编译成功后，文件夹下会出现`libssl.a`和`libcrypto.a`两个文件，后续编译会需要这两个文件

## 调用 OpenSSL

### 样例准备

```c
// md5.c
#include <emscripten.h>
#include <openssl/md5.h>
#include <string.h>
#include <stdio.h>

EMSCRIPTEN_KEEPALIVE
void md5(char *str, char *result) {
  MD5_CTX md5_ctx;
  int MD5_BYTES = 16;
  unsigned char md5sum[MD5_BYTES];
  MD5_Init(&md5_ctx);
  MD5_Update(&md5_ctx, str, strlen(str));
  MD5_Final(md5sum, &md5_ctx);
  char temp[3] = {0};
  memset(result, sizeof(char) * 32, 0);
  for (int i = 0; i < MD5_BYTES; i++) {
    sprintf(temp, "%02x", md5sum[i]);
    strcat(result, temp);
  }
  result[32] = '\0';
}
```

### 代码编译
```bash
emcc md5.c -I ~/Downloads/openssl/include -L ~/Downloads/openssl/libs -lcrypto -s EXTRA_EXPORTED_RUNTIME_METHODS='["cwrap", "ccall"]' -o md5.js
```

编译成功后，会生成`md5.js`和`md5.wasm`两个文件。

### Node.js 调用

```javascript
const m = require('./md5.js')

const mallocByteBuffer = len => {
    const ptr = m._malloc(len)
    const heapBytes = new Uint8Array(m.HEAPU8.buffer, ptr, len)
    return heapBytes
}

// md5.wasm加载完毕后会执行该回调
m.onRuntimeInitialized = () => {
    const md5 = m.cwrap('md5', null, ['number', 'number'])
    // 需要进行md5加密的字符串
    const str = 'admin'
    const array = str.split('').map(v => v.charCodeAt(0))
    const inBuffer = mallocByteBuffer(array.length)
    inBuffer.set(array)
    const outBuffer = mallocByteBuffer(32)
    // 调用wasm接口
    md5(inBuffer.byteOffset, outBuffer.byteOffset)
    console.log(Array.from(outBuffer).map(v => String.fromCharCode(v)).join('')) 
    // 输出结果为：21232f297a57a5a743894a0e4a801fc3
}
```

## 预编译文件

为了方便大家使用，我已经编译好了针对WebAssembly的openSSL，点击[下载](https://raw.githubusercontent.com/flyingzl/webAssembly-openssl/master/openssl-emscripten.zip)。

本文涉及到的样例位于[https://github.com/flyingzl/webAssembly-openssl](https://github.com/flyingzl/webAssembly-openssl)，请自行查阅。

## 参考资料

- [wapm-packages/OpenSSL](https://github.com/wapm-packages/OpenSSL/blob/master/build.sh)
- [TrueBitFoundation/wasm-ports](https://github.com/TrueBitFoundation/wasm-ports/blob/master/openssl.sh)

