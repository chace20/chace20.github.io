---
title: Python生成随机字符串
date: 2017-10-07 12:31:23
layout: post
tags: Python
---

写这篇文章的起因是在看微信JS接口demo的Python版本时，看到了一种生成随机字符串的方式，于是在V2EX上写了一个讨论：[Python生成一段随机字符串的两种写法](https://www.v2ex.com/t/394944#reply16)。这里是对那个讨论的小结。

```
# 方法1
s1 = ''.join(random.choice(string.ascii_letters + string.digits) for _ in range(10))

# 方法2
s2 = ''
for _ in range(10):
    s2 += random.choice(string.ascii_letters + string.digits)

# 方法3
s3 = ''.join(random.choices(string.ascii_letters + string.digits, k=10))

# 方法4
s4 = binascii.hexlify(os.urandom(5)).decode()

# 方法5
s5 = secrets.token_urlsafe(10)
```

下面是对这几种方法的对比：

**速度上来讲，方法4是最快的**

random.choice()和random.choices()底层都是C语言实现。但是因为choice每次只是生成一个随机字符，如果要生成长字符串，需要反复调用Python的函数，导致速度很慢，而choices是一次生成一个k长度的随机字符串，Python代码调用的少，所以速度要快。详情参考附录2。

然而os.random()内部实现是直接调用的syscall(such as /dev/urandom on Unix or CryptGenRandom on Windows)，所以速度最快。详情参考附录3。

**方便程度来讲，方法5是最好的**

secrets模块从Python 3.6引入，目的是为了实现生成用于加密的随机字符串。

对于secrets模块和原理的random模块，官方文档的说明是这样的：In particularly, secrets should be used in preference to the default pseudo-random number generator in the random module, which is designed for modelling and simulation, not security or cryptography.详情参见附录4.


## 参考文献
1. [Python生成一段随机字符串的两种写法](https://www.v2ex.com/t/394944#reply16)
2. [random模块源码](https://hg.python.org/cpython/file/tip/Lib/random.py#l252)
3. [os.urandom文档](https://docs.python.org/3/library/os.html?highlight=os%20urandom#os.urandom)
4. [secrets模块文档](https://docs.python.org/3/library/secrets.html)
