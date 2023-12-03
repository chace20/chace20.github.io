---
title: Fluent Python(17-18)——并发
date: 2019-03-11 20:30:04
tag: Python
---

# 17. concurrent处理多进程和多线程

## 两种方式
1. 线程池和进程池：ThreadPoolExecutor和ProcessPoolExecutor
2. ThreadPoolExecutor.map(func, list_of_param)返回生成器，获取各个函数返回的值。获取返回值时会**阻塞**，返回结果的顺序与调用的顺序一致。
3. ThreadPoolExecutor.submit(func, param)返回future对象，futures.as_completed(list_of_future)返回迭代器，在任一future运行结束后产出future。之后可以使用future.result()获取结果，这样可以**不阻塞**。
4. 另一种方式是future.add_done_callback(callback)

as_completed()内部实现
``` python
        # 将调用as_completed之前已经完成的future直接yield
        yield from finished
        # 死循环
        while pending:
            if timeout is None:
                wait_timeout = None
            else:
                wait_timeout = end_time - time.time()
                if wait_timeout < 0:
                    raise TimeoutError(
                            '%d (of %d) futures unfinished' % (
                            len(pending), len(fs)))
            # 阻塞等待有没有future完成
            waiter.event.wait(wait_timeout)

            with waiter.lock:
                finished = waiter.finished_futures
                waiter.finished_futures = []
                waiter.event.clear()
            # 每当有完成的future时，yield future
            for future in finished:
                yield future
                pending.remove(future)
```

## 性能相关
1. CPython解释器本身不是线程安全的，因此有GIL，一次只允许一个线程执行Python字节码。
2. 标准库中阻塞性IO操作在等待系统返回结果时会释放GIL，因此IO密集型操作可以用多线程并发；time.sleep()也会释放GIL实现并发。
3. CPU密集型操作使用多进程并发。
3. PyPy虽然没有释放GIL，但是因为有JIL，在CPU密集型工作时比CPython快。

## Tips
注意：with块结束后主程序才会继续往下执行，或者显式地调用executor.shutdown(wait=False)不等待继续执行。

shutdown(wait=True)的含义：

Signal the executor that it should free any resources that it is using when the currently pending futures are done executing.

所以调用shutdown并不会中断当前进程池中的future，只是通知future执行完毕后释放资源。


# 18. asyncio协程处理并发

## yield from/await的概念
在调用方-委派生成器-子生成器模型中：
在gen中使用yield from subgen()时，subgen获得控制权，把产出的值传给gen的调用方。gen会阻塞，等待subgen()产出值。
yield from的主要功能是打开双向通道，把最外层的调用方与最内层的子生成器连接起来，这样二者可以直接发送(send)和产出(yield)值。（在Python 3.5以上，可以使用await代替yield from，更容易理解）
假设yield from出现在委派生成器中，客户端代码驱动着委派生成器，而委派生成器驱动着子生成器。

asyncio中：
可以这样理解，yield from跟普通调用的区别就在于不会阻塞事件循环。类似于epoll。
``` python
# 不会阻塞事件循环，因为当调用了yield from以后，控制权就回到事件循环手中了。
resp = yield from aiohttp.request('GET', url)
# resp = await aiohttp.request('GET', url) 用await更容易理解
# 会阻塞主线程
resp = aiohttp.request('GET', url)
```

yield from的用法：
1. 在yield from链接的多个协程最终必须由不是协程的调用方驱动，调用方显示或隐式在最外层委派生成器上调用next(...)或.send(...)。
2. 链条中最内层的子生成器必须是简单生成器(yield)或可迭代对象。

在asyncio中：
我们编写的协程链条始终通过把最外层委派生成器传给asyncio包API中的某个函数(如loop.run_until_complete(...))驱动。也就是说，调用next(...)或者send(...)的操作由asyncio的事件循环完成。

概括起来：使用asyncio包时，使用yield from架起管道，让asyncio的事件循环(通过我们编写的协程)，驱动执行底层异步IO操作的库函数。


## 使用asyncio的步骤

### 1. 创建task(可选)

创建单个task：

 - asyncio.async(...)
 - BaseEventLoop.create_task(...)

创建多个task:
 - asyncio.wait(coros)  全部协程执行完毕后返回结果
 - asyncio.as_completed(coros)  返回一个生成器，当有协程完成时就迭代

### 2. 获取事件循环

loop = asyncio.get_event_loop()

### 3. 将task(s)或者coro加入到事件循环中

 - loop.run_until_complete(coro/task)   普通运行loop
 - loop.run_in_executor(coro/task)   在ThreadPoolExecutor中运行loop

**需要注意的是，第1步不是必须的，如果在第3步中传入的是协程而不是task，那么run_until_complete()会将协程包装成task，之所以用第1步，是为了持有task对象，方便对协程进行操作，比如获取完成的task的result、task.cancel()等。**

至于协程本身的定义，内部使用yield from关键字，函数使用@asyncio.coroutine装饰。

**对于Python 3.5以上，内部使用await代替yield from，函数使用async def func()**，语义更加明确。

### 异步IO的事件循环示例：
``` python
import threading
import asyncio

async def hello():
    print('Hello world! (%s)' % threading.currentThread())
    await asyncio.sleep(1)
    print('Hello again! (%s)' % threading.currentThread())

loop = asyncio.get_event_loop()
tasks = [hello(), hello()]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```
执行结果:
```
Hello world! (<_MainThread(MainThread, started 140735195337472)>)
Hello world! (<_MainThread(MainThread, started 140735195337472)>)
(暂停约1秒)
Hello again! (<_MainThread(MainThread, started 140735195337472)>)
Hello again! (<_MainThread(MainThread, started 140735195337472)>)
```
可以看到，两个协程并发执行，但是是在同一个线程里完成的，中间并没有发生阻塞。也就是说，当协程运行到await asyncio.sleep(1)时，控制权会交换给事件循环，事件循环会去继续执行其他协程，中间不会阻塞。

### 基于异步IO的HTTP服务器
``` python
import sys
import asyncio
from aiohttp import web

async def index(request):
    print('Receive: {}'.format(request))
    await asyncio.sleep(0.5)
    return web.Response(content_type='text/html', text='<h1>Index</h1>')

async def hello(request):
    print('Receive: {}'.format(request))
    await asyncio.sleep(0.5)
    text = '<h1>hello, %s!</h1>' % request.match_info['name']
    return web.Response(content_type='text/html', text=text)

async def init(loop, address, port):
    app = web.Application(loop=loop)
    app.router.add_route('GET', '/', index)
    app.router.add_route('GET', '/hello/{name}', hello)
    srv = await loop.create_server(app.make_handler(), address, port)
    print('Server started at http://{}:{}...'.format(address, port))
    return srv

def main(address='127.0.0.1', port=8888):
    loop = asyncio.get_event_loop()
    # 注意观察这里的srv，其实是协程init结束后的返回值
    srv = loop.run_until_complete(init(loop, address, port))
    try:
        loop.run_forever()
    except KeyboardInterrupt:
        pass
    print('Server {} shutting down.'.format(srv))
    loop.close()

if __name__ == '__main__':
    main(*sys.argv[1:])
```

# Python并发总结
协程的底层还是基于事件循环，类似于IO多路复用这种方式，让单线程可以实现并发。

并发的实现依赖于协程的不阻塞，所以协程最后的操作要使用非阻塞操作(比如asyncio.sleep(0.5)是非阻塞的，或者一些asyncio的网络IO操作)才能发挥作用，否则就跟顺序执行一样了。如果必须要使用阻塞的操作，可以使用`loop.run_in_executor`在线程池中使用多个loop。

asyncio实现并发的流程：主线程持有一个事件循环loop，会去调度加入到事件循环中的协程并调度执行，当协程进行异步操作的时候，控制权回到事件循环，事件循环再去调度其他协程执行。当暂停的协程返回时，事件循环再去调度其执行。因而实现了并发，跟IO多路复用很像。

协程跟多线程相比优势：
1. 没有切换线程的开销
2. 不用处理锁
3. 由用户决定协程的调度（通过send()激活，yield暂停）

协程跟多线程相比劣势：
1. 执行的操作必须是异步操作，否则就没有调度的意义
2. 无论怎么说，只有一个线程，没法使用到多核CPU。(Go语言好像解决了这个问题)

总结来说，并发的实现可以基于三种方式：
1. 多进程
2. 多线程
3. 基于事件循环的异步IO

而对于3来说，目前主流语言有两种实现方式：1.回调 2.协程。NodeJS里使用的是回调，而Python使用的是协程。

与回调相比，协程的优势在于：
1. 不会陷入多层回调嵌套，那样代码是复杂且难以阅读的。
2. 协程可以在中断处保存上下文，下次继续执行时可以恢复，因而不需要再去单独把中间结果保存到全局或者其他地方（回调中必须这样做）。

## 使用场景总结
计算密集型使用多进程。这样可以用到多核CPU。

异步IO密集型可以使用协程，然后用线程池创建多个event_loop可以辅助提高性能；

同步IO密集型使用多线程。Python因为有GIL锁，其实多线程并不能利用多核CPU实现**并行**。
