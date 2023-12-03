---
title: Linux命令行大全(3)——常见任务和工具
date: 2017-08-11 14:33:32+00:00
layout: post
toc: true
tags:
- Linux
---

## 1. 软件包管理
``` shell
apt-cache search pkg_name # 查找软件包
apt-cache show pgk_name # 显示软件包信息
```

``` shell
dpkg -i pkg_file # 用.deb安装软件包
dpkg --list # 列出已安装的软件包列表
dpkg --status pkg_name # 判断软件包是否已安装
dpkg --search file_name # 查看某文件由哪个软件包安装得到
```

## 2. 存储
``` shell
mount
umount /dev/hdc
fdisk # 磁盘分区
mkfs -t vfat /dev/sdb1 # 创建文件系统
dd if=/dev/sdb of=/dev/sdc # 完全复制数据块
md5sum /dev/cdrom
```

## 3. 网络
``` shell
wget scp ssh
netstat -ie # 显示网络状态 ifconfig
netstat -r # 显示路由表
sftp # 此命令尤其好使，因为sftp使用的是ssh的22端口，所以不需要服务器单独再开服务
```

## 4. 文件搜索
``` shell
find ~ -type f -name '*.bak' -delete # 查找用户目录下.bak文件并删除
find ~ -type f | wc -l # 统计用户目录下文件个数
find的两个选项：test和action
```

## 5. 文件归档和压缩
文件归档和压缩是两个概念，zip命令同时包括两种功能

```
rsync -av <dir1> <dir2> # 同步dir1和dir2
```

**压缩**

```
gzip gunzip bzip2 bunzip2
```

**归档**
``` shell
tar xzvf <file_name> -C <dir> # 解压到<dir>文件夹下
tar czvf <file_name> <dir> # 打包dir到file_name
zip -r <dir> # 压缩dir
```

**将远程系统中某目录转移到本地系统**

```
ssh remote-sys 'tar cf - <dir>' | tar xf -
```

## 6. 正则表达式
BRE POSIX基本正则表达式 `grep '...'`

ERE 扩展正则表达式 `grep -E '...'`

```
find <dir> -regex '<regex>'
```

```
? * + {} . [] # 元字符
```

## 7. 文本处理
``` shell
cut -f <字段编号> <file_name> # 切片某字段
cat -n # 显示行号
aspell # 拼写检查

# 比较两文件的不同产生一个patch，并且还原文件
diff -Naur <old_file> <new_file> > file_patch
patch < file_patch

diff -c/-u <file_1> <file_2> # 将file_1与file_2进行比较

# cut以逗号为分隔符，1-5字段的内容
cut -d "," -f "1,2-4,5" <file>
```

## 8. 格式化输出
``` shell
printf "format" arguments # 格式化输出

# 输出manual到PDF文件
zcat /usr/share/man/man1/ls.1.gz | groff -mandoc > ~/foo.ps
ps2pdf foo.ps ls.pdf

a2ps -o ~/ls.ps # ASCII->PostScript
lpstat -s # 查看打印机状态
```

## 9. 编译程序
``` shell
make # 编译程序
./configure # 分析生成环境
make install # 默认安装到/usr/local/bin
```
