---
layout: post
title: "Go微服务四 gRPC"
date: 2022-2-20
tags: [Go]
comments: true
author: mazezen
---



## 环境

​	环境： 

- mac
- go1.17.7

## 安装Protobuf

- <a href="https://github.com/protocolbuffers/protobuf/releases?page=5" target="_blank" rel="noopener">[protoc github下载地址]</a>

  下载下来之后解压，配置.zshrc 

  ```shell
  #protoc
  export PATH=/Users/cc/protoc-3.9.0/bin:$PATH
  
  source ~/.zshrc
  ```

- 安装 golang 的proto工具包

  ```shell
  go get -u google.golang.org/protobuf/proto
  ```

- 安装 goalng 的proto编译支持  会在$GOPATH/bin目录下创建protocole-gen-go文件

  ```shell
  go get -u google.golang.org/protobuf/protoc-gen-go
  OR
  go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
 go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

  ```

## 安装gRPC

​	创建grpcpro项目，在项目目录下执行

```shell
$ go mod init grpcpro
```

安装 gRPC 包

```shell
go get -u google.golang.org/grpc
```



## 编写proto文件

创建项目grpcserver

在项目目录下新建pbfiles文件夹,在pbfiles文件夹下创建Prod.proto文件

```protobuf
syntax = "proto3";

package services;

option go_package = "../services";

message ProdRequest {
  int32 prod_id = 1;
}

message ProdResponse {
  int32 prod_stock = 1;
}
```

cd到pbfiles目录下, 执行  <span style="color: red">protoc --go_out=../services/ Prod.proto</span>  (services是生成的.pb.go存放位置,如果没有services目录自行手动先创建在执行)

## 创建服务端

修改pbfiles/Prod.proto文件,新增以下代码

```protobuf
service ProdService {
  rpc GetProdStock (ProdRequest) returns (ProdResponse);
}
```

执行 <span style="color: red"> protoc --go_out=plugins=grpc:../services Prod.proto </span>会重新生成并覆盖原来的Prod.pb.go文件
services/ProdService.go
```go
package services

import (
	"context"
)

type ProdService struct {
}

func (this *ProdService) GetProdStock(ctx context.Context, request *ProdRequest) (*ProdResponse, error) {

	return &ProdResponse{ProdStock: 20}, nil
}

```

编写server.go 服务端代码

grpcserver/server.go

```go
package main

import (
	"net"

	"google.golang.org/grpc"
	"gorpc.jtthink.co/services"
)

func main() {

	rpcServer := grpc.NewServer()

	services.RegisterProdServiceServer(rpcServer, new(services.ProdService))

	listen, _ := net.Listen("tcp", ":8082")
	rpcServer.Serve(listen)
}

```

执行go run server.go, 将服务端跑起来



## 创建客户端  (服务端既可以是客户端，客户端也可以是服务端)

新建一个项目grpcclient

在项目目录新建services，将服务端的.pb.go文件复制进来

然后安装gRPC包

```shell
go get -u google.golang.org/grpc
```

编写客户端实现

grpcclient/client.go

```go
package main

import (
	"context"
	"fmt"
	"log"

	"grpccli/services"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial(":8082", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	client := services.NewProdServiceClient(conn)
	prodRes, err := client.GetProdStock(context.Background(), &services.ProdRequest{
		ProdId: 10,
	})
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(prodRes.ProdStock)
}

```

执行 go run client.go，即可看到以下输出

```shell
➜  grpccli go run client.go
20

```

补更
提示：
执行 此命令 protoc –go_out=../services/ Prod.proto报错
```markdown
--go_out: protoc-gen-go: plugins are not supported; use 'protoc --go-grpc_out=...' to generate gRPC
```

模块github.com/golang/protobuf由google.golang.org/protobuf"取代
当前版本编译时，之前的方法protoc --go_out=plugins=grpc:. *.proto不再使用，转而用protoc --go_out=. --go-grpc_out=. ./Prod.proto代替



