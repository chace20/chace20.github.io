---
title: Linux命令行大全(1)——shell入门
date: 2017-08-11 14:30:32+00:00
layout: post
toc: true
tags:
- Linux
---

虽然一直在用Linux，也折腾过很多命令，但是因为缺乏系统的学习，所以常用到的命令也就那么些，对于Linux更多的命令，自己是缺乏了解的。

趁着暑假的时间，自己学完了这本《Linux命令行大全》，总体而言这本书讲的还是比较浅，缺乏深入，但是我本身就只是抱着学习命令的态度，所以一路看下来，倒也不至于失望。

如果只是想学习Linux命令入门，这本书挺好的；但是如果想深入学习Linux的使用，甚至内核开发，那还是另请高明吧！

以下四篇，是我在看这本书的过程中记录的笔记，有所整理，尽量涵盖主要命令，同时去除掉书中一些冗余的地方。

补充：[Bash快速参考表](https://github.com/LeCoupa/awesome-cheatsheets/blob/master/languages/bash.sh)

## 1. 导航类
``` shell
cd - # 返回上一次位置
ls -la # 显示隐藏文件和长格式
ls -d # 显示文件夹本身的详细信息，而不是文件夹内的文件信息
/bin # 系统可执行文件
/etc # 配置文件
/usr # 普通用户使用的所有文件和程序
/usr/include # C语言系统头文件
/usr/bin # 用户大部分可执行文件
```

## 2. 操作文件与目录

`-r` 一般用于文件夹目录树的递归操作，复制、删除等

``` shell
mkdir    cp    mv
```

``` shell
rm -r <dir>
```

``` shell
# 建立硬链接和软链接
ln <file> <file_hard>
ln -s <file> <file_soft>
```

## 3. 查看命令的属性
```
type    which    whatis
```
最重要的命令：man、info、help
``` shell
man -k <search_string> # 查找哪些命令可用
```

## 4. 重定向

重定向符和管道符的区别：
```
重定向符将stdout重定向到>后接的文件中；
管道A|B将程序A的标准输出重定向到程序B的标准输入中。
```

重定向：
```
>  标准输出重定向
2>  标准错误重定向
&> 标准输出+标准错误重定向
>&2 标准输出重定向到标准错误中 （可用于shell脚本输出错误信息）
nohup <program> &> xxx.out &  # 在后台执行命令，并将标准输出+标准错误重新向到xxx.out中
```

``` shell
wc -l # 统计行数  
wc -w # 统计字数
head/tail -n <num>
tee # 读取标准输入，同时输出到标准输出和文件中，相当于可以在中途截取掉信息。
```

## 5. 快捷键
| 键 | 动作 | 键 | 动作
| :------------- | :------------- | :--------- | :--------|
| Ctrl+A | 移动到行首  | Ctrl+E | 移动到行尾 |
| Alt+F | 往前一个字  |  Alt+B | 往后一个字 |
| Ctrl+Y | 粘贴 | Ctrl+L | 清屏 |
| Ctrl+K | 向后剪切到行尾 | Ctrl+U | 向前剪切到行头 |

## 6. 权限
``` shell
su    passwd
sudo gpasswd -a $USER <group> # 添加当前用户到指定组
chmod xxx <file>
chown [ower][:[group]] <file>
```

## 7. 进程
``` shell
ps    top    pstree
kill -9 <pid/jobspec> # 杀死进程
kill -l # 查看所有信号格式
killall -9 <name> # 杀死指定程序的所有进程
```

前后台进程切换：

![前后台进程切换](process-switch.png)
