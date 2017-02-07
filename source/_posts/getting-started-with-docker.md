---
title: getting-started-with-docker
date: 2017-02-07 15:33:44
tags:
---


### SECTION 1
#关于Docker

一夜之间，Docker俨然成为开发，打包，部署应用的一种既成标准。为DevOps提供了便捷快速启动轻量级虚拟机环境-容器，容器包含了应用本身与之所依赖的一切，
这种轻量级的虚拟机易于测试，也方便运维人员的部署发布。

Docker让自动化的基础设施，应用隔离，保持一致性，提高资源利用都变的更加简单。

与广受欢迎的源码控制工具Git相似，Docker也有其协作性，开发者和运维人员都可以通过 [Docker Hub](https://hub.docker.com/) 分享镜像。

Docker是开源的解决方案，不仅仅可以运行在Linux上，也可以在 Windows 和 Mac 使用轻量级Linux发行版或者VirtualBox来运行。Docker也复杂的应用为提供了众多的管理与编排工具。


### SECTION 2
# Docker 架构

Docker使用client-server的架构，使用远程API在Linux容器之上来管理，创建Docker容器。Docker容器使用Docker镜像创建。容器和镜像的关系非常像 面向对象语言中  Class 和 Object的概念。

![docker architecture](https://dzone.com/storage/temp/576507-docker1.png)

- Docker Images(镜像)
创建容器的模版，包含了安装步骤和运行所必须的软件。
- Docker Container(容器)
像是小型的虚拟机，从镜像中创建而来。
- Docker Client(客户端)
命令行工具，或者一系列和Docker守护进程交互的[API](https://docs.docker.com/reference/api/docker_remote_api)
- Docker Host(主机)
运行Docker daemon的物理机或者是虚拟机，包含已缓存的镜像已经在运行的容器。
- Docker Registry(镜像仓库)
用来创建docker容器的镜像仓库，[Docker Hub](https://hub.docker.com) 是最受欢迎的Docker镜像仓库。
- Docker Machine(集群)
用来管理多个Docker主机的工具，可以运行在本地的VirtualBox中也可以在各种云平台中。


### SECTION 3

# 从零开始

### 安装 Docker

对于Mac和Windows用户来说，安装已经不能再简单了。
需要下载 Docker Toolbox 在 https://www.docker.com/toolbox。
此安装包包含Docker Client, Docker Machine, Compose (Mac only) 和 VirtualBox。

因为Docker基于Linux容器技术，所以不能够直接运行在Mac 和 Windows上，VirtualBox使用一个小型Linux内核来运行Docker服务。

于此同时，在Linux安装Docker可能就是没有这么容易，为了安装Docker在Linux上，我们必须要安装一些依赖。从 https://docs.docker.com/installation 文中我们可以查看详细，对于某些发行版可能在包管理中已经有Docker,对于其他的发行版通用的方法是：

```
curl -sSL https://get.docker.com/ | sh
```

在Linux上安装Docker-Machine需要root权限，使用以下命令：

```
curl -L https://github.com/docker/machine/releases/download/v0.4.0/docker-machine_linux-amd64 > /usr/local/bin/docker-machine
chmod +x /usr/local/bin/docker-machine
```

如果你想使用
If you want to create machines locally, you will also need to install VirtualBox using the instructions found at https://www.virtualbox.org/wiki/Linux_Downloads.

截止至此文发布，Docker-Machine仍在beta中不建议在生产环节中使用。


### 运行容器

Docker已经安装好，我们开始运行容器，如果你没有必要的镜像来创建容器，Docker会从Hub仓库中下载镜像，然后组建并运行。  
为了运行最简单的 hello-world 容器，来证明我们的环境已经安装完毕，我们执行以下命令：

```
docker run hello-world
```
最终这个命令将打印一系列消息在标准输出上。
Ultimately this command prints a message to standard output explaining all the steps that took place to display the message.


### 典型工作流

Docker对于创建镜像，推送镜像，发布镜像，运行容器有一套典型的工作流。

![Workflow](https://dzone.com/storage/temp/576516-docker2.png)

Docker构建镜像的典型方式是从Dockerfile，Dockerfile是一系列指令包含如何配置容器或者从Docker仓库中拉取镜像。当Docker 镜像在你的Docker环境中，那你就可以运行容器了，运行容器创建运行时操作系统，软件，和一些配置。比如你可以Debian操作系统上运行Mysql 5.5. 并且为你的应用创建指定的数据库，用户和表结构。这些可运行的容器能够象一个虚拟机一样启动和关闭。如果你手动的配置或者手动安装软件，容器能够通过提交的方式创建一个新的镜像再下一次使用。当然，你也可以通过Docker Hub 仓库分享你的镜像。

### 从Docker仓库拉取镜像


你可以通过访问 https://hub.docker.com 查找是否已经你所需要的镜像。仓库内有许多已经被认证的官方镜像，比如 MySQL, Node.js, Java, Nginx, WordPress 当然也有数以万记个人用户创建的镜像，如果你找到你需要的镜像，那就将之拉取下来。

```
docker pull mysql
```
如果你本地还没有缓存这些镜像，Docker会自动从Hub上下载当前最新版本并缓存之，如果你想要指定某个特殊版本，可以使用以下命令：
```
docker pull mysql:5.5.45
```
如果你并不想立即运行镜像，那这个命令将比Run节约一步，并且会在后台自动下载镜像。


### 从Dockerfile构建一个镜像


如果你没有找到任何你需要或者信任的镜像在DockerHub上，你可以通过创建Dokcerfile构建自己的镜像。Dockerfiles可以从已有的镜像中继续编写，从而添加你需要的软件或者自定义配置。

下面就是一个简单的 Dokcerfile 例子：
```
FROM mysql:5.5.45
RUN echo America/New_York | tee /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata
```

这个Dockerfile展示了如何从官方mysql仓库（指定的  5.5.45 版本）创建一个 时区为 America/New_York的新镜像。

关于 Dockerfile 详细的介绍，我们在后面再谈。

为了在目录内构建我们的镜像，我们需要执行：
```
docker build .
```
这个命令将创建一个未命名的镜像，我们可以通过下面这个命令查看我们的镜像列表：

```
docker images
```
这个命令列出我们本地所有缓存的镜像，包括我们自己创建的那个。


```
REPOSITORY  TAG     IMAGE ID      VIRTUAL SIZE
<none>      <none>  4b9b8b27fb42  214.4 MB
mysql       5.5.45  0da0b10c6fd8  213.5 MB
```
如上所示，我们build命令创建一个 REPOSITORY 和  TAG 都是 none 的镜像，这种方式并不推荐，更建议使用下面这种方式，使用 -t 为镜像打上标签：

```
docker build –t est-mysql .

REPOSITORY   TAG     IMAGE ID      VIRTUAL SIZE
est-mysql    latest  4b9b8b27fb42  214.4 MB
mysql        5.5.45  0da0b10c6fd8  213.5 MB
```
这是用还有另外一种创建镜像的方式，我们可以通过一个已经存在的镜像，通过Bash命令安装我们的自己的软件或者更改配置，最后使用Docker Commit为正在运行的容器创建一个新的镜像。不过这种方式并非是最佳实践，因为不能够重复而且文档也是一种自解释。

# 运行镜像

为了运行这个镜像，我们首先需要的在本地找到此镜像的缓存或者从Docker Hub找到。通常 Docker镜像总会需要一些额外的环境配置，我们通过 -e 的方式传入，而一切需要象守护进程需要长期运行的需要 -d 选项。 
为了运行我们 est-mysql 镜像，我们需要配置 Mysql root用户密码，我们可以从  Docker Hub 中Mysql文档里查看到具体的。

```
docker run -e +1 -d est-mysql
```
查看运行中的容器，我们使用ps命令：
```
docker ps
```
ps命令将列出所有运行的镜像，包括从什么镜像创建，哪些命令在运行，哪些端口正在被监听，还有这个容器的名字。