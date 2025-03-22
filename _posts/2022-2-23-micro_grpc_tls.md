---
layout: post
title: "Go微服务五 Go - gRPC 中的TLS认证"
date: 2022-2-23
tags: [Go]
comments: true
author: mazezen
---

## 介绍

在上一篇中简单介绍了 在go中gRPC的使用，但是无签名的认证。这一篇简单介绍生成cert进行TLS认证的调用

代码较上一篇改动不大。包含使用openssl生成ca证书



## 证书生成流程

1. 新增ca.conf

```m
[ req ]
default_bits       = 4096
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
countryName                 = CN
countryName_default         = CN
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = ZheJiang
localityName                = Locality Name (eg, city)
localityName_default        = HangZhou
organizationName            = Organization Name (eg, company)
organizationName_default    = cc
commonName                  = cc.io
commonName_max              = 64
commonName_default          = cc.io
```

2. 生成CA私钥

```shell
openssl genrsa -out ca.key 4096
```

3. 生成CA证书

```shell
openssl req -new -x509 -days 365 -subj "/C=CN/L=HangZhou/O=cc/CN=cc.io" -key ca.key -out ca.crt -config ca.conf
```

4. 生成服务端证书

```m
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
countryName                 = Country Name (2 letter code)
countryName_default         = CN
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = ZheJiang
localityName                = Locality Name (eg, city)
localityName_default        = HangZhou
organizationName            = Organization Name (eg, company)
organizationName_default    = cc
commonName                  = CommonName (e.g. server FQDN or YOUR name)
commonName_max              = 64
commonName_default          = cc.io  (自定义,客户端需要此字段做匹配)
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1   = cc.io
IP      = 127.0.0.1
```

5. 生成公钥

```shell
openssl genrsa -out server.key 2048
```

6. 生成CSR

```shell
openssl req -new  -subj "/C=CN/L=HangZhou/O=cc/CN=cc.io" -key server.key -out server.csr -config \ server.conf
```

7. 基于CA签发证书

```shell
openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -in server.csr -out \ server.crt -extensions req_ext -extfile server.conf
```

8. 生成客户端证书

```shell
openssl genrsa -out client.key 2048
```

9. 生成CSR

```shell
openssl req -new -subj "/C=CN/L=HangZhou/O=cc/CN=cc.io" -key client.key -out client.csr
```

10. 基于CA签发证书

```shell
openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -in client.csr -out \ client.crt
```



![image-20220305141413219](http://images.caixiaoxin.cn//image-20220305141413219.png)

11. 在grpcserver项目和grpcclient项目下分别创建keys目录，将server和client的证书和ca的证书复制进去

    代码实现:

    服务端:

    ```go
    package main
    
    import (
    	"crypto/tls"
    	"crypto/x509"
    	"fmt"
    	"io/ioutil"
    	"net"
    
    	"google.golang.org/grpc/credentials"
    
    	"google.golang.org/grpc"
    	"gorpc.jtthink.co/services"
    )
    
    func main() {
    
    	// 公钥中读取解析公钥、私钥
    	cert, err := tls.LoadX509KeyPair("keys/server.crt", "keys/server.key")
    	if err != nil {
    		fmt.Println("LoadX509KeyPair error", err)
    		return
    	}
    
    	// 创建证书池
    	certPool := x509.NewCertPool()
    	ca, err := ioutil.ReadFile("keys/ca.crt")
    	if err != nil {
    		fmt.Println("read ca pem error ", err)
    		return
    	}
    	
    	// 解析证书
    	if ok := certPool.AppendCertsFromPEM(ca); !ok {
    		fmt.Println("AppendCertsFromPEM error ")
    		return
    	}
    	
    	cred := credentials.NewTLS(&tls.Config{
    		Certificates: []tls.Certificate{cert},
    		ClientAuth:   tls.RequireAndVerifyClientCert,
    		ClientCAs:    certPool,
    	})
    
    	rpcServer := grpc.NewServer(grpc.Creds(cred))
    
    	services.RegisterProdServiceServer(rpcServer, new(services.ProdService))
    
    	listen, _ := net.Listen("tcp", ":8082")
    
    	fmt.Println("服务启动成功...")
    	rpcServer.Serve(listen)
    }
    
    ```

    go run server.go

    结果：

    ```shell
    ➜ go run server.go
    服务启动成功...
    
    
    
    ```

    

    

    客户端代码实现

    ```go
    package main
    
    import (
    	"context"
    	"crypto/tls"
    	"crypto/x509"
    	"fmt"
    	"io/ioutil"
    	"log"
    
    	"google.golang.org/grpc/credentials"
    
    	"grpccli/services"
    
    	"google.golang.org/grpc"
    )
    
    func main() {
    	pair, err := tls.LoadX509KeyPair("keys/client.crt", "keys/client.key")
    	if err != nil {
    		fmt.Println("LoadX509KeyPair error ", err)
    		return
    	}
    	certPool := x509.NewCertPool()
    	ca, err := ioutil.ReadFile("keys/ca.crt")
    	if err != nil {
    		fmt.Println("ReadFile ca.crt error ", err)
    		return
    	}
    
    	if ok := certPool.AppendCertsFromPEM(ca); !ok {
    		fmt.Println("certPool.AppendCertsFromPEM error ")
    		return
    	}
    
    	cred := credentials.NewTLS(&tls.Config{
    		Certificates: []tls.Certificate{pair},
    		ServerName:   "cc.io",
    		RootCAs:      certPool,
    	})
    
    	conn, err := grpc.Dial(":8082", grpc.WithTransportCredentials(cred))
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

    go run client.go

    结果：

    ```shell
    ➜ go run client.go                              
    20
    
    ```

## 上面代码是基于tcp实现的服务端,基于http实现如下

server.go

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"net/http"

	"google.golang.org/grpc/credentials"

	"google.golang.org/grpc"
	"gorpc.jtthink.co/services"
)

func main() {

	// 公钥中读取解析公钥、私钥
	cert, err := tls.LoadX509KeyPair("keys/server.crt", "keys/server.key")
	if err != nil {
		fmt.Println("LoadX509KeyPair error", err)
		return
	}

	// 创建证书池
	certPool := x509.NewCertPool()
	ca, err := ioutil.ReadFile("keys/ca.crt")
	if err != nil {
		fmt.Println("read ca pem error ", err)
		return
	}

	// 解析证书
	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		fmt.Println("AppendCertsFromPEM error ")
		return
	}

	cred := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		ClientAuth:   tls.RequireAndVerifyClientCert,
		ClientCAs:    certPool,
	})

	rpcServer := grpc.NewServer(grpc.Creds(cred))

	services.RegisterProdServiceServer(rpcServer, new(services.ProdService))

	//listen, _ := net.Listen("tcp", ":8082")

	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Println(r.Proto)
		fmt.Println(r.Header)
		fmt.Println(r)
		rpcServer.ServeHTTP(w, r)
	})

	httpServer := &http.Server{
		Addr:    ":8083",
		Handler: mux,
	}

	httpServer.ListenAndServeTLS("keys/server.crt", "keys/server.key")

	//fmt.Println("服务启动成功...")
	//rpcServer.Serve(listen)
}

```

client.go代码不变

执行

```shell
go run server.go
go run client.go
```

结果

```shell
➜ go run client.go
```