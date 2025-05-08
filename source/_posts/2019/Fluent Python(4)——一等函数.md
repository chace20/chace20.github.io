---
title: Fluent Python(4)——一等函数
date: 2017-09-28 20:48
tags: Python
---

可调用对象：在自定义的类Cls中实现__call__()方法即可直接使用cls()调用。

获取关于函数参数的信息：sig=inspect.signature(func)可以查看函数参数的一些信息，还可以使用sig.bind(**args)绑定参数。

函数可以添加注解，虽然并不会被用到，但是可以增加函数的可读性。
```
def func(parm:str, len: 'int > 0'=80) -> str:
    return parm
```

### operator模块的函数支持函数式编程
itemgetter 获取元素
attrgetter 获取对象的属性
methodcaller('replace',' ', '_') 在对象上调用参数指定的方法
```
from operator import itemgetter
odd=itemgetter(0,2,4)
odd(range(10))
# Out[10]: (0, 2, 4)
```

### functools.partial可以绑定一些参数。
以下就是定义了一个nfc函数，不用每次都输入NFC这个参数了。
```
import unicodedata, functools
nfc = functools.partial(unicodedata.normalize, 'NFC')
s='你好'
nfc(s)
# Out[15]: '你好'
```
