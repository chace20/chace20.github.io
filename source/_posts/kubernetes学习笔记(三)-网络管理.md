---
title: kubernetes学习笔记(三)-网络管理
date: 2018-09-11 09:34:41
tags:
 - Docker
 - kubernetes
---

## 1. 要点
IP以Pod为单位分配，一个Pod内部的所有容器共享一个网络堆栈。

一个Pod内部的应用程序看到的自己的IP地址和端口与集群内其他Pod看到的一样，它们都是Pod从docker0分配的。

一个pod内的容器可以通过localhost访问另一个容器。

Pod从网络角度来看，可以看做一台独立的“虚拟机”或“物理机”。

docker原生网络通过动态端口映射的方式实现多节点访问，访问者看到额IP地址和端口与服务提供者实际绑定的不同。服务自身很难知道自己对外暴露的真实的服务IP和端口，外部应用也无法通过服务所在容器的私有IP地址和端口来访问服务。

Docker一开始没有考虑到多主机互联的网络解决方案。

**当前docker的多主机网络解决方案？？？**

## 2. kubernetes的网络实现
Service就是一个反向代理。

1. 同一pod内容器共享网络空间，可以通过localhost访问
2. 同一node上的不同pod之间通信通过docker0网桥
3. 不同node上的不同pod之间通信需要经过宿主机转发

**不同Node上的Pod之间的通信要解决两个问题：**
1. 整个kubernetes集群中对pod的IP分配进行规划，不能有冲突；
2. 找到一种办法，将pod的IP和所在node的IP关联起来，通过这个关联让pod可以互相访问。

pod和service之间的通信通过kube-proxy的反向代理。

外部访问service通过NodePort或者LoadBalancer。NodePort模式会在集群的每个Node上打开一个主机上的真实端口号。

## 3. 使用网络组件

1. Flannel
 - 通过etcd分配不冲突的IP
 - 建立叠加网络

2. Open vSwitch
 - docker0网桥的数据会发给ovs网桥
 - 通过GRE/VxLAN隧道在Node之间传输

3. 直接路由
可以通过在两个Node之间添加静态路由规则来访问

``` sh
# 在Node1(docker0: 10.1.10.0, eth0: 192.168.1.128)上添加规则
route add -net 10.1.20.0 netmask 255.255.255.0 gw 192.168.1.129
# 在Node2(docker0: 10.1.120.1, eth0: 192.168.1.129)上添加规则
route add -net 10.1.10.0 netmask 255.255.255.0 gw 192.168.1.128
```

也可以通过Quagga、Zebra等配置动态路由规则，但是还是要提前规划好docker0的IP分配。

**DNS跟Monitor官网现在已经更新，这本书上的内容有点过时了。** 可以参考下面文章。

 - [配置Kubernetes DNS服务kube-dns](https://jimmysong.io/posts/configuring-kubernetes-kube-dns/)
 - [Tools for Monitoring Resources](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
