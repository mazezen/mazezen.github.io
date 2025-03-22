---
layout: post
title: "使用copier提升开发效率"
date:    2024-8-01
tags: [GO]
comments: true
author: mazezen
---

*截止到目前为止，go1.23的正式版还没有发布，但是其预发布版本，也就是rc版本已经开放下载了。初步体验一下*



### 安装

方式一

```shell
go install golang.org/dl/go1.23rc1@latest
go1.23rc1 download
```

方式二

```shell
curl -o https://go.dev/dl/go1.23rc2.darwin-arm64.tar.gz

# 编辑.zshrc
export GO111MODULE="on"
export GOROOT="$PWD/go123/go"
export GOPATH="$PWD/gopath"
export GOBIN=$GOROOT/bin
export PATH=$PATH:$GOBIN
export GOPROXY="https://goproxy.cn,direct"


source .zshrc
```



### range over func

[Go 1.22版本](https://tonybai.com/2024/02/18/some-changes-in-go-1-22/)引入了[range over func](https://go.dev/wiki/RangefuncExperiment)试验特性，通过GOEXPERIMENT=rangefunc。实现函数迭代器。在1.23开始，在语法层面上支持用户自定义函数实现迭代器，不需要加任何参数。

运行环境go1.23

```go
package main

import "fmt"

func Backward[E any](s []E) func(func(int, E) bool) {
	return func(yield func(int, E) bool) {
		for i := len(s) - 1; i >= 0; i-- {
			if !yield(i, s[i]) {
				return
			}
		}
		return
	}
}

func main() {
	sl := []string{"a", "b", "c"}
	for i, s := range Backward(sl) {
		fmt.Printf("i: %d, s: %s\n", i, s)
	}
}

```

运行结果: 

```markdown
i: 2, s: c
i: 1, s: b
i: 0, s: a

```

上述例子实现了索引按倒序排序


### 对slice和map增加了一些内置函数。
增加了比如slices.All()、slices.Backward() 等
代码示例:
```go
package main

import (
	"fmt"
	"slices"
)

func main() {
	sl := []string{"a", "b", "c"}
	for i, s := range slices.All(sl) {
		fmt.Printf("i: %d, s: %s\n", i, s)
	}

	for i, s := range slices.Backward(sl) {
		fmt.Printf("i: %d, s: %s\n", i, s)
	}
}

```
```markdown
i: 0, s: a
i: 1, s: b
i: 2, s: c
i: 2, s: c
i: 1, s: b
i: 0, s: a

```


### 有可能会限制linkename的使用

//go:linkename指令可用用来链接到标准库或其他包未导出的函数.

runtime.nanotime返回当前时间的纳秒数

代码示例:

```go
package main

import (
	"fmt"
	_ "unsafe"
)

//go:linkname nanotime runtime.nanotime
func nanotime() int64

func main() {
	fmt.Println("current time in nanoseonds: ", nanotime())
}

```

运行结果:

```markdown
current time in nanoseonds:  763815666064833
```



### 增加unique库

代码:

```go
package main

import (
	"fmt"
	"unique"
)

func main() {
	a1 := unique.Make("a")
	c1 := unique.Make("c")

	fmt.Println(a1 == c1)

	fmt.Println(a1.Value())
	fmt.Println(c1.Value())
}

```

结果

```markdown
false
a
c
```



### Timer和Ticker改动

go1.23对Timer和Ticker的垃圾回收进行了特殊的处理。早期版本需要调用调用Stop方法才会被垃圾回收期回收，因为Timer和Ticker的实现原理是独立的goroutine定时向chan写入数据。所以如果不调用Stop方法会造成goroutine泄露。现在做了特殊的优化，便不调用Stop方法，运行时也会检查并回收不用的Timer和Ticker



