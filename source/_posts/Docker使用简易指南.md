---
title: Docker使用简易指南
date: 2020-12-11 15:00:00
categories:
- 系统软件
tags: 
- Docker
- Node.js
---



##  Docker 是什么？

###  定义

**Docker** 是一个基于Go语言开发的应用容器引擎，它遵循了Apache2.0协议开源。**Docker**可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的Linux服务器，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口（类iPhone的app），并且容器开销极其低。

###  同类技术

开源的VirtualBox、QEMU(quick emulator)、KVM, 商业VMware、 Parallels desktop

### 优缺点

![技术对比](https://g.asyncoder.com/images/20201210135256-v2-0327d6666507f8716c3d3630fa4cb671_r.jpg)



Docker容器可以在Mac/Windows/Linux等主机上运行，并与其他容器共享主机的内核，它运行的一个独立的进程，不占用其他任何可执行文件的内存，非常轻量。Docker镜像一般都是MB级别。

虚拟机运行的是一个完成的操作系统，通过虚拟机管理程序对主机资源进行虚拟访问，相比之下需要的资源更多，虚拟机镜像都是GB级别。

当然虚拟机的隔离性更好 ，因为它完全虚拟了一个操作系统，在隔离性特别高的场景下，用虚拟机更为合适，比如一个我们买的VPS，都是一个虚拟机。


## Docker架构

![Docker Engine Components Flow](https://g.asyncoder.com/images/20201211155705-engine-components-flow.png)

![img](https://g.asyncoder.com/images/20201210140553-docker-arch1.jpg)



Docker 是一个 C/S 模式的架构，后端是一个松耦合架构，模块各司其职。

1. 用户是使用 Docker Client 与 Docker Daemon 建立通信，并发送请求给后者。
2. Docker Daemon 作为 Docker 架构中的主体部分，首先提供 Docker Server 的功能使其可以接受 Docker Client 的请求。
3. Docker Engine 执行 Docker 内部的一系列工作，每一项工作都是以一个 Job 的形式的存在。
4. Job 的运行过程中，当需要容器镜像时，则从 Docker Registry 中下载镜像，并通过镜像管理驱动 Graphdriver 将下载镜像以 Graph 的形式存储。
5. 当需要为 Docker 创建网络环境时，通过网络管理驱动 Networkdriver 创建并配置 Docker容器网络环境。
6. 当需要限制 Docker 容器运行资源或执行用户指令等操作时，则通过 Execdriver 来完成。
7. Libcontainer 是一项独立的容器管理包，Networkdriver 以及 Execdriver 都是通过 Libcontainer 来实现具体对容器进行的操作。



## 一些术语

Docker 包括三个基本概念:

- **镜像（Image）**：相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu 16.04 最小系统的 root 文件系统。
- **容器（Container）**：镜像（Image）和容器（Container）的关系，就像是**面向对象程序设计中的类和实例一样**，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
- **仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像，类似于npm的registry，用户可以 上传自己制作好的镜像。



![img](https://g.asyncoder.com/images/20201210141206-576507-docker1.png)

## 为什么会出现

### 解决什么问题

- Docker解决了运行环境和配置问题，方便发布，方便做持续集成，完美解决【在我的电脑上能跑，在XX环境就跑步了】的情况
- 更轻量的虚拟化，节省了虚拟机的性能损耗

### 没这个技术之前怎么作

没使用Docker之前，用户会直接使用虚拟机部署来解决，这样导致：

- 资源利用效率低
- 单物理机多应用无法有效隔离
- 运维部署不方便
- 测试、版本管理复杂
- 传统虚拟机占用空间大、启动慢、管理复杂



## 快速上手

###  Docker 环境安装

[Docker Desktop for Mac](https://desktop.docker.com/mac/stable/Docker.dmg)

[Docker Desktop for Windows](https://desktop.docker.com/win/stable/Docker%20Desktop%20Installer.exe)

![image-20201210142314530](https://g.asyncoder.com/images/20201210142316-image-20201210142314530.png)



为了加快镜像拉取速度，需要修改下`Docker Engine`的镜像地址：

```json
{
    "debug":true,
    "experimental":false,
    "registry-mirrors":[
        "https://75r8zlet.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com/"
    ]
}
```



### Docker运行过程

Docker的运行过程可以简单的理解为，从远端仓库中拉取合适的镜像到本地-->通过镜像创建容器-->启动容器                                                                

![image-20201210173545494](https://g.asyncoder.com/images/20201210173546-image-20201210173545494.png)

## Docker 容器使用

### Hello World运行

```bash
➜  docker run --rm hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pulling fs layer
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete
Digest: sha256:e7c70bb24b462baa86c102610182e3efcb12a04854e8c582838d92970a09f323
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### 查看镜像

`docker images`用于查看本地已拉取 镜像

```bash
➜  ~ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
flyingzl/mrtc-server       v2                  b7fa95c740d8        21 hours ago        218MB
flyingzl/invoice-ocr       latest              b5354990d5d4        5 days ago          8.35MB
node                       latest              2d840844f8e7        2 weeks ago         935MB
node                       lts-alpine3.12      870de9c16886        3 weeks ago         117MB
electronuserland/builder   wine                2b534717a8c8        4 months ago        1.88GB
mysql                      latest              e3fcc9e1cc04        4 months ago        544MB
nginx                      latest              8cf1bfb43ff5        4 months ago        132MB
mongo                      latest              6d11486a97a7        5 months ago        388MB
alfg/nginx-rtmp            latest              db65d3c185cf        7 months ago        115MB
kagamihi/ffmpeg.js         latest              09fd1823daed        7 months ago        1.38GB
```



### 创建容器

`docker run`用于创建一个容器，容器创建成功后，会返回一个container ID, 通过改ID可以对容器进行停止、更新 或者删除操作。

`docker run -i -d --name node-demo node`可以生成一个名为`node-demo`的容器

```bash
 ~ docker run -i -d --name node-demo node
1f396e48e47c8c5fd6ee6851ff5436317c92258dbc92bda67ca4b5a24e0442cd
```



执行`docker ps`可以看到当前正在运行的容器

 ![image-20201210153731282](https://g.asyncoder.com/images/20201210153733-image-20201210153731282.png)



### 进入容器

执行`docker exec -it container-id bash`可以进入容器 内部

这里`-i`打表`interactive`,表示可交互， `-t`表示`tty`，模拟一个伪终端。`-i`和`-t`一般是配合使用

![image-20201210154026575](https://g.asyncoder.com/images/20201210154028-image-20201210154026575.png)

可以 看到， 容器其实就是一个Linux运行环境

### 端口映射

默认情况下，容器和宿主机的网络是不通的，例如我在上文创建的容器中安装一个`http-server`,然后执行`http-server -p 3000`,在外部是没法访问的。

![image-20201210154314535](https://g.asyncoder.com/images/20201210154315-image-20201210154314535.png)



通过`docker run  -i -d --name node-demo-port -p 3000:3000 node` 可以将本地的3000端口 和容器的3000端口进行映射， 这样就可以访问到容器

![image-20201210160250881](https://g.asyncoder.com/images/20201210160252-image-20201210160250881.png)

如果想查看容器的端口映射情况，可以运行`docker port container-id`来进行查看

```bash
➜  ~ docker port ubuntu
5900/tcp -> 0.0.0.0:5900
80/tcp -> 0.0.0.0:6080
➜  ~
```

### 停止容器

`docker stop container-id/container-name`可以停止容器

`docker stop $(docker ps -aq)`可以量停止所有容器

### 启动容器

`docker start container-id/container-name`可以启动之前停止的 容器

`docker start $(docker ps -aq)`可以量启动之前所有停止的容器

### 删除容器

`docker rm container-id/container-name`可以停止容器

`docker rm $(docker ps -aq)`可以删除所有容器

> *如果某个容器正在运行中*，删除容器时会提示错误`Error response from daemon: You cannot remove a running container XXXXX. Stop the container before attempting removal or force remove`

如果想强制删除，可以执行`docker rm -f container-id`



### 创建镜像

使用某个容器后，如果进入容器后安装了新的软件或者增加新的功能，可以基于修改后的容器新建一个镜像，比如：

```bash
# 创建容器 
docker run -id --name node-demo node

# 进入容器
docker exec -it node-demo bash

# 安装一些新的软件, 更新下载镜像源
root@688e8e176515:/# apt-get update
root@688e8e176515:/# apt-get install apt-transport-https
root@688e8e176515:/# sed -i 's#http://deb.debian.org#https://mirrors.163.com#g' /etc/apt/sources.list
root@688e8e176515:/# apt-get  update
root@688e8e176515:/# apt-get install vim  -y
root@688e8e176515:/# npm  install http-server -g

# 退出容器,将容器重新制作为一个镜像
docker commit -a zhaolei -m "docker with node and vim" node-demo node-vim:latest

# 重新创建容器
docker run -id --name node-vim-demo node-vim
```



镜像也可以从从外部导出导出，可以参考 [Docker镜像导入导出](https://www.hangge.com/blog/cache/detail_2411.html)一文：

- `docker export`、`docker import` 
- `docker save`、`docker load`

### 删除镜像

`docker rmi image-id/image-name`可以删除镜像

`docker rmi $(docker images -aq)`可以删除所有镜像

### 文件拷贝

`docker cp local-file  conainer:container-path `复制宿主本地文件到容器

`docker cp container:container-path local-file` 复制容器文件到宿主本地 

![image-20201210163239033](https://g.asyncoder.com/images/20201210163240-image-20201210163239033.png)



### 文件目录映射

Docker可以将宿主机的某个目录 映射到 容器，这样宿主机对文件进行修改，容器可以 读取到；反之怡然。

可以增加`-v`参数即可。

![image-20201210164355778](https://g.asyncoder.com/images/20201210164357-image-20201210164355778.png)

### 查看日志

`docker logs -f containerId` 用于查看容器日志

![image-20201210163642819](https://g.asyncoder.com/images/20201210163644-image-20201210163642819.png)



### 输出格式化

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"

alias dps='docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"'
```

![image-20201210163716443](https://g.asyncoder.com/images/20201210163718-image-20201210163716443.png)



## 实战

### 发票识别

`docker run --rm -e USER=石头 -v ~/Downloads/images/:/opt/invoice/images flyingzl/invoice-ocr` 用于直接识别图片。

`~/Downloads/images`表示本地存放发票图片的路径，根据本机图片目录进行修改

`USER`用于修改 输出结果的显示名称

![image-20201210164529246](https://g.asyncoder.com/images/20201210164530-image-20201210164529246.png)



### 基于Nginx的RTMP推流

1. `docker run --rm -itd -p 1935:1935 -p 8080:80 alfg/nginx-rtmp` 拉取并运行容器

2. `ffmpeg -re -stream_loop -1 -i ~/Movies/霸王龙.mp4 -f flv 表示rtmp://127.0.0.1:1935/stream/demo`进行 视频循环推流， `demo`表示流名称 
3. `ffplay http://127.0.0.1:8080/live/demo.m3u8`可以 播放HLS视频流



![image-20201210165552582](https://g.asyncoder.com/images/20201210165554-image-20201210165552582.png)



###  Mac上构建Electron Windows应用

由于Mac上通过`electron-builder`来生成windows应用， 需要本机安装`wine`来模拟环境，比较麻烦，而且污染系统。可以考虑通过一个名为[electronuserland/builder](https://hub.docker.com/r/electronuserland/builder/)的Docker镜像来进行快速构建

```bash
cd electron
docker run -it --rm \
--env ELECTRON_MIRROR="https://npm.taobao.org/mirrors/electron/" \
--env ELECTRON_CACHE="/root/.cache/electron" \
--env ELECTRON_BUILDER_CACHE="/root/.cache/electron-builder" \
-v ${PWD}:/project \
-v ${PWD##*/}-node-modules:/project/node_modules \
-v ~/.cache/electron:/root/.cache/electron \
-v ~/.cache/electron-builder:/root/.cache/electron-builder \
electronuserland/builder:wine

```

安装好依赖，就 可以进行构建 windows应用了。



### 可视化管理Docker

可以运行安装`portainer/portainer-ce`镜像对docker进行可视化管理。



```bash
docker volume create portainer_data

docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

![image-20201210175414519](https://g.asyncoder.com/images/20201210175415-image-20201210175414519.png)

###  创建本地镜像

创建本地镜像需要事先生成一个`Dockerfile`文件，然后 执行输入类似下面的内容：

```bash
FROM node:lts-alpine3.12

RUN echo "http://mirrors.aliyun.com/alpine/v3.12/main/" > /etc/apk/repositories

LABEL maintainer="flyingzl"

RUN apk update \
        && apk upgrade \
        && apk add --no-cache bash \
        bash-doc \
        bash-completion \
        && rm -rf /var/cache/apk/* \
        && /bin/bash


RUN mkdir -p /home/node/app/node_modules

WORKDIR /home/node/app

COPY package*.json ./

COPY . .


RUN npm config set registry https://registry.npm.taobao.org/

RUN npm install

RUN npm rum build

EXPOSE 8080

CMD ["node", "dist/bin/server.js"]

```

`Dockfile`的使用 可以参考 https://www.runoob.com/docker/docker-dockerfile.html



运行`docker docker build -t flyingzl/mrtc-server:v2 .`可以 生成一个本地镜像

![image-20201210172604018](https://g.asyncoder.com/images/20201210172605-image-20201210172604018.png)

![image-20201210172805229](https://g.asyncoder.com/images/20201210172806-image-20201210172805229.png)



### 通过API和 SDK来获取Docker信息

Docker提供了[API](https://docs.docker.com/engine/api/)来访问访问 ，可以通过 REST API或者第三方SDK来访问



```bash
~ curl --unix-socket /var/run/docker.sock http://v1.40/containers/json | jq '[.[] | {name: .Names, image: .Image, status: .Status}]'

[
  {
    "name": [
      "/node-vim-demo"
    ],
    "image": "node-vim",
    "status": "Up 41 minutes"
  },
  {
    "name": [
      "/portainer"
    ],
    "image": "portainer/portainer-ce",
    "status": "Up 17 hours"
  }
]
```



或者在Node.js上 使用 [dockerode](https://github.com/apocas/dockerode)来进行访问

```javascript
import Docker from 'dockerode'

const docker = new Docker()
// 获取Docker版本信息
docker.version().then(v => {
    console.log(`docker version: ${v.Version}`)
})

// 获取当前正在运行的容器
docker.listContainers().then(containers => {
    containers.forEach(container =>  {
        console.log(`${container.Names}\t${container.Image}\t${container.Status}`)
    })
})
```



##  参考资料

[谁说前端不用懂，手摸手 Docker 从入门到实践](https://mp.weixin.qq.com/s?__biz=MzAxODE2MjM1MA==&mid=2651562919&idx=1&sn=e104ff4e6d1f4ea88f419244bb9da608&chksm=80257466b752fd70be127c910e1d7ef8e36b573e9e4a96f3371d4f890c6e49b9e53699b9c59c&mpshare=1&scene=1&srcid=1210YXwNWCaNxeGwtYT0NEhX&sharer_sharetime=1607592691939&sharer_shareid=281b99e11b10b50250f9d04de224f74d#rd)

[小白学习docker只要这篇文章就够了](https://juejin.cn/post/6844904117165359111)

[Docker 教程](https://www.runoob.com/docker/docker-tutorial.html)

[docker容器与虚拟机有什么区别？](https://www.zhihu.com/question/48174633)

[Docker镜像导入导出](https://www.hangge.com/blog/cache/detail_2411.html)

[Dockfile Reference](https://docs.docker.com/engine/reference/builder/)

[Docker运行Unbuntu桌面](https://yeqing.run/202005/docker-vnc-desktop.html)




