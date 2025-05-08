---
title: Fluent Python(5)——函数装饰器和闭包
date: 2017-10-13 20:04:04
tag: Python
---

这一章主要介绍的是Python中的装饰器，装饰器有两大特性：

1. 把被装饰的函数替换成其他函数；
2. **装饰器在加载模块时立即执行**

即函数装饰器在导入模块时立即执行，而被装饰的函数只在明确调用时运行。

Python不要求声明变量，但是假定在函数定义体中赋值的变量是局部变量，所以这一点必须要小心。

## 闭包
闭包指延伸了作用域的函数，其中包含函数定义体中引用、但是不在定义体中定义的非全局变量。函数能访问定义体之外定义的非全局变量。

另一个术语是自由变量(free variable)。[wiki的定义](https://en.wikipedia.org/wiki/Free_variables_and_bound_variables)是这样的：

> In computer programming, the term free variable refers to variables used in a function that are neither local variables nor parameters of that function. The term non-local variable is often a synonym in this context.

通俗来说，也就是未在本地作用域中绑定的变量。

自由变量可以隐式定义，也可以使用`nolocal`关键字显示定义，避免出现上面说的因为在函数定义体中赋值导致变量成为局部变量的情况。

## 装饰器
装饰器从本质上就是返回一个装饰后的新的函数。

标准库中的`functools.lru_cache(maxsize=128, typed=True)`实现缓存经常被调用的函数返回结果，在递归中使用地会比较多。

另外，`functools.singledispatch`可以实现泛函数。

如果是带有参数的装饰器，要设计到三层的函数：
``` python
# clock是参数化装饰器工厂函数
def clock(*deco_args):
    def decorate(func):
        def clocked(*func_args):
            result = func(*func_args)
            return result # 返回原本func运行的结果
        return clocked # 返回func被装饰后的函数clocked
    return decorate # 返回真正的装饰函数
```

这个比较复杂，有些是用class而不是function来实现的，参见拓展阅读。有时间再看一下。

## 拓展阅读
 - [Closures in Python](http://effbot.org/zone/closure.html)
 - [How you implemented your Python decorator is wrong](https://github.com/GrahamDumpleton/wrapt/blob/develop/blog/01-how-you-implemented-your-python-decorator-is-wrong.md)
