---
title: Docker学习笔记(二)-工具及底层实现
date: 2018-09-14 11:15:43
tags: Docker
---

## Docker compose
Compose 中有两个重要的概念：

 - 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
 - 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。

命令有点类似docker
``` sh
# 构建项目中的服务容器
docker-compose build [options] [SERVICE...]
# 启动一个service
docker-compose run [options] [-p PORT...] [-e KEY=VAL...] <SERVICE> [COMMAND] [ARGS...]
# 启动一个project
docker-compose up
```

然后是模板文件docker-compose.yml
``` yaml
version: "3"

services:
  webapp:
    image: examples/web
    ports:
      - "80:80"
    volumes:
      - "/data"
```

## Docker machine
Docker machine大概是个用来创建管理虚拟机的工具，当然这些虚拟机都装好了Docker Engine。通过这种方式可以方便地创建多个docker节点。

> Docker Machine is a tool that lets you install Docker Engine on virtual hosts, and manage the hosts with docker-machine commands. You can use Machine to create Docker hosts on your local Mac or Windows box, on your company network, in your data center, or on cloud providers like Azure, AWS, or Digital Ocean.

## Docker Swarm
用于集群管理的一个工具。

### 1.创建swarm集群
``` sh
docker swarm init --advertise-addr 192.168.99.100
# 用machine创建一个worker1节点
docker-machine create -d virtualbox worker1
docker-machine ssh worker1
# 加入swarm集群
docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377
docker node ls
```

### 2.部署服务
``` sh
# 创建nginx服务
docker service create --replicas 3 -p 80:80 --name nginx nginx:1.13.7-alpine
# 查看
docker service ps nginx
docker service logs nginx
```
### 3.在swarm集群中使用compose来快速部署
``` sh
docker stack deploy -c docker-compose.yml wordpress
docker stack ls
docker stack down
```

## 底层实现

### 1. 命名空间
命名空间是 Linux内核一个强大的特性。每个容器都有自己单独的命名空间，运行在其中的应用都像是在独立的操作系统中运行一样。命名空间保证了容器之间彼此互不影响。

### 2. 控制组
控制组（cgroups）是Linux内核的一个特性，主要用来对共享资源进行隔离、限制、审计等。只有能控制分配到容器的资源，才能避免当多个容器同时运行时的对系统资源的竞争。

### 3. 联合文件系统
联合文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。

### 4. 网络
Docker 创建一个容器的时候，会执行如下操作：

 - 创建一对虚拟接口，分别放到本地主机和新容器中；
 - 本地主机一端桥接到默认的 docker0 或指定网桥上，并具有一个唯一的名字，如 veth65f9；
 - 容器一端放到新容器中，并修改名字作为 eth0，这个接口只在容器的命名空间可见；
 - 从网桥可用地址段中获取一个空闲地址分配给容器的 eth0，并配置默认路由到桥接网卡 veth65f9。

参考：[Docker 网络实现](https://www.kancloud.cn/docker_practice/docker_practice/469861)
