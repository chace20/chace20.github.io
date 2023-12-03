---
title: Java多线程编程实战指南(8)-线程管理
date: 2019-10-11 21:23:17
tags: Java
---

# 线程管理

ThreadGroup是个隐含的概念，通常情况下不用管。

uncaughtExceptionHandler可以用来处理线程的异常退出状态，比如实现线程退出重启。uncaughtExceptionHandler实例选择的优先级：本线程设置的handler > 线程池handler > 默认的handler。

ThreadFactory用工厂模式来生产一组统一格式的线程，比如统一的线程名前缀、统一的uncaughtExceptionHandler。

可以用condition的wait、signalAll实现已经被废弃的Thread.suspend()和Thread.resume()方法。

线程池可以减少线程创建销毁的次数从而减少开销，同时更有效利用CPU资源。向线程池中提交Callable任务返回Future对象，future对象可以获取任务的执行结果、取消任务、判断任务是否执行。future对象还可以捕获线程池中线程的uncaughtException。**如果线程池不再使用，记得调用shutdown/shutdownNow**。executor对象有一些获取线程池状态的方法，据此可以监控线程池，也有beforeExecute和afterExecute回调可以在任务执行前后进行一些操作。对于线程池中线程的uncaughtExceptionHandler，只对通过executor.execute()方法提交的任务有效。**同一线程池中不要提交有相互依赖关系的任务，会导致死锁。**

# 线程池工作流程

```java
public ThreadExecutor(int corePoolSize, int maxPoolSize, BlockingQueue<Runnable> bq, long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory, RejectedExecutionHandler handler);
```

线程池在启动后，根据参数的prestartAllCoreThreads或者prestartCoreThread启动corePoolSize的线程或者启动一个线程。如果这两者都没有设定，那么就会新来一个task就创建一个线程，线程是由参数里的ThreadFactory创建的。

当线程数量达到corePoolSize以后，新的task将会加入到参数中的BlockingQueue中，线程池中任务在执行完当前任务后，会尝试去BlockingQueue中取新的task(空闲线程回收也是在这一步完成的，具体见下面)。

当BlockingQueue满了以后，线程池会继续创建新的线程直到maxPoolSize。如果线程数量达到maxPoolSize并且BlockingQueue也满了，线程池会执行拒绝策略，具体的拒绝策略依然由用户传入的参数决定，默认是直接丢弃。

# 线程池的空闲线程回收的细节

空闲线程回收实际上在worker去getTask的时候实现的。如果是allowCoreThreadTimeOut或者当前线程数大于corePoolSize的情况下，会去尝试调用workQueue.poll(keepAliveTime)，否则调用workQueue.take()。poll在队列为空时会等待超时返回null，take会无限等待。poll或者take返回null时，会回收当前线程，这样就达到了等待keepAliveTime回收空闲线程的效果(也就是队列中没有新的task)。

所以，**只有在allowCoreThreadTimeOut或者当前线程数大于corePoolSize的情况下，才会回收空闲线程**。也就是说，只有在设置allowCoreThreadTimeOut=true的情况下，才会回收corePoolSize的线程。

# 参考资料

[关于初始化线程和核心线程是否会被回收](https://objcoding.com/2019/04/14/threadpool-some-settings/)