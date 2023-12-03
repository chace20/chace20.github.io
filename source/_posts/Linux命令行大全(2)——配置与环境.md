---
title: Linux命令行大全(2)——配置与环境
date: 2017-08-11 14:32:35+00:00
layout: post
toc: true
tags:
- Linux
---

## 1.环境
``` shell
printenv | less # 查看本机的各种环境变量
alias ll=“ls -laF” # 为命令起别名
```

## 2.vi的使用
这本书里只是简单地讲了一遍，这篇文章讲得更多一些：[Learn Vim Progressively](http://yannesposito.com/Scratch/en/blog/Learn-Vim-Progressively/)。另外就是阅读`:help usr_02.txt`

拓展阅读：[Vim命令](https://vim.rtorr.com/)

移动光标、删除、复制粘贴对比来记忆就很容易了，都一样。

**移动光标**

| 键  | 动作 |
| :------------- | :------------- |
| 数字0 | 至本行开头 |
| $ | 至本行末尾   |
| G | 至文件末尾 |
| gg | 至文件开头 |
| ctrl+F | 下翻一页 |
| ctrl+B | 上翻一页 |
| number+G | 至第number行 |

**删除文本**

| 键 | 动作 |
| :------------- | :------------- |
| dd | 删除一行 |
| d$ | 删除光标到行末尾 |
| d0 | 删除光标到行开头 |
| d7 | 删除光标后的7行 |

**复制粘贴**

| 键 | 动作 |
| :------------- | :------------- |
| p  | 粘贴文本 |
| yy | 复制一行 |
| y$ | 复制到行末尾 |
| y0 | 复制到行开头 |
| y7 | 复制光标后的7行 |

**搜索**

``` shell
/pattern # 按n查找下一个
:%s/pattern_old/pattern_new/g # 全局替换
```

**多文件**

``` shell
:n # 切换下一个
:N # 切换上一个
:buffers # 正在编辑的文件
:buffer 2 # 切换到第2个文件

:e <filename> # 打开一个新的文件
:r <filename> # 将filename的内容整个复制到当前文件中，光标前

:w <filename> # 另存为新文件
ZZ # 相当于:wq，智障操作
```

## 3.提示符
环境变量```PS1```中保存的是提示符信息，喜欢可以自己随便改。
