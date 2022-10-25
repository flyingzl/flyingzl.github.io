---
title: whistle使用介绍
date: 2019-11-29 15:00:00
categories:
- 前端技术
tags: 
- JavaScript
- Web调试
---

whistle(读音[ˈwɪsəl])基于Node实现的跨平台web调试代理工具，类似的工具有Windows平台上的Fiddler和Mac上的Charles，主要用于查看、修改HTTP、HTTPS、Websocket的请求、响应，也可以作为HTTP代理服务器使用。

![简介](https://g.asyncoder.com/images/20191029/whistle.png)


## 安装

```bash
$ npm install -g whistle
```

或者通过`npm taobao` 安装

```bash
npm install whistle -g --registry=https://registry.npm.taobao.org
```

## 启动

```bash
$ w2 --version
2.3.0
$ start -p 3333
[!] whistle@2.3.0 is running
[i] 1. use your device to visit the following URL list, gets the IP of the URL you can access:
       http://127.0.0.1:3333/
       http://192.168.31.107:3333/
       Note: If all the above URLs are unable to access, check the firewall settings
             For help see https://github.com/avwo/whistle
[i] 2. configure your device to use whistle as its HTTP and HTTPS proxy on IP:3333
[i] 3. use Chrome to visit http://local.whistlejs.com/ to get started
➜  ~
```

## 访问

设置好代理

![设置代理](https://g.asyncoder.com/images/20191029/1.jpg)

- 直接访问 [http://127.0.0.1:3333](http://localhost:3333)
- 在设置好代理后访问 [http://local.whistlejs.com/](http://local.whistlejs.com/)


![访问效果](https://g.asyncoder.com/images/20191029/2.jpg)


## 使用


### 抓包

网络抓包是一个抓包工具的必备功能

![抓包](https://g.asyncoder.com/images/20191029/20.jpg)

### 设置host

将某个域名指向指定的ip，这样可以绕过本地DNS的解析。比如访问[http://larry.dev](http://larry.dev)就相当于访问`http://192.168.3.107`

![设置host](https://g.asyncoder.com/images/20191029/3.jpg)


### 端口转发

可以将某个域名映射到某个ip和端口，比如访问 [http://dev.asyncoder.com](http://dev.asyncoder.com]) 就相当于访问`http://192.168.3.107:3000`。这种场景非常适合在H5下联调微信分享等场景。


![设置host](https://g.asyncoder.com/images/20191029/4.jpg)

访问效果： 

![ip访问](https://g.asyncoder.com/images/20191029/6.jpg)

![域名访问](https://g.asyncoder.com/images/20191029/5.jpg)


### 代码注入

#### 脚本注入

![引用脚本](https://g.asyncoder.com/images/20191029/8.jpg)

![脚本定义](https://g.asyncoder.com/images/20191029/9.jpg)

![脚本注入效果](https://g.asyncoder.com/images/20191029/7.jpg)


#### 样式注入


![引用样式](https://g.asyncoder.com/images/20191029/10.jpg)

![样式定义](https://g.asyncoder.com/images/20191029/11.jpg)

![样式注入效果](https://g.asyncoder.com/images/20191029/12.jpg)

#### HTML内容注入

![引用HTML文件](https://g.asyncoder.com/images/20191029/13.jpg)

![HTML文件内容](https://g.asyncoder.com/images/20191029/14.jpg)

![HTML注入效果](https://g.asyncoder.com/images/20191029/15.jpg)


### 线上文件替换

可以将线上的文件映射到本地， 本地文件修改后可以调试线上效果

![线上文件注入](https://g.asyncoder.com/images/20191029/16.jpg)

![本地文件内容](https://g.asyncoder.com/images/20191029/19.jpg)

![线上文件注入内容](https://g.asyncoder.com/images/20191029/18.jpg)

![线上文件注入效果](https://g.asyncoder.com/images/20191029/17.jpg)


## 其它

更多请参考官方文档[http://wproxy.org/whistle/quickstart.html](http://wproxy.org/whistle/quickstart.html)





