---
title: Docker入门指南
date: 2018-03-04 22:10:42
tags: Docker
---
## 什么是 Docker

Docker 是一种轻量级的虚拟化技术，它源自 dotCloud 公司的内部项目。Docker 是一个开源项目，其在 GitHub 上的[仓库](https://github.com/moby/moby)已有四万多的 star。

## 为什么要用 Docker

传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

|   特性     |   容器    |   虚拟机   |
| :--------   | :--------  | :---------- |
| 启动       | 秒级      | 分钟级     |
| 硬盘使用   | 一般为 `MB` | 一般为 `GB`  |
| 性能       | 接近原生  | 弱于       |
| 系统支持量 | 单机支持上千个容器 | 一般几十个 |

## 安装 Docker

Docker 支持的 Windows 系统的最低版本是 Windows 10 Pro，且必须开启 Hyper-V，支持的 macOS 的最低版本是 macOS 10.10.3 Yosemite。不满足以上系统要求的可以使用 [Docker Toolbox](https://docs.docker.com/toolbox/toolbox_install_windows/)。

## Docker的基本概念

### 镜像

Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

利用 Union FS 的技术，分层存储的架构。镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。

目前在 [Docker Hub](https://hub.docker.com/) 上有大量的开源镜像，可以通过 `docker pull` 命令将这些镜像拉取到本地。

### 容器

镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为容器存储层。容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。可以通过 `docker run` 命令来启动一个容器。

## 示例

从官方仓库拉取 nodejs 的镜像，使用 `Dockerfile` 构建一个新镜像，基于此镜像新建一个容器并启动，此容器监听本机的3000端口，访问 `localhost:3000`，页面返回 `hello world`。

### 拉取镜像

在命令行输入 `docker pull node:8`，即可从官方仓库拉取到 nodejs 8.x 版本的镜像。

### 构建新镜像

在一个空目录中新建 `test.js` 文件。

```javascript
const http = require('http');

http.createServer((req, res) => res.end('hello world')).listen(3000);
```

在该目录中新建 `Dockerfile` 文件。

```dockerfile
FROM node:8
COPY . /app
WORKDIR /app
EXPOSE 3000
CMD ["node", "test.js"]
```

在该目录中执行 `docker build -t node:v1 .`，即可在本地构建一个名为 `node:v1` 的新镜像。

### 启动容器

执行 `docker run --name nv -d -p 3000:3000 node:v1` 即可以 `node:v1` 为基础新建一个名为 `nv` 的容器并启动。

此时在浏览器地址栏中输入 `localhost:3000` 即可看到页面返回了 `hello world`。