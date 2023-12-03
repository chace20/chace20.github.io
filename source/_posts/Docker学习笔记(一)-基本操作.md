---
title: Docker学习笔记(一)-基本操作
date: 2018-08-27 18:26:44
tags: Docker
---

## 1. 基本概念
**镜像**：Docker镜像是分层存储的。

**容器**：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

**仓库**：仓库名经常以*两段式路径*形式出现，比如 jwilder/nginx-proxy，前者往往意味着 Docker Registry 多用户环境下的用户名，后者则往往是对应的软件名。我们可以通过*<仓库名>:<标签>*的格式来指定具体是这个软件哪个版本的镜像。

## 2. 镜像

### 2.1 commit 保存当前容器为镜像
最好不要用commit来在实际应用中保存镜像。
```
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]
```

### 2.2 docker file
```
FROM 基础镜像
RUN 运行命令，最好只有一行 \
    && 否则每RUN一次都是一次commit \
    && 最后记得清理依赖 \
    && apt-get purge -y --auto-remove $buildDeps
```

#### 2.2.1 构建镜像
```
docker build [选项] <上下文路径/URL/->

Example:
docker build -t nginx:v3 -f Dockerfile .
```

#### 2.2.2 构建过程
当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会**将路径下的所有内容打包**，然后上传给 Docker 引擎。这样Docker引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。可以用`.dockerignore`文件指定忽略哪些文件，否则就是打包所有文件。

#### 2.2.3 其余命令
**COPY**
```
COPY <源路径>... <目标路径>
COPY ["<源路径1>",... "<目标路径>"]
```

**CMD**
用于指定默认的容器主进程的启动命令。比如，ubuntu 镜像默认的 CMD 是 /bin/bash，如果我们直接 docker run -it ubuntu 的话，会直接进入 bash。
```
CMD <命令>
CMD ["可执行文件", "参数1", "参数2"...]
在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。
```

**ENV**
环境变量
```
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
```

**其他**
```
# run时运行这个文件
ENTRYPOINT [] 
# 挂载主机卷
VOLUME ["<主机路径>", "<容器路径>"...]
# 工作目录
WORKDIR <工作目录路径>
```

### 2.3 多阶段构建
```
# 设置阶段构建目标
FROM golang:1.9-alpine as builder

# build指定构建目标
docker build --target builder -t username/imagename:tag .

# 从构建目标拷贝文件
COPY --from=builder /etc/nginx/nginx.conf /nginx.conf
```

## 3. 操作容器

### 3.1 run
当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

 - 检查本地是否存在指定的镜像，不存在就从公有仓库下载
利用镜像创建并启动一个容器
 - 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
 - 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
 - 从地址池配置一个 ip 地址给容器
 - 执行用户指定的应用程序
 - 执行完毕后容器被终止

```
# 运行docker容器并交互式运行/bin/bash
docker run -ti ubuntu:14.04 /bin/bash
# 进入正在运行的docker容器
docker exec -ti <container id> CMD
# 日志，-f是
docker logs -f <container id>
# 导入/导出
docker export <container id> > xxx.tar.gz
docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]
# 保留历史信息
docker load
docker save
# 删除容器
docker container rm <container id>
# 删除镜像
docker rm <镜像>
```

## 4. 操作仓库
```
docker search xxx
docker pull xxx
docker push docker.domain.com/username/ubuntu:18.04
```

### 自建仓库
```
# 在5000端口启动一个仓库
docker run -d -p 5000:5000 --restart=always --name registry registry
```

### Nexus3
```
docker run -d --name nexus3 --restart=always \
    -p 8081:8081 \
    --mount src=nexus-data,target=/nexus-data \
    sonatype/nexus3
```

## 5. 数据存储
### 5.1 数据卷
数据卷是一个可供一个或多个容器使用的特殊目录。与容器独立。
```
# 创建一个名为my-vol的数据卷
$ docker volume create my-vol
# 查看所有数据卷
$ docker volume ls
# 启动时加上--mount挂载数据卷
--mount source=my-vol,target=/webapp \
```

### 5.2 挂载主机目录
使用 --mount 标记可以指定挂载一个本地主机的目录到容器中去。
```
# 加载主机的 /src/webapp 目录到容器的 /opt/webapp目录
--mount type=bind,source=/src/webapp,target=/opt/webapp
```

### 5.3 -v和--mount的区别
两者的区别在于，-v将所有选项组合在一个字段中，--mount 则将它们分开。可以参考：[docker volume容器卷的那些事](https://deepzz.com/post/the-docker-volumes-basic.html)

## 6. 容器网络
### 6.1 原理
当 Docker 启动时，会自动在主机上创建一个docker0虚拟网桥，实际上是Linux的一个bridge，可以理解为一个软件交换机。它会在挂载到它的网口之间进行转发。

同时，Docker 随机分配一个本地未占用的私有网段（在 RFC1918 中定义）中的一个地址给 docker0接口。比如典型的172.17.42.1，掩码为255.255.0.0。此后启动的容器内的网口也会自动分配一个同一网段（172.17.0.0/16）的地址。

![docker网络](https://yeasy.gitbooks.io/docker_practice/content/advanced_network/_images/network.png)

### 6.2 docker内网
```
# 创建一个docker网络
$ docker network create -d bridge my-net
# 加入docker网络
$ docker run -it --rm --name busybox1 --network my-net busybox sh
```


## 参考文献
1. [Docker — 从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/)
上面太慢可以看这个：[Docker — 从入门到实践-看云]https://www.kancloud.cn/docker_practice/docker_practice