---
title: Docker中的namespace和cgroup
date: 2019-09-01 10:24:07
tags: Docker
---

## 1. namespace资源隔离

Linux提供6种namespace隔离。

|namespace|flag|备注|
| -- | -- | -- |
| UTS | CLONE_NEWUTS | 主机名和域名 |
| IPC | CLONE_NEWIPC | 进程间通信 |
| PID | CLONE_NEWPID | 进程PID |
| MOUNT | CLONE_NEWNS | 文件系统挂载点(mount) |
| NET | CLONE_NEWNET | 网络 |
| USER | CLONE_NEWUSER | 用户权限 |

tips: 文件系统挂载点之所以是NS，是因为这是第一个namespace，当时没有想到会有其他namespace，所以直接用的NS。

### namespace提供的系统调用

#### clone: 在新namespace中创建进程
传入哪些flag中就可以达到隔离哪些资源的目的，以`|`分隔，比如`CLONE_NEWUTS|CLONE_NEWIPC`就隔离了主机名和进程间通信。

```
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

#### setns: 加入一个已经存在的namespace
```
int setns(int fd, int nstype);
```

#### unshare: 将当前进程加入到新的namespace中
```
int unshare(int flags);
```

#### 一些/proc下的文件
可以修改/proc下的部分文件达到namespace隔离的效果，比如修改user namespace中的/proc/$$/uid_map和/proc/$$/proc/gid_map可以完成用户绑定的操作。

### UTS namespace
隔离主机名和域名。在clone中传入CLONE_NEWUTS，然后在子进程中修改hostname不会影响到父进程。

### IPC namespace
隔离进程间通信的文件，比如信号量、消息队列、PIPE等。

### PID namespace
隔离进程。在新的namespace下不会看到其他namespace下的进程。

在新的namespace下启动的第一个进程相当于Linux下的init进程，同时要承担init进程收养孤儿，传递SIGNAL的责任，比较重要。

这时候直接用`ps`看到的还是原来namespace的进程，需要重新挂载`proc`。

```
# mount -t <文件系统类型> <设备名> <挂载点>
mount -t proc proc /proc
```

但是这时候父子进程的文件系统并没有隔离，所以挂载到子进程后父进程也会受影响，所以在退出子进程后。需要在父进程中重新挂载`proc`

### Mount namespace
隔离文件系统挂载点。子进程会复制父进程的所有挂载点，但是之后彼此是独立的。

需要注意的是Linux有**挂载传播**的特性，也就是说挂载的时候可以将文件系统指定为**shared/slave/private/unbindable**等属性，从而可以控制不同namespace下文件系统的共享状态。

因此如果挂载点是shared的状态，上述的namespace隔离不会生效。需要通过`mount --make-private -t <文件系统类型> <设备名> <挂载点>`修改为private。

### Net namespace
隔离网络设备，在子进程中将看不到父进程中的网络设备。

为了不同namespace可以通过网络互相访问，通常的做法是创建一个veth pair，一端在容器内部，另一端接在网桥上(docker中是docker 0网桥)。通过合理分配IP，不同namespace下的veth通过网桥互相访问。

还有个细节是，在容器内的veth创建之前，外部是如何与namespace通信的？答案是PIPE。docker daemon先在宿主机上创建一个veth，然后通过PIPE通知容器内部创建veth，容器内部在veth创建之前会循环等待PIPE，完成两个veth的绑定后，移除PIPE。

到这里，可以实验一下各种namespace的隔离效果。

net.c
``` C
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE];
char* const child_args[] = {
    "/bin/bash",
    NULL
};

int child_main(void* args) {
    printf("在子进程中!\n");
    sethostname("NewNS", 12);
    mount("proc", "/proc", "proc", 0, NULL);
    execv(child_args[0], child_args);
    return 1;
}

int main() {
    printf("程序开始: \n");
    int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWUTS | SIGCHLD, NULL);
    waitpid(child_pid, NULL, 0);
    printf("已退出\n");
    return 0;
}
```

### User namespace
隔离用户和权限。不同namespace下的用户相互看不到，权限也不通。

需要注意的是，新的namespace下的用户需要绑定外部namespace下的用户才能正常显示，通过修改/proc/$$/uid_map和/proc/$$/proc/gid_map完成绑定。


## 2. cgroups资源限制
官方定义：

    cgroups是Linux内核提供的一种机制，这种机制可以根据需求把一系列系统任务及其子任务整合（或分隔）到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。

作用：资源限制，资源统计，任务控制，优先级分配

基本概念：
 - task: 进程或者线程
 - cgroup: 按某种资源控制标准划分成的任务组
 - subsystem: 控制某一种资源，比如CPU子系统，内存子系统
 - hierachy(层级): 层级由一系列cgroup排列而成，每个层级通过绑定子系统进行资源控制

### cgroups的实现
Linux中cgroup的实现形式表现为一个文件系统，所以可以通过操作文件的方式调用cgroup。

docker实现：

    在docker的实现中，docker daemon会在单独挂载了每一个子系统的cgroup目录(比如/sys/fs/cgroup/cpu)下创建一个名为docker的控制组，然后在docker控制组里面，再为每个容器创建一个以容器ID为名称的容器控制组，这个容器里所有进程都会写到该控制组tasks中，并且会在控制文件(比如cpu.cfs_quota_us)中写入预设的限制参数值。

cgroups的实现本质上是个任务挂上钩子，当任务运行的过程中涉及某种资源时，就会触发钩子上所附带的子系统进程检测，根据资源类别的不同，使用对应的技术进行资源限制和优先级分配。

---

参考资料
1. [Docker背后的内核知识：命名空间资源隔离](https://linux.cn/article-5057-5.html)
2. [Docker容器与容器云](https://book.douban.com/subject/26894736/)
