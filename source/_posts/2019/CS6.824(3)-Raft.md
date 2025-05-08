---
title: CS6.824(3)-Raft
date: 2019-08-31 20:41:50
tags: 分布式系统
---

Raft作为两大分布式一致性协议之一（另一个就是大名鼎鼎的Paxos），本身是为了解决Paxos学习成本过高，工程实现过于困难的问题。在论文中也是遵循这个原则，因此阅读下来还是比较轻松的，而且也有丰富的图表可以帮助思考。

不过在实现上还是会有很多坑，再次感受到了分布式编程的困难之处，尤其是调试，也再次印证了printf是最好的调试工具~（事实上只能看日志来调试）

下面简单过一下Raft比较有意思的点。

## 选举安全
在同一term最多选举出一个leader。

首先所有节点都是follower，term=0，term相当于一个阶段。

follower等待一个随机的超时时间，超时以后该节点成为candidate，term+1，向其余节点发起投票请求。

其余节点发现投票请求中的term>=currentTerm，并且在currentTerm没有投过票，并且投票请求的日志记录至少与自己的日志记录一样新，就授权这次投票请求。

当某个candidate获得majority的授权以后，成为leader，并且向其余follower发送心跳。

如果某个term没有leader产生，那么在选举超时以后，会再发起选举。

## Leader的日志完整性
只要在某个term一条日志被commit，那么这条日志将会出现在所有更高term的leader中。

假设存在一个比term T大且最小的term U，term U的leader不包含term T已经commit的日志。

因为要成为leader，必须获得majority follower的投票；同样地，一个日志要被commit，也必须被majority follower接收。所以至少存在一个follower，既接收了term T的leader的日志，也投票给了term U的leader。

如果term U的leader不包含之前commit的日志，那么这个follower是不会投票给他的。因此会矛盾。

论文里面是用反证法证明的。
