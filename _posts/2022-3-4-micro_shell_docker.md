---
layout: post
title: "Go微服务十 编写shell脚本启动docker部署的go项目"
date: 2022-3-4
tags: [Go]
comments: true
author: mazezen
---


上一篇，通过编写Dockerfile部署go项目。存在一个不方便的地方。每次将新的打包好的go项目传到服务上之后都需要先停止docker,删除docker 容器 , 删除docker 镜像，再执行dokcer build 和docker run 步骤台繁琐.

这一篇通过编写shell脚本一键执行命令完成上述所有步骤实现自动重启docker部署的goladn项目

Demo

在go项目下，编写一个start.sh文件（名字随便取，后缀为.sh）
```shell
#! /bin/bash

if [ ! -f 'main' ]; then
  echo 文件不存在! 待添加的安装包: 'main'
  exit
fi

echo "demo-go..."
sleep 3
docker stop demo-go

sleep 2
docker rm demo-go

docker rmi demo-go
echo ""

echo "deomo-go packaging..."
sleep 3
docker build -t demo-go .
echo ""

echo "demo-go running..."
sleep 3
docker run --name demo-go \
  -p 9801:9801 \
  -d demo-go

docker logs -f demo-go | sed '/Started CashierApplication/q'

echo ""
```