---
title: Fluent Python(2)——字典和集合
date: 2017-09-13 17:30:18
tags: Python
---

## 1. 字典
只有可散列类型可用作键，可散列类型：str、bytes、数值、frozenset。

通过查找来插入新值的时候优化，使用setdefault函数：
`my_dict.setdefault(key, []).append(new_value)`

查找取值的时候优化，使用defaultdict：
`my_dict = collections.defaultdict(list)`

collections.Counter：给每个键准备一个整数计数器
继承UserDict去自定义dict类型
types.MappingProxyType：创建一个视图，只可读不可写

## 2. 集合
求两个集合相同的元素个数：`found = len(needles & haystack)`
用{...}比用构造方法set([...])速度要快

---

1. dict和set内部都是用hash表来实现快速查询，也就是空间换时间。
2. 不可在迭代dict的同时添加新值，因为可能导致散列表发生变化，某些键在迭代时被跳过。
