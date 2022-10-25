---
title: 聊聊WebTransport
date: 2021-02-03 15:00:00
categories:
- 前端技术
tags: JavaScript
---

## 简介

### WebTransport 是什么？

[WebTransport](https://w3c.github.io/webtransport/)是浏览器提供的一套基于[QUIC](https://tools.ietf.org/html/draft-ietf-quic-transport-34)协议的 API 接口，方便浏览器和服务器之间进行实时数据传输，它填补了 Web 平台中的一些空白：

-   缺少类似 UDP 的网络 API
-   缺少类似于 WebSocket 但不受队头阻塞影响（Head of Line Blocking）的 API

### WebTransport 特性

Webtransport 基于 QUIC 协议，其底层是 UDP。虽然是 UDP 是不可靠的传输协议，但是 QUIC 在 UDP 的基础上融合了 TCP、TLS、HTTP/2 等协议的特性，使得 QUIC 成为一种低时延、安全可靠的传输协议。可以简单理解 QUIC 把 TCP+TLS 的功能基于 UDP 重新实现了一遍。

WebTransport 提供了如下功能特性：

-   传输可靠数据流 （类似 TCP）
-   传输不可靠数据流（类似 UDP）
-   数据加密和拥塞控制（congestion control）
-   基于 Origin 的安全模型（校验请求方是否在白名单内，类似于 CORS 的[Access-Control-Allow-Origin](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)）
-   支持多路复用（类似于 HTTP2 的 Stream）

<!--more-->

### WebTransport 适用场景

-   不考虑数据传输可靠性和数据到达到顺序的场景，比如游戏中向 服务器 发送 游戏状态 数据
-   服务器消息推送
-   其它不考虑数据达到顺序的场景

## API 使用

WebTransport 主要提供三种类型的 API

-   `datagramReadable`和`datagramWritable`，用于不可靠数据传输
-   `createBidirectionalStream`, 用于双向数据流可靠传输
-   `createUnidirectionalStream`, 用于单向数据流可靠传输

### 创建 WebTransport 对象

创建 WebTransport 对象，有一定的 URI 要求，格式为: `quic-transport://domain:port/path`，如下所示：

```javascript
const transport = new WebTransport('quic-transport://localhost:4433/counter')
await transport.ready
transport.closed.then(() => {
    console.log('webtransport closed')
})
```

`transport.ready`返回一个 Promise，如果 QUIC 连接失败会报错。

`transport.closed`也返回一个 Promise，QUIC 连接关闭时会执行

使用 WebTransport 时需要创建一个 QUIC Server， 可以基于 Python 库[aioquic](https://github.com/aiortc/aioquic)来创建服务器，也可以直接使用 Google Chrome 的[样例代码](https://github.com/GoogleChrome/samples/blob/gh-pages/quictransport/quic_transport_server.py)。

### 不可靠数据数传

WebTransport 提供了类似于 UDP 的不可靠传输接口，分别为`datagramReadable`和`datagramWritable`。前者用于读取数据，后者用于发送数据。完整的样例如下所示：

```javascript
;(async () => {
    const transport = new WebTransport(
        'quic-transport://localhost:4433/counter'
    )
    await transport.ready
    transport.closed.then(() => {
        console.log('webtransport closed')
    })

    const createUdpWriter = () => {
        const datagramWritable = transport.datagramWritable
        if (datagramWritable.locked) {
            throw new Error('previous datagram writer should be relased')
        }
        return datagramWritable.getWriter()
    }

    const createUdpReader = () => {
        const datagramReadable = transport.datagramReadable
        if (datagramReadable.locked) {
            throw new Error('previous datagram reader should be relased')
        }
        return datagramReadable.getReader()
    }

    ;(async () => {
        const udpReader = createUdpReader()

        while (true) {
            const { value, done } = await udpReader.read()

            if (done) {
                console.log('done, close datagram reader...')
                return
            }

            console.log(new TextDecoder().decode(value))
        }
    })()

    const updWriter = createUdpWriter()

    timer = setInterval(() => {
        updWriter.write(new TextEncoder('utf-8').encode(Date.now()))
    }, 1000)

    setTimeout(async () => {
        clearInterval(timer)
        // 关闭writer
        await updWriter.close()
        // 释放锁
        updWriter.releaseLock()
        // 关闭transport
        transport.close()
    }, 5 * 1000)
})()
```

在 上面代码中，核心代码为：`transport.datagramWritable.getWriter()`以及`transport.datagramReadable.getReader()`。

`transport.datagramWritable.getWriter()`返回 一个 [WritableStreamDefaultWriter](https://developer.mozilla.org/en-US/docs/Web/API/WritableStreamDefaultWriter)对象，其`write`方法用于 发送数据，注意数据必须为[TypedArray](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/TypedArray)数据类型， 代码中我们用[TextEncoder.encode()](https://developer.mozilla.org/zh-CN/docs/Web/API/TextEncoder)将字符串转为了`Uint8Array`

`transport.datagramReadable.getReader() `返回一个[ReadableStreamDefaultReader](https://developer.mozilla.org/zh-CN/docs/Web/API/ReadableStreamDefaultReader)对象，其`read`方法用于读取数据，代码中我们用[TextEncoder.decode()](https://developer.mozilla.org/zh-CN/docs/Web/API/TextEncoder)将`Uint8Array`类似数据转为了字符串。

### 双向可靠数据流

如果需要保证可靠的数据传输并且需要返回实时结果，可以通过`transport.createBidirectionalStream()`来创建可靠数据传输。这个接口的功能类似于 WebSocket，但优点在于不受对头阻塞影响。

```javascript
;(async () => {
    const transport = new WebTransport(
        'quic-transport://localhost:4433/counter'
    )
    await transport.ready
    transport.closed.then(() => {
        console.log('webtransport closed')
    })

    window.transport = transport

    const readerStream = async (reader) => {
        while (1) {
            const { value, done } = await reader.read()
            if (done) {
                return
            }
            console.log(value)
        }
    }

    timer = setInterval(async () => {
        try {
            const stream = await transport.createBidirectionalStream()
            let decoder = new TextDecoderStream('utf-8')
            let reader = stream.readable.pipeThrough(decoder).getReader()
            readerStream(reader)

            const writer = stream.writable.getWriter()
            writer.write(new TextEncoder('utf-8').encode(Date.now()))
            await writer.close()
        } catch (error) {
            console.error(error.toString())
        }
    }, 1000)

    setTimeout(() => {
        clearInterval(timer)
        // transport.close()
    }, 5 * 1000)
})()
```

注意发送可靠数据流时和 `transport.datagramWritable`有所不同，每次发送数据，都需要创建一个`transport.createBidirectionalStream()`对象，发送完毕后需要调用`close()`将其关闭。

这个就是就是多路复用，可以非常高效、低成本的同时创建多个 Stream，而且多个 Stream 之间相互独立，不像 HTTP2 那样受对头阻塞的影响。

`transport.createBidirectionalStream()`返回一个`Promise<BidirectionalStream>`, `BidirectionalStream`拥有`readable`和`writable`对象，分别用于可靠的发送数据和接受数据。

### 单向可靠数据流

如果想保证数据可靠到达，但是对返回结果不感兴趣，可以使用`transport.createUnidirectionalStream()`时来创建单向数据流，它返回一个`SendStream`对象，只可以发送数据。如果想获取返回的数据，可以调用`transport.transport.incomingUnidirectionalStreams`来获取，但是数据顺序就不保证了。一个完整的样例如下所示：

```javascript
;(async () => {
    const transport = new WebTransport(
        'quic-transport://localhost:4433/counter'
    )
    await transport.ready
    transport.closed.then(() => {
        console.log('webtransport closed')
    })

    window.transport = transport

    const readIncomingStream = async () => {
        let reader = transport.incomingUnidirectionalStreams.getReader()
        while (1) {
            const { value, done } = await reader.read()
            if (done) {
                console.log('incoming stream done...')
                return
            }
            readerStream(value)
        }
    }

    const readerStream = async (stream) => {
        let reader = stream.readable.getReader()
        while (1) {
            const { value, done } = await reader.read()
            if (done) {
                return
            }
            console.log(new TextDecoder('utf-8').decode(value))
        }
    }

    readIncomingStream()

    timer = setInterval(async () => {
        try {
            // 创建单向数据流
            const stream = await transport.createUnidirectionalStream()
            const writer = stream.writable.getWriter()
            writer.write(new TextEncoder('utf-8').encode(Date.now()))
            await writer.close()
        } catch (error) {
            console.error(error.toString())
        }
    }, 1000)

    setTimeout(() => {
        clearInterval(timer)
        // transport.close()
    }, 5 * 1000)
})()
```

## 小结

WebTransport 提供的三种 API 可以根据实际情况来使用：

1. 不需要保证数据发送先后顺序，就选择`transport.datagramWritable`
2. 需要保证数据发送顺序，但是不关心返回值，可以选择`transport.createUnidirectionalStream()`
3. 需要保证数据发送顺序且关心返回结果，就选择`transport.createBidirectionalStream()`

## 注意事项

WebTransport 现在只有 Google Chrome 支持，其标准也处于草案阶段，不建议使用生产环境。而且现阶段处于[origin trial](https://web.dev/webtransport/#register-for-ot)，也就是说，必须申请试用才可以使用。例如官方样例[https://googlechrome.github.io/samples/webtransport/client.html](https://googlechrome.github.io/samples/webtransport/client.html)就有类似的代码：

```html
<!-- QuicTransport origin trial token. See https://developers.chrome.com/origintrials/#/view_trial/-6744140441987317759 -->
  <meta http-equiv="origin-trial" content="AtxDQl4geYcHaq74wzqCV5DB6zr3+aOffkteLTTLu1+S7VJPwCiUHe2qyUh8kcez+UnKg+g79wzkhdgWvtShmAgAAABdeyJvcmlnaW4iOiJodHRwczovL2dvb2dsZWNocm9tZS5naXRodWIuaW86NDQzIiwiZmVhdHVyZSI6IlF1aWNUcmFuc3BvcnQiLCJleHBpcnkiOjE2MTQxMjQ3OTl9">
</html>
```

浏览器检测到`origin-trial`才开启 WebTransport 功能。

如果向自己玩耍 WebTransport，可以通过[mkcert](https://github.com/FiloSottile/mkcert)生成下 HTTPS 证书，然后在 Google Chrome 时加上自定参数，例如 Mac 下启动 Google Chrome 需要加上如下类似代码：

```bash
open -n -a /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --args --origin-to-force-quic-on=localhost:4433 https://googlechrome.github.io/samples/webtransport/client.html
```

具体代码可以参考[https://github.com/GoogleChrome/samples/blob/gh-pages/quictransport/quic_transport_server.py](https://github.com/GoogleChrome/samples/blob/gh-pages/quictransport/quic_transport_server.py).

以Mac为例子，在命令行依此执行如下代码，就可以启动一个 QUIC Server。

```bash
brew install mkcert
mkcert -install
mkcert localhost
python quic_transport_server.py localhost.pem localhost-key.pem
```

## 参考资料

-   [WebTransport explainer](https://github.com/w3c/webtransport/blob/main/explainer.md)
-   [Experimenting with WebTransport](https://web.dev/webtransport/)
-   [Experimenting with QUIC and WebTransport in Go](https://centrifugal.github.io/centrifugo/blog/quic_web_transport/)
-   [How to use HTTPS for local development](https://web.dev/how-to-use-local-https/)
-   [让互联网更快的“快”---QUIC 协议原理分析](https://zhuanlan.zhihu.com/p/32630510)
-   [天下武功，唯'QUICK'不破，揭秘 QUIC 的五大特性及外网表现](https://cloud.tencent.com/developer/article/1155289)
-   [QUIC 协议设计要点分析](https://zhuanlan.zhihu.com/p/60999430)
