---
title: k8s核心组件分析-scheduler
date: 2019-09-02 15:36:10
tags: kubernetes
---
k8s中的scheduler负责资源调度，具体来说，是将某个pod绑定到某个node上。scheduler的输入是待调度的pod和可调度的node，输出是pod和node的bind对象。

![scheduler的输入输出](../img/2019/09/k8s-scheduler.png)


## k8s采集数据

要进行调度首先得知道哪些需要调度，哪些node可以调度，调度策略需要很多信息来进行决策。k8s中并不会有消息队列来进行组件之间的通信，scheduler是直接访问APIServer来获取集群信息的。但是如果是暴力轮询肯定会给APIServer带来很大负担。所以scheduler在本地做了cache。

本地cache主要有两种，一种是对于有序资源信息，比如待调度的pod，使用Queue来保存；另一种是无序资源信息，比如可调度的node，使用链表保存。

## k8s的调度算法
k8s的调度算法是可插拔的，由算法provider提供，一个provider即提供一个调度策略集合，而这个调度策略集合下会包括很多的计算方法。

这些具体的计算方法包括两类：predicates和priorities。

predicates解决pod能不能被调度到某个node上去，而priorities是在可调度的node中排优先级。

### predicates
有判断会不会HostPort冲突的，也有判断NodeSelector的。所有predicates都满足，则该pod和该node是可以绑定的。

### priorities
这里是采用计算score的方式。每个priority算法返回一个0~10的score，然后计算`sum = priorities · weights`(这里是点乘的意思)。调度pod到得分最高的pod上去。

一般就根据资源分配情况，selector的分配情况来计算。

## k8s的启动与运行
1. 收集scheduler产生的事件信息，发送给APIServer
2. 创建一个http server，端口默认是10250
3. 根据配置创建启动scheduler
4. 注册metrics规则，检测scheduler的性能

k8s在完成绑定以后，其实是创建了一个Binding对象，发送给APIServer。（最后由Controller Manager通知绑定的node的kubelet创建绑定的pod）

## multi-scheduler的问题
如果需要多个调度器，目前的解决办法是使用annotations的方式给pod指定由某个scheduler调度。

