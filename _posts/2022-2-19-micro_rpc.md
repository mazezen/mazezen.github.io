---
layout: post
title: "Go微服务 rpc"
date: 2022-2-19
tags: [Go]
comments: true
author: mazezen
---


## 1. 简介 什么是RPC？

- 远程过程调用（Remote Procedure Call, RPC）。计算机的一个通信协议

- 该协议允许一台计算机的程序调用另一台计算机的程序。
- 如果涉及的软件采用的面向对象编程，远程调用也可以叫做远程调用或远程方法调用



## 2. Go实现RPC

### 2.1 如何实现

- go使用官方提供的nei/rpc包使用encoding/gob进行编码，支持tcp和http数据传输,由于其他语言暂不支持gob编码方式，所以go的rpc只支持go开发的服务与客户端之间的交互
- 但是官方还提供了net/rpc/jsonrpc库实现rpc方法，jsonrpc采用的是json格式进行数据编码，因此支持跨语言调用，目前jsonrpc是基于tcp协议实现的，暂不支持http传输方式

### 2.2 Go写RPC，必须符合的四个条件，不然使用不了

- 结构体字段首字母要大写，可以别人调用
- 函数名必须首字母大写
- 函数第一参数是接收参数，第二个参数是返回给客户端的参数，必须是指针类型
- 函数还必须有一个返回值error

### 2.3 RPC调用流程

- 微服务架构下数据交互一般是对内RPC，对外REST

- 将应用按照业务边界拆分成各个服务，降低耦合度
- 各个服务单独运行，通过网络调用

### 2.4 RPC服务端

- 服务端接收到的数据包括调用的函数名、参数列表、还有一个返回值error
- 服务端核心功能包括维护函数map、接收客户端传来的参数解析处理相应的逻辑、函数的返回值打包，传给客户端

### 2.5 RPC - 实现 支持tcp和http 但是不支持跨语言

Example:

server.go

```go
package main

import (
	"log"
	"net/http"
	"net/rpc"
)

type Params struct {
	Width, Height int
}

type Rect struct{}

// RPC服务端方法，求矩形面积
func (r *Rect) Area(p Params, ret *int) error {
	*ret = p.Height * p.Width
	return nil
}

// 周长
func (r *Rect) Perimeter(p Params, ret *int) error {
	*ret = (p.Height + p.Width) * 2
	return nil
}

// 主函数
func main() {
	// 1.注册服务
	rect := new(Rect)
	// 注册一个rect的服务
	rpc.Register(rect)
	// 2.服务处理绑定到http协议上
	rpc.HandleHTTP()
	// 3.监听服务
	err := http.ListenAndServe(":8095", nil)
	if err != nil {
		log.Panicln(err)
	}
}
```

client.go

```go
package main

import (
	"fmt"
	"log"
	"net/rpc"
)

// 传的参数
type Param struct {
	Width, Height int
}

func main() {
	// 1.连接远程rpc服务
	conn, err := rpc.DialHTTP("tcp", ":8095")
	if err != nil {
		log.Fatal(err)
	}
	// 2.调用方法
	// 面积
	ret := 0
	err2 := conn.Call("Rect.Area", Param{50, 100}, &ret)
	if err2 != nil {
		log.Fatal(err2)
	}
	fmt.Println("面积：", ret)
	// 周长
	err3 := conn.Call("Rect.Perimeter", Param{50, 100}, &ret)
	if err3 != nil {
		log.Fatal(err3)
	}
	fmt.Println("周长：", ret)
}
```



### 2.6 jsonrpc - 实现 - 支持tcp，支持跨语言，但是不支持http

因为rpc只能用于go开发的服务之间进行通信，所以下面采用jsonrpc实现跨语言实现rpc

Example:

server.go

```go
package main

import (
	"fmt"
	"log"
	"net"
	"net/rpc"
	"net/rpc/jsonrpc"
)

type Params struct {
	Width, Height int
}
type Rect struct {
}

func (r *Rect) Area(p Params, ret *int) error {
	*ret = p.Width * p.Height
	return nil
}
func (r *Rect) Perimeter(p Params, ret *int) error {
	*ret = (p.Height + p.Width) * 2
	return nil
}
func main() {
	rpc.Register(new(Rect))
	lis, err := net.Listen("tcp", ":8096")
	if err != nil {
		log.Panicln(err)
	}
	for {
		conn, err := lis.Accept()
		if err != nil {
			continue
		}
		go func(conn net.Conn) {
			fmt.Println("new client")
			jsonrpc.ServeConn(conn)
		}(conn)
	}
}
```

client.go

```go
package main

import (
	"fmt"
	"log"
	"net/rpc/jsonrpc"
)

type Param struct {
	Width, Height int
}

func main() {
	conn, err := jsonrpc.Dial("tcp", ":8096")
	if err != nil {
		log.Panicln(err)
	}
	ret := 0
	err2 := conn.Call("Rect.Area", Param{50, 100}, &ret)
	if err2 != nil {
		log.Panicln(err2)
	}
	fmt.Println("面积：", ret)
	err3 := conn.Call("Rect.Perimeter", Param{50, 100}, &ret)
	if err3 != nil {
		log.Panicln(err3)
	}
	fmt.Println("周长：", ret)
}
```

php客户端调用go server服务

```go
<?php

class Client {

    private $conn;

    function __construct($host, $port) {
        $this->conn = fsockopen($host, $port, $errno, $php_errormsg, 3);
        if ($this->conn) {
            return false;
        }
    }

    public function Call($method, $params) {
        if (!$this->conn) {
            return false;
        }

        $err = fwrite($this->conn, json_encode(array(
            "method" => $method,
            "params" => array($params),
            "id" => 0,
        ))."\n");

        if ($err === false) {
            return false;
        }

        stream_set_timeout($this->conn, 0, 3000);
        $line = fgets($this->conn);

        if ($line === false) {
            return false;
        }

        return json_decode($line, true);
    }
}

$client = new Client("127.0.0.1", "8096");
$args = array("A" => 9, "B" => 2);
$res = $client->Call("Arith.Multiply", $args);
printf("%d * %d = %d\n", $args["A"], $args["B"], $res["result"]["Pro"]);

$r = $client->Call("Arith.Divide", $args);
printf("%d / %d, Quo is %d, Rem is %d\n", $args['A'], $args['B'], $r['result']['Quo'], $r['result']['Rem']);

```

### 2.7 protorpc - 实现 - 支持tcp和http， 支持跨语言

安装：[下载地址mac64位](https://github.com/protocolbuffers/protobuf/releases/download/v3.5.0/protoc-3.5.0-osx-x86_64.zip)

下载完，解压

```shell
// 配置全局变量
vim ~/.zshrc

#protoc
export PATH=/Users/cc/protoc-3.5.0-osx/bin:$PATH

source ~/.zshrc

protoc --version //查看
```

切换到自己的项目目录下，在go中安装protobuf相关的库

```shell
go get -u github.com/golang/protobuf/protoc-gen-go
```

![image-20220228093755435](http://images.caixiaoxin.cn//image-20220228093755435.png)

Note : 执行安装包错

1. 现在想要拉取protoc-gen-go需要去**google.golang.org/protobuf**拉取，原来的路径已经废弃了。
2. 我使用的go版本是1.17。而**Go1.17版使用go install安装依赖**。所以应该按照它下面的格式**go install pkg@version**进行拉取。

把上面的命令改为：

```shell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

安装成功之后,创建一个.proto文件

pb/arith.proto

```protobuf
syntax = "proto3";

option go_package = "../pb";

// 算术运算符结构
message ArithRequest {
  int32 a = 1;
  int32 b = 2;
}

// 算术运算符响应结构
message ArithReponse {
  // 乘积
  int32 pro = 1;
  // 商
  int32 que = 2;
  // 余数
  int32 rem = 3;
}

// rpc方法
service ArithService {
  // 乘法运算方法
  rpc multiply (ArithRequest) returns (ArithReponse);
}
```

```vim
cd proto && protoc --go_out=. --go-grpc_out=. ./arith.proto
```




在pb目录下生成了arith.pb.go文件



基于生成的`arith.pb.go`代码我们来实现一个rpc服务端
```go
package main

import (
	"net"
	"rpc-demo/protorpc/server/pb"
	"rpc-demo/protorpc/server/service"

	"google.golang.org/grpc"
)

type Arith struct{}

func (a *Arith) Multiply(req *pb.ArithRequst, res *pb.ArithResponse) error {
	res.Pro = req.GetA() * req.GetB()
	return nil
}

func main() {
	rpcServer := grpc.NewServer()
	pb.RegisterArithSeriveServer(rpcServer, new(service.ArithSerive))
	listen, _ := net.Listen("tcp", ":8087")
	rpcServer.Serve(listen)
}

```

客户端
```go
package main

import (
	"context"
	"fmt"
	"log"
	"rpc-demo/protorpc/server/pb"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial(":8087", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}
	defer conn.Close()

	client := pb.NewArithSeriveClient(conn)
	response, err := client.Multiply(context.Background(), &pb.ArithRequst{
		A: 100,
		B: 20,
	})
	if err != nil {
		log.Fatalln(err)
	}
	fmt.Println(response.Pro)
}

```



