---
layout: post
title: "Go微服务七 同一端口监听grpc和http服务"
date: 2022-02-26
tags: [Go]
comments: true
author: mazezen
---

在上一篇文章中，基于TLS认证，使用grpc-gateway提供了grpc和http服务。两个服务启动了两个端口。

可能还有同一个端口启用两个服务的业务。rpc是rpc服务,另外是api接口。其实一个链接可以是rpc或者http,但不能同时是两者

### 一 在项目目录下执行

```shell
go get -u github.com/soheilhy/cmux
```

### 二 打开main.go 

```go
//RunTCPServer
func RunTCPServer() (net.Listener, error) {
	return net.Listen("tcp", ":9802")
}

//RunGrpcServer
func RunGrpcServer() *grpc.Server {
	s := grpc.NewServer()
	pbfiles.RegisterRpcOrderServiceServer(s, new(omp_order_service.RpcOrderService))
	return s
}

//RunHttpServer
func RunHttpServer() *http.Server {
	mux := http.NewServeMux()
  serveMux.HandleFunc("/ping", func(w http.ResponseWriter, r *http.Request) {
		_, _ = w.Write([]byte(`pong`))
	})
	return &http.Server{
		Addr:    ":9802",
		Handler: mux,
	}
}

func main() {
	l, err := router.RunTCPServer()
	if err != nil {
		logger.OmpLog.Fatal("Run tcp server err: %v", err)
	}

	m := cmux.New(l)
	grpcL := m.MatchWithWriters(cmux.HTTP2MatchHeaderFieldPrefixSendSettings("content-type", "application/grpc"))
	httpL := m.Match(cmux.HTTP1Fast())

	grpcS := router.RunGrpcServer()
	httpS := router.RunHttpServer()

	go grpcS.Serve(grpcL)
	go httpS.Serve(httpL)

	err = m.Serve()
	if err != nil {
		logger.OmpLog.Fatal("Run server failed, %v", err)
	}
}
```

基于 cmux 实现的同端口支持多协议已经完成了，你需要重新启动服务进行验证，确保 grpcurl 工具和利用 curl 调用 HTTP/1.1 接口响应正常

