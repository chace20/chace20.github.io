---
title: Go tour(3)-Concurrency
date: 2019-06-11 11:05:58
tags: Go
---
## 并发
 - gorotine 协程
 - channel 信道

``` go
ch := make(chan TYPE, BUFFER_SIZE) // 创建一个信道
go FUNC(ch) // 用协程执行FUNC，并传递一个信道
for y := range ch // 用range循环等待
```

### 练习：比较二叉树是否存储相同的值
``` go
package main

import "golang.org/x/tour/tree"
import "fmt"

// Walk walks the tree t sending all values
// from the tree to the channel ch.
func Walk(t *tree.Tree, ch chan int) {
	if t == nil {
		return
	}
	ch <- t.Value
	Walk(t.Left, ch)
	Walk(t.Right, ch)
}

// Same determines whether the trees
// t1 and t2 contain the same values.
func Same(t1, t2 *tree.Tree) bool {
	ch1 := make(chan int, 10)
	ch2 := make(chan int, 10)
	go Walk(t1, ch1)
	go Walk(t2, ch2)
	values1 := make([]int, 0, 10)
	values2 := make([]int, 0, 10)
	for i:=0; i<20; i++ {
		// 阻塞在select
		select {
		case v1 := <- ch1:
			values1 = append(values1, v1)
		case v2 := <- ch2:
			values2 = append(values2, v2)
		//在其余case没有准备好的时候，default会执行。不会阻塞
			//default:
			//fmt.Println("default")
		}
	}
	
	fmt.Printf("values1: %v\n", values1)
	fmt.Printf("values2: %v\n", values2)
	
	same := false
	for v1:=range values1 {
		same = false
		for v2:=range values2 {
			if v1 == v2 {
				same = true
				break
			}
		}
		if !same {
			break
		}
	}
	return same
}

func main() {
	ch := make(chan int, 10)
	go Walk(tree.New(1), ch)
	
	tmp := make([]int, 10)
	for i:=0; i<10; i++ {
		v, ok := <-ch
		if ok {
			tmp[i] = v
		} else {
			break
		}
	}
	fmt.Printf("tmp: %v\n", tmp)
	fmt.Println("----------")
	
	isSame := Same(tree.New(1), tree.New(2))
	fmt.Println("----------")
	fmt.Println(isSame)
}

```

1. 基于channel的同步
By default, sends and receives block until the other side is ready. This allows goroutines to synchronize without explicit locks or condition variables.
``` go
ch = make(chan int)
ch <- 2
ch <- 3 // 会出现死锁
```

2. 基于sync.Mutex的同步
``` go
var mux sync.Mutex
mux.Lock()
// do something
mux.Unlock()
```

3. 基于sync.WaitGroup同步
主线程等待所有协程返回
``` go
var wg sync.WaitGroup
wg.Add(1) // 在执行协程之前调用Add
go func(){
    defer wg.Done()
    // do something
}()
wg.Wait()
```
在执行协程之前调用Add。文档说明：Typically this means the calls to Add should execute before the statement creating the goroutine or other event to be waited for. 

## 练习：并发爬虫
``` go
package main

import (
	"fmt"
	"sync"
)

type Fetcher interface {
	// Fetch returns the body of URL and
	// a slice of URLs found on that page.
	Fetch(url string) (body string, urls []string, err error)
}

// Crawl uses fetcher to recursively crawl
// pages starting with url, to a maximum of depth.
func Crawl(url string, depth int, fetcher Fetcher) {
	// 用waitgroup实现主线程等待
	defer wg.Done()
	// TODO: Fetch URLs in parallel.
	// TODO: Don't fetch the same URL twice.
	// 判断是否已访问
	if safemap.IsVisited(url) {
		return
	}

	// This implementation doesn't do either:
	if depth <= 0 {
		return
	}
	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		fmt.Println(err)
		return
	}
	// 标记为已访问
	safemap.Visit(url)

	fmt.Printf("found: %s %q\n", url, body)
	for _, u := range urls {
		wg.Add(1)
		go Crawl(u, depth-1, fetcher)
	}
	return
}

func main() {
	wg.Add(1)
	go Crawl("https://golang.org/", 4, fetcher)
	wg.Wait()
}

// 访问过的url
type SafeMap struct {
	visited map[string]bool
	mux sync.Mutex
}

// 实现安全操作map
func (m SafeMap) IsVisited(url string) bool {
	m.mux.Lock()
	_, ok := m.visited[url]
	defer m.mux.Unlock()
	return ok
}

// 实现安全操作map
func (m SafeMap) Visit(url string) {
	m.mux.Lock()
	m.visited[url] = true
	m.mux.Unlock()
}

var safemap = SafeMap{visited: make(map[string]bool)}
var wg sync.WaitGroup

// fakeFetcher is Fetcher that returns canned results.
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

// fetcher is a populated fakeFetcher.
var fetcher = fakeFetcher{
	"https://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"https://golang.org/pkg/",
			"https://golang.org/cmd/",
		},
	},
	"https://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"https://golang.org/",
			"https://golang.org/cmd/",
			"https://golang.org/pkg/fmt/",
			"https://golang.org/pkg/os/",
		},
	},
	"https://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
	"https://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
}
```
