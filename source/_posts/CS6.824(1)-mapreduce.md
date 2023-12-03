---
title: CS6.824(1)-mapreduce
date: 2019-05-15 20:48:00
tags: 分布式系统
---

## 整体架构
master: 获取Job信息，决定mapper和reducer个数，并分割文件。同时接收来自worker的注册，根据一定策略调度task给可用的worker，并进行出错处理。

worker: 执行具体的mapper或者reducer。

map阶段: map(k1, v1) -> list(k2, v2)

reduce阶段: reduce(k2, list(v2)) -> list(k2, v3)

## Mapper
Mapper读取master分配给自己的一份小文件，调用用户定义的map函数处理文件内容，map函数会返回一个key-value列表。

对于所有的key，mapper会计算其hash值并跟reducer个数取模，这样就完成了partition的过程，这个过程主要是为了将key分散到不同的reducer中。

对于上面的每一个partition，mapper会生成一个文件，将属于其的key-value对列表写入到该文件中。为了方便，序列化为JSON格式。在reduce阶段，reducer会从多个mapper生成的文件中读取分配给自己的文件。

## Reducer
Reducer从map阶段生成的文件中读取分配给自己的文件，反序列化为key-value对列表。接下来，Reducer需要完成根据key来分组的过程。

这个过程可以用hashmap来实现，也可以用排序来实现。因为考虑到文件会很大，在内存中保存这样一个hashmap消耗很大。这里使用排序来实现。对于key-value对列表，调用排序函数根据key进行排序。排序完成后，只要发现下一个key跟上一个key不一样，就可以判断在这里是两个组的分割点，上一个分割点到这一个分割点中间的数据就是同一个key的列表。

分组完成后，调用用户定义的reduce函数处理每一个key及其value列表，reduce函数返回一个字符串，代表这个key规约的结果。

最后将key以及key规约的结果序列化成JSON格式，写入到reduce输出的文件中。

之后还会有merge的阶段，将多个reducer生成的文件合并成一个完整文件，作为MapReduce最终的输出结果。

## Scheduler
master启动一个RPC服务器，用来接收worker的注册。worker通过RPC注册到master后，master会将worker的信息发送给registerChan。Scheduler从registerChan中接收注册的worker，加入到idleWorkerChan中。这个过程是在一个go程里循环读取的。

在map阶段或者reduce阶段，对于所有的task，首先将所有task放入到taskChan中，之后循环从这个taskChan中获取一个task，然后启动一个go程，在这个go程里从idleWorkerChan中获取一个worker，把这个task通过rpc的方式交给这个worker去执行。

在这里简化了worker崩溃的判断，只要rpc调用失败就认为worker崩溃，这个时候将task放回到taskChan中。如果rpc调用成功，则将worker放入idleWorkerChan中，使得其可以重新被调度。

当所有task执行完毕后，主线程会收到一个通知，跳出循环，结束该阶段的任务。

## 容错
因为只有当map阶段的任务完成以后才会开始执行reduce任务，只有当reduce阶段任务全部完成后才会开始执行merge，因此保证了map操作跟reduce操作具有原子性。也就是说，map(reduce)的输出文件要么不可用，要么就是完整的。下一阶段的task不会读取到不一致的数据。

map跟reduce操作都是幂等的，也就是多次重复执行产生的结果一直，这也是上述在worker失败时，schedule可以将task放回到taskChan中重新执行的背后原理。
