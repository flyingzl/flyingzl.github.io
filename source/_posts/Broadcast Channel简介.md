---
title: Broadcast Channel简介
date: 2019-07-26 17:30:00
categories:
- 前端技术
tags: JavaScript
---


在前端，我们经常会用`postMessage`来实现页面间的点对点通信，对于一些需要广播（让所有页面知道）的消息，用`postMessage`无法满足需求。[`Broadcast Channel`](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API)就用来解决这个问题

![架构](https://g.asyncoder.com/20190726-1.png)



## 使用 

```javascript

// 创建广播
const channel = new BroadcastChannel('channel')

// 监听广播
channel.onmessage = e => {
    console.log(e.data)
}

// 广播事件
channel.postMessage('hello message')

// 关闭广播
channel.close()

```


## 兼容性

当前firefox和google chrome已经支持， IE 和safari还不支持

![兼容性](https://g.asyncoder.com/20190726-2.png)



## polyfill支持

可以参考 [github polyfill](https://gist.github.com/alexis89x/041a8e20a9193f3c47fb)

polyfill原理: 所有页面都可以监听storage的事件变化

```javascript
window.addEventListener('storage', e => {
    console.log(`storage key changed: ${e.key}, oldValue is ${e.oldValue}, newValue is ${e.newValue}`)
})
```

