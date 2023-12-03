---
title: Linux命令行大全(4)——shell编程
date: 2017-08-11 14:35:32+00:00
layout: post
toc: true
tags:
- Linux
---

## 1. 数据类型及操作

### 变量定义
1. =赋值，但是=前后不能加空格
2. **双引号内字符会扩展，单引号内的是纯文本**

```
unset variable # 取消变量
declare -i variable # 定义为整数
# 这两个都是定义为环境变量
decalre -x variable
export variable
```

**数值：** 默认的bash程序中只支持整数运算，使用`bc`命令可用于浮点运算：
```
# 数值运算需要在(())中
echo $((1+3-2/5*3)) # 输出"4"
echo $((2**3)) # 输出"8"
bc <<< '2.5+4.9*4.7' # 输出"25.5"
```

**字符串：** 字符串的操作比较多：
``` shell
foo="Hello world"
echo ${#foo} # 统计字符数，输出"11"
echo ${foo/world/World} # 替换第一个匹配的字符串，输出"Hello World"
echo ${foo#*l} # 从前往后，删除符合的最短数据，输出"lo World"
echo ${foo##l} # 从前往后，删除符合的最长数据，输出"d"
echo ${foo%rld} # 从后往前，删除符合的最短数据，输出"Hello wo"
var=${str:-expr} # str==null或者str==""时设置为expr
var=${str:?expr} # str==null或者str==""时expr输出到stderr
```

**数组**
``` shell
declare -a a # 创建一个数值
arr=("1" "2" "abc" "4" 5) # 数组赋值
arr[0]=8 # 数组赋值

# 以下是*和@输出的区别，*是扩展成一串，而@分开了
for i in "${arr[*]}"; do echo $i; done
8 2 abc 4 5
for i in "${arr[@]}"; do echo $i; done
8
2
abc
4
5
```

## 2. 分支、循环
跟类C语言大同小异，需要注意的是条件分支的test命令

**中括号内部前后必须加一个空格**

**分支：** test命令判断是否符合条件，其中传统的test命令形式为`[ express ]`，更为现代的形式为用于字符串和普通变量的`[[ express ]]`和用于数值的`(( express ))`：
```
string=20
if [ -z string ]; then # 判断字符串是否为空
    echo "string is empty"
elif [[ $string =~ ^h.+o$ ]]; then # 匹配正则表达式
    echo "string is right"
elif (( string>10 )); then # 数值比较
    echo "string is greater than 10"
else
    echo "string is not be matched"
fi
# 输出"string is greater than 10"
```

**case分支**
```
case $i in
    "1")
    echo "input is 1"
    ;;
    "2")
    echo "input is 2"
    ;;
    *)
    echo "input is $i"
    ;;
esac
```

**循环：**
``` shell
# while型
# 当条件成立时继续循环
count=1
while [ $count -le 5 ]; do
    printf "%d " $((count++))
done

# until型
# 当条件成立时，就终止循环。也是先判断条件。
until [ $count -gt 10 ]; do
    printf "%d " $((count++))
done

# for-loop型
# 或者用for i in $(seq 1 10)
for i in {1..10}
do
    printf "%d " $i
done

# for-loop型
for (( i=1; i<=10; i++))
do
    printf "%d " $i
done

# 输出"1 2 3 4 5 6 7 8 9 10"
```

## 3. 函数
shell中的函数定义如下：

``` shell
[function] foo [()] {
    # action
    [return int]
}

# 定义其实是两种形式:
function foo {...}
foo () {...}

foo # 函数调用
```

不像类C语言在函数括号中传递参数，shell中使用**位置参数**来进行参数传递。shell中的返回值通过`$?`接收。
如下所示：

``` shell
foo () {
    echo $0 # 输出的永远是运行脚本的命令本身
    echo $($1+$2) # 输出58
    return (($1+$2+2)) # 返回60，返回值范围是0~255整数
}
foo 13 45 # 函数调用
echo $? # 输出的是最后一次调用foo的返回值60
```

一些特殊的参数如下：
``` shell
echo $* # 输出的是"$1 $2 $3"
echo $@ # 输出的是"$1""$2""$3"
echo $$ # 输出当前进程pid
echo $# # 输出命令参数个数
echo $? # 输出上个函数的返回值
echo $! # 输出后台中最后一次运行的进程pid
```

## 4. 执行与调试
```
# +x 可以显示执行的命令，加号开头的都是命令串
# +n 是检测语法
# -x和-n是不执行命令，+x和+n是同时执行命令
bash -n <script> 
# 使用source是在当前进程中执行.sh，否则是新开一个子进程执行
source <script> 
```

## 5. 实现quicksort
拿个quicksort来作为练习倒是一个不错的选择

``` shell
#!/bin/bash -x

# a quicksort algorithm implement in shell

arr=(3 23 4 1 45 56 34 78 79 89)

get_pos() {
    local low=$1
    local high=$2
    local value=${arr[$low]} # 数组取值

    while (( $low<$high )); do
        while (( $low<$high && $value<=${arr[$high]} )); do
            high=$(($high-1))
        done
        if (( $low<$high )); then
            arr[$low]=${arr[$high]}
            ((++low)) # 整数运算只能在(())中
        fi
        while (( $low<$high && $value>=${arr[$low]} )); do
            low=$(($low+1))
        done
        if (( $low<$high )); then
            arr[$high]=${arr[$low]}
            ((--high)) # 整数运算只能在(())中
        fi
    done

    arr[$low]=$value
    return $low
}

quicksort() {
    local low=$1
    local high=$2
    local pos=0
    local tempLow=0
    local tempHigh=0
    if (( $low<$high )); then
        get_pos $low $high
        pos=$? # 获得返回值
        tempLow=$(($pos-1))
        tempHigh=$(($pos+1))
        quicksort $low $tempLow
        quicksort $tempHigh $high
    fi
}

quicksort 0 9 # 调用函数

echo ${arr[@]} # 另一种形式是${arr[*]}，两者区别在前者扩展数组单个元素，后者将数组扩展成一个长串

```
