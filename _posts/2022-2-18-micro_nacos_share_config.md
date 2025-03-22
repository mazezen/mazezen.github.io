---
layout: post
title: "Go微服务二 Go+nacos 实现配置共享"
date: 2022-2-18
tags: [Go]
comments: true
author: mazezen
---

发现、配置和管理微服务

### 特性

- **服务发现和服务健康监测**、
- **动态配置服务**
- **动态 DNS 服务**
- **服务及其元数据管理**

## 搭建Nacos

### 安装

####  压缩包方式

<a href="https://github.com/alibaba/nacos/releases/download/2.0.4/nacos-server-2.0.4.tar.gz" target="_blank" rel="noopener">[nacos2.0.4 github下载地址]</a>

```shell
➜ mkdir nacos-server
➜ cd Downloads && mv nacos-server-2.0.4.tar.gz /Users/cc/nacos-server
➜ cd /Users/cc/nacos-server && tar -zxvf nacos-server-2.0.4.tar.gz
➜ mv nacos nacos204
➜ cd nacos204/bin	
➜ ./startup.sh -m standalone  # standalone代表着单机模式运行，非集群模式
➜ ps aux | grep nacos-server
➜ ./shutdown.sh # 关闭nacos服务
```

#### Docker 方式 - 推荐

```shell
docker run --name nacos-standalone -e MODE=standalone -e JVM_XMS=512m -e JVM_MAX=512m -e JVM_XMN=256m -p 8848:8848 -d nacos/nacos-server:latest
```

### 使用

#### 1. 登陆

浏览器访问： 127.0.0.1:8848/nacos/#/login

账号密码： nacos/nacos

![image-20220225132043056](http://images.caixiaoxin.cn//image-20220225132043056.png)

#### 2. 配置 - 新建配置

![image-20220225132607821](http://images.caixiaoxin.cn//image-20220225132607821.png)

#### 3. 命名空间 

可以新建命名空间，也可以使用默认的public

![image-20220225132903966](http://images.caixiaoxin.cn//image-20220225132903966.png)

#### 4. 组 - 隔离

场景：把不同微服务的开发、测试、生成配置文件隔离（相同名字的配置文件，属于不同的组）

![image-20220225145636002](http://images.caixiaoxin.cn//image-20220225145636002.png)

#### 5. Data ID - 配置集

一个配置集就是一个配置文件, 还可以将db,server内容等配置分开，实现更灵活的管理



## Go集成nacos

<a href="https://github.com/nacos-group/nacos-sdk-go/blob/master/README_CN.md" target="_blank" rel="noopener">https://github.com/nacos-group/nacos-sdk-go/blob/master/README_CN.md</a>

### 1. 安装

```shell
go get -u github.com/nacos-group/nacos-sdk-go
```

Demo:

```shell
package main

import (
	"fmt"
	"time"

	"github.com/nacos-group/nacos-sdk-go/clients"
	"github.com/nacos-group/nacos-sdk-go/common/constant"
	"github.com/nacos-group/nacos-sdk-go/vo"
)

func main() {
	// Note：我们可以配置多个ServerConfig，客户端会对这些服务端做轮询请求

	clientConfig := constant.ClientConfig{
		TimeoutMs:           5000,    // 请求Nacos服务端的超时时间，默认是10000ms
		NamespaceId:         "",      // 如果需要支持多namespace，我们可以场景多个client,它们有不同的NamespaceId。当namespace是public时，此处填空字符串。
		CacheDir:            "cache", // 缓存service信息的目录，默认是当前运行目录
		NotLoadCacheAtStart: false,   // 在启动的时候不读取缓存在CacheDir的service信息
		LogDir:              "log",   // 日志存储路径
		LogLevel:            "debug", // 日志默认级别，值必须是：debug,info,warn,error，默认值是info
	}

	serverConfigs := []constant.ServerConfig{
		{
			IpAddr:      "127.0.0.1",
			ContextPath: "/nacos",
			Port:        8848,
			Scheme:      "http",
		},
	}

	// 创建动态配置客户端的另一种方式 (推荐)
	// 创建动态配置客户端的另一种方式 (推荐)
	configClient, err := clients.NewConfigClient(
		vo.NacosClientParam{
			ClientConfig:  &clientConfig,
			ServerConfigs: serverConfigs,
		},
	)
	if err != nil {
		panic(err)
	}

	content, err := configClient.GetConfig(vo.ConfigParam{
		DataId: "config.yaml",
		Group:  "dev", // 开发环境配置文件
	})

	if err != nil {
		panic(err)
	}
	fmt.Println(content)

	// 监听配置变化
	configClient.ListenConfig(vo.ConfigParam{
		DataId: "config.yaml",
		Group:  "dev",
		OnChange: func(namespace, group, dataId, data string) {
			fmt.Println("配置文件发生了变化...")
			fmt.Println("group:" + group + ", dataId:" + dataId + ", data:" + data)
		},
	})

	time.Sleep(300 * time.Second)
}

```

![image-20220225151429926](http://images.caixiaoxin.cn//image-20220225151429926.png)