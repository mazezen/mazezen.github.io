---
layout: post
title: "Go微服务六  Go - gRPC - TLS 使用rpc-gatway 不同端口同时提供rpc和http服务"
date: 2022-02-25
tags: [Go]
comments: true
author: mazezen
---

为什么要这么做： 

不管时内部另外一个服务还是外部第三方服务，如果调用者也使用了rpc，可以调用写好的服务端。

如果调用者没有使用rpc而使用了http RESTFUL API ，那就要使用rpc-gatway提供http服务了

简而言之：一个商品详情服务接口即可以提供rpc也可以支持http RESUTFUL API (rpc和api不同的启动端口)



## 一 安装

```shell
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
```



## 二 COPY

拷贝google到服务端项目目录下的pbfiles中



## 三 修改.proto文件

```protobuf
syntax = "proto3";

package services;

import "google/api/annotations.proto"; 

option go_package = "../services";

message ProdRequest {
  int32 prod_id = 1;
}

message ProdResponse {
  int32 prod_stock = 1;
}

service ProdService {
  rpc GetProdStock (ProdRequest) returns (ProdResponse) {
    option (google.api.http) = {
      get: "/v1/prod/{prod_id}"
    };
  }
}
```

执行以下命令：

```shell
protoc --go_out=plugins=grpc:../services Prod.proto 
protoc --grpc-gateway_out=logtostderr=true:../services Prod.proto  # 生成一个 xxx.pb.gw.go文件
```

## 四 将客户端项目下的client.key和client.crt复制到本项目的keys目录下

## 五 封装服务端和客户端证书配置

helper/CertHelper.go

```go
package helper

import (
	"crypto/tls"
	"crypto/x509"
	"io/ioutil"
	"log"

	"google.golang.org/grpc/credentials"
)

//GetServerCred 获取服务端证书配置
func GetServerCred() credentials.TransportCredentials {
	// 公钥中读取解析公钥、私钥
	cert, err := tls.LoadX509KeyPair("keys/server.crt", "keys/server.key")
	if err != nil {
		log.Fatal("LoadX509KeyPair error", err)
	}
	// 创建证书池
	certPool := x509.NewCertPool()
	ca, err := ioutil.ReadFile("keys/ca.crt")
	if err != nil {
		log.Fatal("read ca pem error ", err)
	}

	// 解析证书
	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		log.Fatal("AppendCertsFromPEM error ")
	}

	cred := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		ClientAuth:   tls.RequireAndVerifyClientCert,
		ClientCAs:    certPool,
	})

	return cred
}

//GetClientCred 获取客户端证书配置
func GetClientCred() credentials.TransportCredentials {
	pair, err := tls.LoadX509KeyPair("keys/client.crt", "keys/client.key")
	if err != nil {
		log.Fatal("LoadX509KeyPair error ", err)
	}
	certPool := x509.NewCertPool()
	ca, err := ioutil.ReadFile("keys/ca.crt")
	if err != nil {
		log.Fatal("ReadFile ca.crt error ", err)
	}

	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		log.Fatal("certPool.AppendCertsFromPEM error ")
	}

	cred := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{pair},
		ServerName:   "cc.io",
		RootCAs:      certPool,
	})

	return cred
}

```

GetClientCred校验客户端http RESTFUL API 使用

server.go

```go
package main

import (
	"fmt"
	"net/http"

	"gorpc.jtthink.co/helper"

	"google.golang.org/grpc"
	"gorpc.jtthink.co/services"
)

func main() {

	rpcServer := grpc.NewServer(grpc.Creds(helper.GetServerCred()))

	services.RegisterProdServiceServer(rpcServer, new(services.ProdService))


	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Println(r.Proto)
		fmt.Println(r.Header)
		fmt.Println(r)
		rpcServer.ServeHTTP(w, r)
	})

	httpServer := &http.Server{
		Addr:    ":8081",
		Handler: mux,
	}

	httpServer.ListenAndServeTLS("keys/server.crt", "keys/server.key")
}

```

httpServer.go

```go
package main

import (
	"context"
	"log"
	"net/http"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
	"gorpc.jtthink.co/helper"
	"gorpc.jtthink.co/services"
)

func main() {

	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)

	defer cancel()

	mux := runtime.NewServeMux()

	//客户端请求时验证
	opts := []grpc.DialOption{grpc.WithTransportCredentials(helper.GetClientCred())}
	err := services.RegisterProdServiceHandlerFromEndpoint(
		ctx,
		mux,
		"localhost:8081",
		opts,
		)

	if err != nil {
		log.Fatal(err)
	}

	httpServer := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}
	httpServer.ListenAndServe()
}

```

依次启动

```shell
go run server.go
go run client.go # grpccli项目下
```

结果

```shell
➜ go run client.go
20

```

启动httpServer

```shell
go run httpServer.go
```

浏览器访问http://localhost:8080/v1/prod/123

![image-20220305200710760](http://images.caixiaoxin.cn//image-20220305200710760.png)





### 问题1:  import "google/api/annotations.proto"; 引入爆红 不影响使用，暂为解决idea爆红的原因 



### 为题2:

 cannot use mux (type *"github.com/grpc-ecosystem/grpc-gateway/runtime".ServeMux) as type *"github.com/grpc-ecosystem/grpc-gateway/v2/runtime".ServeMux in argument to services.RegisterProdServiceHandlerFromEndpoint

向services.RegisterProdServiceHandlerFromEndpoint传的参数一摸一样，看报错信息分析一下就能知道问题的原因。引入的版本不一致。

将github.com/grpc-ecosystem/grpc-gateway/runtime改为github.com/grpc-ecosystem/grpc-gateway/v2/runtime即可
