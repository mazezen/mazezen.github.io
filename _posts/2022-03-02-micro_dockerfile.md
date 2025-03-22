---
layout: post
title: "Go微服务九 go语言工程制作dockerfile，通过docker将项目或者独立服务部署到服务器"
date: 2022-03-02
tags: [Go]
comments: true
author: mazezen
---


#go语言工程制作dockerfile，通过docker将项目或者独立服务部署到服务器

## 前言

docker，<a href="https://so.csdn.net/so/search?q=kubernetes&spm=1001.2101.3001.7020" target="_blank" rel="noopener">[kubernetes]</a>，如何使用docker将go语言项目部署到docker容器，然后部署到服务器>[kubernetes]的天下，如何使用docker将go语言项目部署到docker容器，然后部署到服务器

## 编写一个服务（项目）

![image-20220228131804065](http://images.caixiaoxin.cn//image-20220228131804065.png)

## 制作dockerfile

dockerfile 制作的源镜像我们可以在 hub.docker.com 找到 [golang](https://so.csdn.net/so/search?q=golang&spm=1001.2101.3001.7020)官方提供的源镜像,我们采用golang:1.17.2-alpine和我本机go保持一致。

```dockerfile
FROM golang:1.17.2-alpine

# 环境
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    GOPROXY="https://goproxy.io"

# 代码目录
WORKDIR /root/go-web/demo

ADD . /root/go-web/demo

# 编译二进制文件
RUN go build .

# 声明服务端口
EXPOSE 7804

# 启动容器时运行的命令
CMD ["./demo"]

```

意dockerfile文件名称必须是Dockerfile，其文件必须再工程目录下。

## 将项目代码上传到服务

```shell
scp -r demo/ root@ip:/root/go-web/demo
```

## 执行dockerfile

```shell
docker build -t 名称 .
```

```markdown
Sending build context to Docker daemon  133.1kB
Step 1/7 : FROM golang:1.17.2-alpine
 ---> 35cd8c8897b1
Step 2/7 : ENV GO111MODULE=on     CGO_ENABLED=0     GOOS=linux     GOARCH=amd64     GOPROXY="https://goproxy.io"
 ---> Using cache
 ---> 72365ed772a3
Step 3/7 : WORKDIR /root/go-omp/omp-cashier
 ---> Using cache
 ---> c7726f0d2d63
Step 4/7 : ADD . /root/go-omp/omp-cashier
 ---> 6acd4399654a
Step 5/7 : RUN go build .
 ---> Running in 114ad88252ea
go: downloading github.com/labstack/echo v3.3.10+incompatible
go: downloading github.com/labstack/gommon v0.3.1
go: downloading github.com/mattn/go-colorable v0.1.12
go: downloading github.com/mattn/go-isatty v0.0.14
go: downloading github.com/valyala/fasttemplate v1.2.1
go: downloading golang.org/x/sys v0.0.0-20220224120231-95c6836cb0e7
go: downloading golang.org/x/crypto v0.0.0-20220214200702-86341886e292
go: downloading github.com/valyala/bytebufferpool v1.0.0
go: downloading golang.org/x/net v0.0.0-20220127200216-cd36cc0744dd
go: downloading golang.org/x/text v0.3.7
Removing intermediate container 114ad88252ea
 ---> 335dd128fd61
Step 6/7 : EXPOSE 7804
 ---> Running in 2c712608fdef
Removing intermediate container 2c712608fdef
 ---> 8dc2ae5a1ea2
Step 7/7 : CMD ["./demo"]
 ---> Running in f4363e7b0856
```

制作完成后，我们即可以用**docker images**查看制作好的镜像。

## 运行创建好的镜像

```shell
docker run --name demo -p 7804:7804 -d demo
```

## 访问 ip:port/api即可