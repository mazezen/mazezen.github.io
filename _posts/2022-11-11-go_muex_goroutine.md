---
layout: post
title: "Go 控制Goroutine的数量"
date:    2022-11-11
tags: [Go]
comments: true
author: mazezen
---


Goroutine虽然体量很小（2kb），理论可以开启上百万个Goroutine。但也不是多多益善。一旦Goroutine过多，会占用大量的cpu 内存，可能导致服务器速度变慢甚至服务挂掉。



先看一下不控制Goroutine数量，看能跑多少

Cpu: 4

Mem: 16G

```go
	tasks := math.MaxInt64
	for i := 0; i < tasks; i++ {
		go func(i int) {
			fmt.Println("go func ", i, " goroutine count = ", runtime.NumGoroutine())
		}(i)
	}

```

```shell
go func  1327774  goroutine count =  1029277
panic: too many concurrent operations on a single file or socket (max 1048575)

goroutine 1352265 [running]:
internal/poll.(*fdMutex).rwlock(0xc0000540c0, 0x20)
```



如何控制Goroutine

```go
type Pool struct {
	queue chan int
	wg    *sync.WaitGroup
}

func NewPool(size int) *Pool {
	if size <= 0 {
		size = 1
	}
	return &Pool{
		queue: make(chan int, size),
		wg:    &sync.WaitGroup{},
	}
}

func (p *Pool) Add(task int) {
	for i := 0; i < task; i++ {
		p.queue <- task
	}
	p.wg.Add(task)
}

func (p *Pool) Done() {
	<-p.queue
	p.wg.Done()
}

func (p *Pool) Wait() {
	p.wg.Wait()
}

func main() {
	pool := NewPool(5)
	fmt.Println("the NumGoroutine begin is:", runtime.NumGoroutine())
	for i := 0; i < 53; i++ {
		pool.Add(1)
		go func() {
			time.Sleep(time.Second)
			fmt.Println("the NumGoroutine continue is:", runtime.NumGoroutine())
			pool.Done()
		}()
	}
	pool.Wait()
	fmt.Println("the NumGoroutine done is:", runtime.NumGoroutine())
}
```

 