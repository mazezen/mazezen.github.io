---
layout: post
title: "Go 使用IP纯真库获取IP对应的国家、省、市"
date: 2022-02-16
tags: [Go]
comments: true
author: mazezen
---

runtime里的三个函数Gosched、Goexit、GOMAXPROCS

Gosched：让出cpu时间切片。用于让出当前grouting的执行权限，调度器安排其他等待的goroutine执行任务，并在某个位置恢复执行


Goexit：终止当前的goroutine执行，并不会影响其他的goroutine执行。并在终止当前的goroutine之前，执行还未执行的defery语句. 如果Goexit在main函数中执行会报panic


GOMAXPROCS： 用来设置可以并行计算的CPU核数最大值，并返回之前的值。 1～256 最好在主函数之前设置，否则会终止当前的任务执行

Go 默认是一个cpu核的，可以通过GOMAXPROCS来设置

Gosched：

不加Gosched, 子协程并不会打印出来

```go
func main() {
  // 子协程
	go func() {
		for i := 0; i < 2; i++ {
			fmt.Println("会打印吗")
		}
	}()

	// 主协程
	for i := 0; i < 2; i++ {
		fmt.Println("我会打印！！！！")
	} 
}
```

结果：

```shell
我会打印！！！！
我会打印！！！！
```

加Gosched, 子协程就可以打印出来

```go
func main() {
  	go func() {
		for i := 0; i < 2; i++ {
			fmt.Println("会打印吗")
		}
	}()

	// 主协程
	for i := 0; i < 2; i++ {
		runtime.Gosched()
		fmt.Println("我会打印！！！！")
	}
}
```

结果：

```shell
会打印吗
会打印吗
我会打印！！！！
我会打印！！！！
```





Goexit：

```go
func test() {
	defer fmt.Println("还未执行的defer语句")
	runtime.Goexit()
	fmt.Println("这条信息还能打印吗？？？？")
}

func main() {
  // 子协程
	go func() {
		fmt.Println("哈哈哈哈哈哈哈")
		test()
		fmt.Println("来打我呀！！！！！")
	}()

  // 死循环，为了不让主协程退出
	for {

	}
}
```

结果:

```shell
哈哈哈哈哈哈哈
还未执行的defer语句
```





GOMAXPROCS:

```go
func main() {

	runtime.GOMAXPROCS(2)

	for {
		go fmt.Println("go")
		fmt.Print(0)
	}
}
```



