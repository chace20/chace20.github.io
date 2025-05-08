---
title: CS6.824(2)-GFS
date: 2019-08-31 20:37:48
tags: 分布式系统
---

## GFS的架构
GFS模型的master，chunkserver。master保存文件的元数据，比如file目录结构，文件大小，文件包含的chunk以及chunk在什么位置。chunkserver上保存的是具体的文件数据。

## GFS读文件过程
client访问master，获取到文件的chunk所在的chunkserver，然后client直接去这些chunkserver上请求数据。控制流跟数据流是解耦的。

## GFS写文件过程
1. client访问master，获取到应该写入chunk所在的chunkserver。（超出一个chunk的话，会把写请求分成多次）
2. Master会给chunk授权一个租约，增加chunk的版本号，然后让chunkserver也同样地增加版本号。并且会在chunkserver中给本次写请求指定一个primary节点，由他来负责协调本次的写入。然后回应client，后面client跟master不需要再通信了，这样可以减少master的负载。
3. client把数据push到master告诉它的chunkserver上。
4. 一旦数据push完毕，client发送写请求给primary。primary决定本次写入的顺序并且应用到chunk上。
5. primary主节点完成修改后，将这个顺序传递给从节点secondaries，因此他们也能应用同样的修改顺序。
6. 从节点完成修改后，回应主节点。
7. 主节点之后回应client成功或者失败状态：
    - 主节点和从节点都成功，则本次写入成功；
    - 如果有失败，那么client会重试写入

## GFS的一致性模型
主要针对并发写文件的过程。在GFS里，有两种写文件的方式，一种是随机写，一种是append追加数据。这里面的一致性有两个维度，确定与一致。一致是指多个副本的内容相同，确定就是与串行写入时的内容相同。

append会有一个填充的操作，就是发现原来的chunk+新的数据可能超出一个chunk的大小，就先把chunk填满，再通知client在下一块chunk进行append。这里一次append的数据限制为chunk的1/4大小，这样就不会填充太多的无效数据。这样做的原因是前面GFS的写入操作是针对一个chunk的，就是为了避免一次写两块chunk，这样一致性会被破坏掉。

随机写是一致的但是不确定的，append填充的部分是不一致的，但是其他正常的数据是确定的。

并发写同一个区域，可能会出现覆盖的情况，导致undefined，但是几个副本的内容是一致的。
Record append填充的部分是不一致的（因此也就不一致），但是其他正常的数据是确定的。
