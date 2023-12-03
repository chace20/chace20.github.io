---
title: Go tour(1)-Basics
date: 2019-06-11 11:05:17
tags: Go
---

## 基础语法
``` go
// :=是在函数内部声明变量并赋值，在函数外部不能用
// 变量用var，常量用const，常量不能用:=
k := 3
const K int = 3
var k = 3
// 返回值可以命名，可以多个
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
// 没有初始值的变量会被赋零值
/* 基本数据类型
bool string
int int8 .... int64
uint uint8 ...... uint64
byte // uint8的别名 
rune // int32的别名
float32 float64
complex64 complex128
*/
```

## 分支循环

``` go
// if后面可以跟一句初始语句，整个条件语句作用域可用
if a:=b-32; a>0 {
} else {
}
// switch也可以这样，而且不用显示break
	switch os := runtime.GOOS; os {
	case "darwin":
		fmt.Println("OS X.")
	default:
		fmt.Printf("%s.\n", os)
	}
// switch也可以没有条件表达式
switch {
	case t.Hour() < 12:
		fmt.Println("Good morning!")
	default:
		fmt.Println("Good evening.")
	}
// 只有一种循环for，不用小括号，必须用花括号
for i:=0; i<10; i++ {
    fmt.Println(i)
}
// defer声明的语句会在外部函数返回才调用，这些语句会被压入到栈中
defer fmt.Println("world")
```

## 高级数据结构
``` go
// 指针
var p *int = 42
i := 21
pp = &i

// 结构体
type Vertext struct {
    X int
    Y int
}
v := Vertex{X:1, Y:2}
p := &v

// 数组
// 切片传参是传指针
var a [10]string
primes := [6]int{2, 3, 5, 7, 11, 13}

// 切片以后是在原数组上操作
// 切片的长度和容量：len是切片含有多少个元素，capacity是从切片第一个元素到数组最后一个元素含有的元素个数
s := []int{2,3,4,5}
s = s[:2]
len(s) // 2
cap(s) // 4
s = s[2:]
len(s) // 2
cap(s) // 2

// make创建动态数组
a := make([]int, len, cap)

// append进行切片追加的时候，如果元素个数大于了capacity，那么按什么方式将底层数组扩容？2,4,8?
func append(s []T, vs ...T) []T

// range
// v是浅拷贝还是深拷贝？浅拷贝，依然可以通过v操作原数组
func main() {
	var pow = [][]int{[]int{1, 2}, []int{3, 4}}
	for i, v := range pow {
		v[0] = 5
		fmt.Println(v)
       // 可以通过pow[i]对原数组进行操作
	}
	fmt.Printf("%v", pow)
}
/*
[5 2]
[5 4]
[[5 2] [5 4]]
*/
```

实现built-in的append函数
``` go
func appendInt(slice []int, data ...int) []int {
	m := len(slice)
	n := m+len(data)
	if n > cap(slice) {
		var newSlice []int
		if m == 0 {
			newSlice = make([]int, 2)
		} else {
			newSlice = make([]int, 2*m)
		}
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[0:n]
	copy(slice[m:n], data)
	return slice
}
```

slice练习
``` go
func Pic(dx, dy int) [][]uint8 {
	ret := make([][]uint8, dy)
	for y := range ret {
		ret[y] = make([]uint8, dx)
		for x:=0; x < dx; x++ {
			ret[y][x] = uint8(float64(x)*math.Log(float64(y)))
            // 两种方法都可以
			// ret[y] = append(ret[y], uint8(float64(x)*math.Log(float64(y))))
		}
	}
	return ret
}
```

``` go
// map初始化
m := make(map[string]Vertex)
// 操作
a = m[key]
delete(m, key) // 删除key
item, ok := m[key] // 判断key是否在m中

// 参数可以是函数，也可以有匿名函数
// 赋值再调用
fplus := func(x, y int) int { return x + y }
fplus(3,4)
// 直接调用匿名函数
func(x, y int) int { return x + y } (3, 4)
```

闭包实现斐波那契数列
``` go

// 跟Python不一样的是，不会发生自由变量降级成为局部变量的现象，也不需要nolocal声明。因为go语言需要变量声明，Python不需要变量声明。

// fibonacci is a function that returns
// a function that returns an int.
func fibonacci() func() int {
	pre2 := -1
	pre1 := 1
	return func() int {
		tmp := pre2
		pre2 = pre1
		pre1 = tmp + pre1
		return pre1
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Println(f())
	}
}
```
