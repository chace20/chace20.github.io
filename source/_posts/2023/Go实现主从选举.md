---
title: Go实现主从选举
date: 2023-10-08 12:06:46
tags: 
 - k8s
 - go语言
---

## 背景

最近项目上需要重构一个模块，因为这个模块设计到一些全局信息的维护，多副本会有竞争问题，希望采用单副本的方式来运行。但是单副本的话又会面临高可用的问题。因此要解决的问题就是**如何做主从选举来保证既是单副本工作，又可以保证高可用。**

## 思路

本来想手撸的话可以依然用Redis的分布式锁或者Etcd的分布式锁来实现。

1. 在服务启动的时候去拿锁，拿不到锁就放弃，然后过一段时间再去拿锁，直到拿到锁成为leader
2. 成为leader后给这个锁加上过期时间，并且周期性去续约

但是自己手撸就需要自己写很多代码了，边界条件也比较多，比较麻烦。

刚好发现k8s的client-go中已经有实现了，直接用就好了，不需要重复造轮子，还是很爽的。正好我们的服务也是部署在k8s上的，这样就相当“云原生”了。

没错，就是这个轮子：https://pkg.go.dev/k8s.io/client-go/tools/leaderelection

## 代码实现

1. 创建资源锁
2. 开启选举循环，注册回调
3. 在onStartedLeading回调中继续业务逻辑即可

```go
// 创建资源锁
	lock := &resourcelock.LeaseLock{
		LeaseMeta: metav1.ObjectMeta{
			Name:      leaseLockName,
			Namespace: leaseLockNamespace,
		},
		Client: k8sclient.CoordinationV1(),
		LockConfig: resourcelock.ResourceLockConfig{
			Identity: id,
		},
	}

	// 开启选举循环。如果没有成为leader，会阻塞在这里
	leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
		Lock: lock,
		ReleaseOnCancel: true,
		LeaseDuration:   60 * time.Second,
		RenewDeadline:   15 * time.Second,
		RetryPeriod:     5 * time.Second,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				// 成为leader后的回调，一般继续业务逻辑
				run(ctx)
			},
			OnStoppedLeading: func() {
				// 失去leader后的回调
				klog.Infof("leader lost: %s", id)
				os.Exit(0)
			},
			OnNewLeader: func(identity string) {
				// 发现有leader选举成功的回调
				if identity == id {
					// 自己获得了leader
					return
				}
				klog.Infof("new leader elected: %s", identity)
			},
		},
	})
}
```

最后需要注意服务的权限，需要创建ClusterRole、ServiceAccount和Binding来授权，否则可能会报错。

### 参考资料

- **[kubernetes源码-控制器 leader 选举机制](https://isekiro.com/kubernetes%E6%BA%90%E7%A0%81-%E6%8E%A7%E5%88%B6%E5%99%A8-leader-%E9%80%89%E4%B8%BE%E6%9C%BA%E5%88%B6)**
- **[源码分析 kubernetes leaderelection 选举的实现原理](https://xiaorui.cc/archives/7331)**
