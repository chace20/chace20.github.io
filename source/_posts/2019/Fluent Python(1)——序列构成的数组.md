---
title: Fluent Python(1)——序列构成的数组
date: 2017-09-13 17:21:18
tags: Python
---

这一章在讲Python中的序列类型，其中关于切片、元组还有+=的谜题值得一读。

## 1. 序列类型

容器序列：list,tuple,collections.deque

扁平序列：str,bytes,bytearray,memoryview,array.array

可变序列：list, bytearray,array.array,collections.deque,memoryview

不可变序列：tuple,str,bytes

## 2. 生成器表达式与列表推导

生成器表达式可以逐个地产出元素，而不是先建立一个完整的列表，然后再把这个列表传递到某个构造函数中。因此**使用生成器表达式(generator expression)相比列表推导(list comprehension)更能够节省内存**。

## 3. 更多
![](/img/序列构成的数组.jpeg)

---

## 拓展阅读

[string format语法格式](https://docs.python.org/3/library/string.html#format-string-syntax)

[string format](http://blog.csdn.net/handsomekang/article/details/9183303)

[为什么数组要从零开始编号 and 为什么是2 ≤ i < 13？](http://www.cs.utexas.edu/users/EWD/transcriptions/EWD08xx/EWD831.html)

[memoryview教程减少buffer的复制](http://eli.thegreenplace.net/2011/11/28/less-copies-in-python-with-the-buffer-protocol-and-memoryviews/)
