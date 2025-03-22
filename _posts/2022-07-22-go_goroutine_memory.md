---
layout: post
title: "Go编写开机自启动服务"
date:    2022-07-22
tags: [Go]
comments: true
author: mazezen
---


### 1.闲聊

在远古时期cpu都是以单核的形式执行机器的指令。随着科技和时代的进步，单核无法满足人类日益贪婪的需求，于是多核cpu应用而生。编程语言也不甘示弱，开始利用多核cpu的优势逐渐走向了并行的方向。

Go语言在出生的时候，就已经是多核时代的天下了。于是那群牛逼的Go语言之父呢，就结合了多门语言的特性，创造了Go语言自身的并发机制。这也是为什么Go语言有着“天生高并发“的称号。

### 2.内存模型

##### 2.1 什么是内存模型

在多核多线程的背景下，多个不同的cpu是如何以一种统一的形式来与内存进行交互的。

##### 2.2 内存模型有哪些？

多线程、消息传递、顺序一致性内存模型等

### 3. 回归

这篇文章是写Go的并发的内存模型，不讲其他的(因为也不懂)

聊Go的并发离不开Goroutine。

### 4. 什么是Goroutine

Goroutine是go语言独有的并发体。名字叫协程，其实就是一种轻量级的线程。我们都知道每个线程都有一个固定的大小栈，一般默认为2MB，Goroutine呢也有大小，她的大小栈呢默认为2KB或4KB，在内存空间高昂的年代，小就是牛逼。

线程呢固定了栈的小大产生了两个问题：

1. 对于很多只需要很小的栈空间的线程来说是一个巨大的浪费
2. 对于少数需要巨大栈空间的线程来说又面临栈溢出的风险

如何解决：

要么降低固定的栈大小，提升空间的利用率；要么增大栈的大小以允许更深的函数递归调用，但这两者是没法同时兼得

于是呼Goroutine站了出来。上面讲到了Goroutine的栈大小默认为2KB或4KB，所以她启动的时候占用很小的空间资源，但是当遇到深度递归导致当前栈空间不足时，Goroutine会根据需要动态的伸缩栈的大小（主流实现中栈的最大值可达到1GB). 有一种你线程解决不了，我解决了！一副傲视群雄的样子。Goroutine理论上可以启动成千上万个。这么牛逼，如何启动呢，她只需要一个 `go` 关键字即可启动。

Go呢自带调度器，可以调度Goroutine。调度器呢属于半抢半占的形式进行调度的，当当前的Goroutine发生阻塞时，才会导致调度。（有点类似于男人之间的对话，你行不行 不行我来的意思）

闲话不多话，遭多也没了～



### 5. 原子操作

出现这个问题的场景是在并发的情况下，出现了多个并发体对同一个共享资源数据竞争的问题。如何解决呢。

##### 5.1  加锁

可以通过 `互斥锁` 实。Go 语言提供了两个包，分别为sync.Mutex和sync.RWMutex，在对同一资源操作的时候比如更新、删除、读等，对当前的Goroutine进行加锁。

```go
package main

import (
	"fmt"
	"sync"
)

var (
	m     sync.Mutex
	v int
)

func do(wg *sync.WaitGroup) {
	defer wg.Done()

	for i := 0; i <= 100; i++ {
		m.Lock()
		v++
		m.Unlock()
	}
}

func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	go do(&wg)
	go do(&wg)
	wg.Wait()

	fmt.Println(v)
}
```

##### 5.2 sync/atomic包

加锁在并发量和数据量大的情况下，会导致一个性能下载的问题。那么Go语言提供了sync/atomic包，内置了对一个数值型的共享资源的原子操作。简直不要太好～

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

var score uint64

func do(wg *sync.WaitGroup) {
	defer wg.Done()

	var i uint64
	for i = 0; i <= 100; i++ {
		atomic.AddUint64(&score, i)
	}
}

func main() {
	var wg sync.WaitGroup
	wg.Add(2)

	go do(&wg)
	go do(&wg)

	wg.Wait()
	fmt.Println(score)
}
```

如果你用过sync.Once 实现的单列模式，会发现它也是使用atomic实现的。

```go
// A Once must not be copied after first use.
type Once struct {
	// done indicates whether the action has been performed.
	// It is first in the struct because it is used in the hot path.
	// The hot path is inlined at every call site.
	// Placing done first allows more compact instructions on some architectures (amd64/386),
	// and fewer instructions (to calculate offset) on other architectures.
	done uint32
	m    Mutex
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
// 	var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// 	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}
```





### 6 顺序一致性内存模型

Goroutine 是以一种异步的形式执行的。如何保证多个Goroutine按顺序执行呢？

看下面代码

```go
package main

func main() {
	go func() {
		println("我会执行吗？")
	}()
}
```

这段代码会打印出来这句话吗？

写Go的小伙伴都知道，main函数其实也是一个Goroutine,当main所在的Goroutine执行完之后，就会执行os.Exit(1)退出当前Goroutine，main退出了，那句也就无法打印出来了。

如何实现 打印出 `我会执行吗？`这句话呢？

方法呢有多种，一一来实现，可能不全，望指正～

##### 6.1 第一种 for 阻塞

有一点不太友好的地方，会一直阻塞下去，直到天荒地老～

```go
package main

func main() {
	go func() {
		println("我会执行吗？")
	}()
	for {

	}
}
```

##### 6.2 第二种 加锁

```go
package main

import "sync"

func main() {
	var wg sync.Mutex

	wg.Lock()
	go func() {
		println("我会执行吗？")
		wg.Unlock()
	}()
	wg.Lock()
	println("我肯定执行的...")
}
```

##### 6.3 第三种 通道

Go语言内置channel 分为两种，一种是无缓冲的通道和有缓冲的通道。无缓冲的通道，在从通道接受前一定会先执行往通道里发送。其实可以理解为无缓冲的通道是同步的。通道更详细的理解自行查看相关文档

采用无缓冲的通道来实现

```go
package main

func main() {
	c := make(chan int)

	go func() {
		println("我会执行吗？")
		c <- 1
	}()
	<-c
}
```

##### 6.4 第四种 

Go语言内置了sync.WaitGroup包 等带一个Goroutine执行完，执行下一个Goroutine。代码实现

```go
package main

import "sync"

func main() {
	var s sync.WaitGroup

	s.Add(1)
	go func() {
		println("我会执行吗？")
		s.Done()
	}()
	s.Wait()
}
```
