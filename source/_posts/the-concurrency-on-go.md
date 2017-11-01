---
title: The Concurrency on Go
date: 2017-10-25 19:17:29
categories:
- golang
tags:
- concurrency
---
之前在学习MIT6.824的实验中，遇到了大量考虑程序并发性的内容，而go语言的特色之处就是在对并发性的支持上，因此我们来总结一下go语言在并发性方面的编程，希望给自己加深一下理解

## goroutinue
goroutine在我的理解中就是快速的开辟一个线程执行相应的函数
`go f(x, y, z)`
在新的线程中执行
`f(x, y, z)`
```go
package main
import (
	"fmt"
	"time"
)
func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}
func main() {
	go say("world")
	say("hello")
}
```
执行的结果为
{% asset_image all_1.png %}

## channel

channel在我理解是一个信道，信道中传输的数据类型可以是任意的，在另一端准备好之前，发送端和接收端都会被阻塞，在接收端接收的顺序是**先进先出**，类似于队列。
```go
ch <- v
x := <- ch
```
这一段代码的意思是在发送方将v送入，在接收端用x去接收,channel在使用前必须创建，
`ch := make(chan int)`
实验的代码如下所示

```go
package main

import "fmt"

func sum(a []int, c chan int) {
	sum := 0
	for _, v := range a {
		sum += v
	}
	c <- sum // 将和送入 c
}

func main() {
	a := []int{7, 2, 8, -9, 4, 0}
	c := make(chan int)
	go sum(a[:len(a)/2], c)
	go sum(a[len(a)/2:], c)
	x := <-c
	y := <-c
	fmt.Println(x, y, x+y)
}
```
代码的执行结果如下所示：
`17 -5 12`
由于此处使用的channel不带有缓冲，每在channel中放入一个变量，在取走之前`c <- sum`之前的代码阻塞，直到channel中的变量被取走，程序才会继续执行

## 带有缓冲的channel

带有缓冲的channel在构建时，`make()`的第二个参数作为缓冲区的大小
向缓冲区填充变量的时候，只有当缓冲区满的时候发送数据程序会阻塞，而当缓冲区为空的时候，接收变量的程序会阻塞

## range 和 close
`range`可以源源不断的从channel中读取数据直到关闭
`close`可以关闭channel表示再也没有值被发送了
`v, ok = <- ch`可以用来测试channel是否被关闭，若关闭ok的返回值为`false`
实验的代码如下所示：

```go
package main
import (
	"fmt"
)
func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)
}
func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c)
	for i := range c {
		fmt.Println(i)
	}
	_, ok := <-c
	fmt.Print(ok)
}
```
实验中实现的是打印斐波那契数列中的前10个数字，当产生完10个数字并全部放入缓冲区后，程序阻塞，主程序中的`range`开始读取channel缓存，当读取完毕后，函数`fibonacci()`关闭了channel，于是`ok = false`

实验的结果如下所示：

{% asset_image test4.png %}

## select
`select`有两种执行情况

1. 阻塞直到条件分支中的某个分支可以执行时，便会执行这个分支
2. 当有多个分支符合条件的时候，便会随机选择一个分支执行


实验的代码如下所示：
```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```
实验的结果如下图所示：
{% asset_image test5.png %}

### 结果解析

1. 序先执行至`fmt.Println(<-c)`时阻塞,接收程序缓冲区为空
2. 在`fibonacci()`函数中,`select`语句智能选择往channel c中放入数据，由于`quit`为空，一直阻塞
3. 当打印完成结果后，`quit <- 0`执行，`select`可以执行`case <- quit`之后的代码，打印`quit`并返回主程序

### 默认选择

当`select`中的其他条件分支都没有准备好的时候`default`分支会被执行
通常用作非阻塞的发送或接收程序

## sync.Mutex
当我们不需要在goroutine中通信时，我们可以使用互斥锁在使用**共享**资源时进行封锁，使用结束后进行解锁
go库中提供了`sync.Mutex`类型以及两个方法：
`Lock()`
`Unlock()`
参考的代码如下所示：


```go
func (c *SafeCounter) Inc(key string) {
	c.mux.Lock()
	// Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	c.v[key]++
	c.mux.Unlock()
}
```

在增加`c.v[key]`之前先对其进行封锁，使用结束后解锁，而在定义这个可以封锁的资源时，我们采用了结构体的方式，将这个互斥锁**绑定**在资源上
```go
type SafeCounter struct {
	v   map[string]int
	mux sync.Mutex
}
```
## 多种方式的crawer
```go
package main

import (
	"fmt"
	"sync"
)

//
// Several solutions to the crawler exercise from the Go tutorial (https://tour.golang.org/concurrency/10)
//

//
// Serial crawler
//

func CrawlSerial(url string, fetcher Fetcher, fetched map[string]bool) {
	if fetched[url] {
		return
	}
	fetched[url] = true
	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("found: %s %q\n", url, body)
	for _, u := range urls {
		CrawlSerial(u, fetcher, fetched)
	}
	return
}

//
// Concurrent crawler with Mutex and WaitGroup
//

type fetchState struct {
	mu      sync.Mutex
	fetched map[string]bool
}

func (f *fetchState) CheckAndMark(url string) bool {
	//defer f.mu.Unlock()

	//f.mu.Lock()
	if f.fetched[url] {
		return true
	}
	f.fetched[url] = true
	return false
}

func mkFetchState() *fetchState {
	f := &fetchState{}
	f.fetched = make(map[string]bool)
	return f
}

func CrawlConcurrentMutex(url string, fetcher Fetcher, f *fetchState) {
	if f.CheckAndMark(url) {
		return
	}

	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("found: %s %q\n", url, body)
	var done sync.WaitGroup
	for _, u := range urls {
		done.Add(1)
		go func(u string) {
			defer done.Done()
			CrawlConcurrentMutex(u, fetcher, f)
		}(u) // Without the u argument there is a race
	}
	done.Wait()
	return
}

//
// Concurrent crawler with channels
//

func dofetch(url1 string, ch chan []string, fetcher Fetcher) {
	body, urls, err := fetcher.Fetch(url1)
	if err != nil {
		fmt.Println(err)
		ch <- []string{}
	} else {
		fmt.Printf("found: %s %q\n", url1, body)
		ch <- urls
	}
}

func master(ch chan []string, fetcher Fetcher) {
	n := 1
	fetched := make(map[string]bool)
	for urls := range ch {
		for _, u := range urls {
			if _, ok := fetched[u]; ok == false {
				fetched[u] = true
				n += 1
				go dofetch(u, ch, fetcher)
			}
		}
		n -= 1
		if n == 0 {
			break
		}
	}
}

func CrawlConcurrentChannel(url string, fetcher Fetcher) {
	ch := make(chan []string)
	go func() {
		ch <- []string{url}
	}()
	master(ch, fetcher)
}

//
// main
//

func main() {
	fmt.Printf("=== Serial===\n")
	CrawlSerial("http://golang.org/", fetcher, make(map[string]bool))

	fmt.Printf("=== ConcurrentMutex ===\n")
	CrawlConcurrentMutex("http://golang.org/", fetcher, mkFetchState())

	fmt.Printf("=== ConcurrentChannel ===\n")
	CrawlConcurrentChannel("http://golang.org/", fetcher)
}

//
// Fetcher
//

type Fetcher interface {
	// Fetch returns the body of URL and
	// a slice of URLs found on that page.
	Fetch(url string) (body string, urls []string, err error)
}

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
	"http://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"http://golang.org/pkg/",
			"http://golang.org/cmd/",
		},
	},
	"http://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"http://golang.org/",
			"http://golang.org/cmd/",
			"http://golang.org/pkg/fmt/",
			"http://golang.org/pkg/os/",
		},
	},
	"http://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
	"http://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"http://golang.org/",
			"http://golang.org/pkg/",
		},
	},
}
```

### 代码解析
#### 串行的方式爬取内容
1. 在`CrawlSewrial`中首先判断当前这个`url`是否被爬取过
2. 若未爬取`body, urls, err := fetcher.Feach(url)`，打印当前的`body`
3. 遍历urls，继续爬取内容


#### 使用Mutex并行爬取数据
```go
var done sync.WaitGroup
for _, u := range urls {
	done.Add(1)
	go func(u string) {
		defer done.Done()
		CrawlConcurrentMutex(u, fetcher, f)
	}(u) // Without the u argument there is a race
}
done.Wait()
```
遍历`urls`，每读取一条就并行的执行，并将任务添加进入`var done sync.WaitGroup`中
在循环外设置`done.Wait()`等待所有的线程执行完毕后继续执行主线程代码

#### 使用Channel并行爬取数据
```go
func master(ch chan []string, fetcher Fetcher) {
	n := 1
	fetched := make(map[string]bool)
	for urls := range ch {
		for _, u := range urls {
			if _, ok := fetched[u]; ok == false {
				fetched[u] = true
				n += 1
				go dofetch(u, ch, fetcher)
			}
		}
		n -= 1
		if n == 0 {
			break
		}
	}
}
```
将要爬取的`url`放入`channel`中，使用`master`处理，判断是否已经被访问过
若否开辟一个线程去访问，并将新的`urls`放入`channel`中 `n+1`
读完一个`ch`中的`urls`后`n-1`直到最后`n=0`退出循环，所有的`url`都已经访问










