---
title: Go tour(2)-Methods and Interfaces
date: 2019-06-11 11:05:45
tags: Go
---
## 方法

Go没有类，方法跟函数的区别就是多了一个接收器Receiver

指针接收器，接收到的变量是指针形式的，意味着可以修改原变量。pointer receivers比value receivers更加常见。

函数的参数为指针必须显式传入一个指针，但如果是方法，Go解释器会隐式地将对象的指针传入，不需要显式取地址。同样的情况也发生在参数为值value的情况下，方法依然会进行隐式转换。
``` go
var v Vertex
ScaleFunc(v, 5) // error
ScaleFunc(&v, 5) // ok
v.Scale(5) // OK
(&v).Scale(5) // OK
```
指针接收器优点: 1.可以改变原变量；2.避免了传值拷贝

## 接口
接口也是一种type。没有类，感觉像是在type上去实现方法。可以看成(value, type)的元组。
接口内的值就算是nil，也可以调用它的方法，不会报空指针异常。但是如果是nil接口，就会报错。
``` go
// 不会报错
func (t *T) M() {}
var t *T
t.M()
// 会报错
var i I
i.M()

type I interface {
    M()
}
type F float64
func (f F) M() {} // 这样就实现了接口I
var i I = F(1.2) // 可以给接口赋值
i.M() // 调用接口实现中的方法

// 空接口可以接收任意类型的值
func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}
// 判断接口具体的类型，跟map类似
t, ok := i.(T)
```

Error处理
``` go
// 实现了error接口的ErrNegativeSqrt
type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
	return fmt.Sprintf("cannot Sqrt negative number: %v", float64(e))
}

func Sqrt(x float64) (float64, error) {
	if x > 0 {
		z := 1.0
		for last_z := z; math.Abs(last_z-z) > 0.00001; last_z = z {
			z -= (z*z - x) / (2*z)
		}
		return z, nil
	}
	return 0, ErrNegativeSqrt(x)
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}

// 实现了从无限流中读取'A'的Read方法
type MyReader struct{}

func (m MyReader) Read(bytes []byte) (n int, e error) {
	for i:=0; i<len(bytes); i++ {
		bytes[i]='A'
	}
	return len(bytes), nil
}
```

在io.Reader上包装，实现rot13算法
``` go
package main

import (
	"io"
	"os"
	"strings"
)

type rot13Reader struct {
	r io.Reader
}

func transform(b byte) (bb byte) {
	if (b >= 65 && b < 78) || (b >= 97 && b < 110) {
		bb = b+13
	} else if (b >= 78 && b < 91) || (b >= 110 && b < 123) {
		bb = b-13
	}
	return bb
}

func (rot rot13Reader) Read(b []byte) (int, error) {
	n, e := rot.r.Read(b)
	for i:=0; i<n; i++ {
		b[i] = transform(b[i])
	}
	return n, e
} 

func main() {
	s := strings.NewReader("Lbh penpxrq gur pbqr!")
	r := rot13Reader{s}
	io.Copy(os.Stdout, &r)
    // output: Youcrackedthecode
}
```

实现Image接口
``` go
package main

import (
	"golang.org/x/tour/pic"
	"image/color"
	"image"
)
// 定义了一个Image结构体
type Image struct{
	w int
	h int
}
// Image实现Image的接口
/*
type Image interface {
    ColorModel() color.Model
    Bounds() Rectangle
    At(x, y int) color.Color
}
*/
func (img Image) ColorModel() color.Model {
	return color.RGBAModel
}

func (img Image) Bounds() image.Rectangle {
	return image.Rect(0, 0, img.w, img.h)
}

func (img Image) At(x, y int) color.Color {
	v := uint8(x^y)
	return color.RGBA{v, v, 255, 255}
}

func main() {
	m := Image{w:200, h:200}
	// 使用实现了Image接口的m
	pic.ShowImage(m)
}

```
