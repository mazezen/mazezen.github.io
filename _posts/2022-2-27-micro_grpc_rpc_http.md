---
layout: post
title: "Go微服务八  Go-gGRPC 不同端口提供rpc和http服务"
date: 2022-2-27
tags: [Go]
comments: true
author: mazezen
---

与<a href="http://blog.caixiaoxin.cn/?p=578" target="_blank" rel="noopener">[微服务六]</a>略有不同。

以商品详情场景解释本篇和<a href="http://blog.caixiaoxin.cn/?p=578" target="_blank" rel="noopener">[微服务六]</a>还有<a href="http://blog.caixiaoxin.cn/?p=591" target="_blank" rel="noopener">[微服务七]</a>的不同之处

微服务六: 不同端口同时支持rpc和http。

​	场景: 商品服务-> 商品库存等

​	商品服务-> 商品库存提供rpc:端口(9001)   客户端可以通过rpc调用商品库存rpc接口服务，

此时客户端可能没有使用rpc，只会http api调用的形式。这个时候客户端也可以通过http, 调用用商品库存rpc接口提供的http: 端口(8001)

<font style="color: red">相当于一个商品库存，通过基于protobuf协议封装的grpc即提供了rpc又提供了http，而且端口还不同，这样也可以，在基于protobuf封装rpc的时候，加一个options参数，使用对应的路由实现。但是有一点不方便，不管是rpc还是http api变动都需要使用proto重新生成对应的*.pb.go文件，增加维护成本，如果调用方不支持rpc可以使用此方式</font>



微服务七： 同一个端口支持grpc和http

​	场景: 商品服务-> 商品库存、商品详情、商品列表等

​	商品服务-> 商品库存rpc接口端口(9001)  客户端可以通过rpc调用商品库存rpc接口，但不能通过http调用商品库存rpc服务接口. 但是此时商品服务可能还存在一些其他的api，比如商品详情、商品列表等，此时不想通过封装rpc供客户端调用，想通过简单的http api方式供客户端使用，但是又不想一个商品服务同时暴漏两个端口，可以采用此方法。比方客户端可以通过http调用商品服务另外提供的api，端口也是9001

<font style="color: red;">不建议,最好还是分别使用不同的端口，端口对应独立的服务</font>



本篇文章: 不同端口提供rpc和http

​	场景: 商品服务-> 商品库存、商品详情、商品列表等
商品服务 ->商品库存只提供rpc 但是商品详情和商品列表不提供rpc只提供http api，端口又不同

 



目录结构

![image-20220311173132032](http://images.caixiaoxin.cn//image-20220311173132032.png)

### 一 创建项目

```shell
mkdir demo

cd demo && go mod init demo && go get google.golang.org/grpc && touch generate_pb && chmod +x generate_pb

mkdir demo-server && mkdir demo-client

cd demo-server && mkdir pb && mkdir proto && mkdir rpcServices && touch main.go

cd demo-client && touch main.go
```



### 二 代码编写

rpcDemoService.proto

```protobuf
syntax = "proto3";

package pb;

option go_package = "../pb";

message GetUserInfoRequest {
  int32 id = 1;
}

message GetUserInfoResponse {
  string name = 1;
  int32 age = 2;
}

service RpcDemoService {
  rpc GetUserInfo(GetUserInfoRequest) returns (GetUserInfoResponse){}
}
```

generate_pb

```shell
cd proto && protoc --go_out=plugins=grpc:../pb rpcDemoService.proto

```

执行./generate_pb,会在pb目录闲生成*.pb.go文件



rpcDemoService.go

```go
package rpcServices

import (
	"context"
	"demo/demo-server/pb"
)

type RpcDemoService struct{}

func (rd *RpcDemoService) GetUserInfo(ctx context.Context, r *pb.GetUserInfoRequest) (*pb.GetUserInfoResponse, error) {
	return &pb.GetUserInfoResponse{
		Name: "demo",
		Age:  18,
	}, nil
}

```

main.go

```go
package main

import (
	"demo/demo-server/pb"
	"demo/demo-server/rpcServices"
	"flag"
	"log"
	"net"
	"net/http"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

var grpcPort string
var httpPort string

func init() {
	flag.StringVar(&grpcPort, "g", "8001", "gRPC 启动端口号")
	flag.StringVar(&httpPort, "h", "9001", "HTTP 启动端口号")
	flag.Parse()
}

//RunHttpServer
func RunHttpServer(port string) error {
	serveMux := http.NewServeMux()
	serveMux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte(`pong`))
	})

	return http.ListenAndServe(":"+port, serveMux)
}

//RunGrpcServer
func RunGrpcServer(port string) error {
	s := grpc.NewServer()
	pb.RegisterRpcDemoServiceServer(s, new(rpcServices.RpcDemoService))
	reflection.Register(s)
	lis, err := net.Listen("tcp", ":"+port)
	if err != nil {
		return err
	}

	return s.Serve(lis)
}

func main() {
	errs := make(chan error)
	go func() {
		err := RunHttpServer(httpPort)
		if err != nil {
			errs <- err
		}
	}()
	go func() {
		err := RunGrpcServer(grpcPort)
		if err != nil {
			errs <- err
		}
	}()

	select {
	case err := <-errs:
		log.Fatalf("Run Server err: %v", err)
	}
}

```

 cd demo-client

main.go

```go
package main

import (
	"context"
	"demo/demo-server/pb"
	"fmt"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial(":8001", grpc.WithInsecure())
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	demoServiceClient := pb.NewRpcDemoServiceClient(conn)
	result, err := demoServiceClient.GetUserInfo(context.Background(), &pb.GetUserInfoRequest{
		Id: 10,
	})
	if err != nil {
		panic(err)
	}

	fmt.Println(result)
}

```

测试grpc

执行

```shell
cd demo-server
go run main.go
```

postman 或者curl访问 127.0.0.1:9001/ping

![image-20220311173928559](http://images.caixiaoxin.cn//image-20220311173928559.png)

测试http

执行

```shell
cd demo-client && go run main.go
```

![image-20220311174052210](http://images.caixiaoxin.cn//image-20220311174052210.png)



修改demo-server/main.go 结合echo框架支持restful api

```go
package main

import (
	"demo/demo-server/pb"
	"demo/demo-server/rpcServices"
	"flag"
	"log"
	"net"
	"net/http"

	"github.com/labstack/echo"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

var grpcPort string
var httpPort string

func init() {
	flag.StringVar(&grpcPort, "g", "8001", "gRPC 启动端口号")
	flag.StringVar(&httpPort, "h", "9001", "HTTP 启动端口号")
	flag.Parse()
}

//RunHttpServer
func RunHttpServer(port string) {
	//serveMux := http.NewServeMux()
	//serveMux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
	//	_, _ = w.Write([]byte(`pong`))
	//})
	//
	//return http.ListenAndServe(":"+port, serveMux)
	e := echo.New()
	e.GET("/ping", func(c echo.Context) error {
		return c.JSON(http.StatusOK, "pong...")
	})
	e.Logger.Fatal(e.Start(":" + port))
}

//RunGrpcServer
func RunGrpcServer(port string) error {
	s := grpc.NewServer()
	pb.RegisterRpcDemoServiceServer(s, new(rpcServices.RpcDemoService))
	reflection.Register(s)
	lis, err := net.Listen("tcp", ":"+port)
	if err != nil {
		return err
	}

	return s.Serve(lis)
}

func main() {
	errs := make(chan error)
	go func() {
		RunHttpServer(httpPort)
	}()
	go func() {
		err := RunGrpcServer(grpcPort)
		if err != nil {
			errs <- err
		}
	}()

	select {
	case err := <-errs:
		log.Fatalf("Run Server err: %v", err)
	}
}

```


