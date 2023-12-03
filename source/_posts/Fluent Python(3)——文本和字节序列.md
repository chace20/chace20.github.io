---
title: Fluent Python(3)——文本和字节序列
date: 2017-09-25 16:38
tags: Python
---

## 1.处理文本文件
**unicode三明治**：bytes->str->bytes，中间的文本处理只涉及到str。

**chardet**：检测文本编码的模块。

不要依赖系统的默认编码，一定要设置编码。

```
# 使用文本方式打开文本文件
open('a.txt', 'r', encoding='utf-8')
# 使用二进制模式打开二进制文件
open('a.gif', 'rb')
```

## 2.规范化Unicode字符串
```
from unicodedata import normalize

def nfc_equal(str1, str2):
    return normalize('NFC', str1)==normalize('NFC', str2)

def fold_equal(str1, str2):
    return (normalize('NFC', str1).casefold() ==
        normalize('NFC', str2).casefold())
```

## 3.正则表达式对str和bytes的匹配

```
# 按str形式进行匹配，可以匹配到中文
re.compile(r'\w')
# 按bytes形式进行匹配，只匹配ascii
re.compile(rb'\w')
```
