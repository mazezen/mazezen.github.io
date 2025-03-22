---
layout: post
title: "Go微服务十二  Go使用nacos 注册服务，服务发现"
date: 2022-03-12
tags: [Go]
comments: true
author: mazezen
---

Go使用阿里nacos管理服务注册和发现，思路Demo

### 连接nacos

```go
func InitNacos() *nacosRF {
	//return &nacosRF{}
	d := &nacosRF{}
	clientConfig := constant.ClientConfig{
		TimeoutMs:           5000,
		NotLoadCacheAtStart: true,
		LogDir:              "./log",
		CacheDir:            "./cache",
		LogLevel:            "debug",
	}

	serverConfigs := []constant.ServerConfig{
		{
			IpAddr:      "127.0.0.1",
			ContextPath: "/nacos",
			Port:        8848,
			Scheme:      "http",
		},
	}

	var err error
	d.client, err = clients.NewNamingClient(
		vo.NacosClientParam{
			ClientConfig:  &clientConfig,
			ServerConfigs: serverConfigs,
		},
	)
	if err != nil {
		panic(err)
	}

	NacosInstance = d
	return d
}
```

### 注册服务

```go
func (this *nacosRF) Register(serverName string, port int) {
	ok, err := InitNacos().client.RegisterInstance(vo.RegisterInstanceParam{
		Ip:          utils.GetIp(),
		Port:        uint64(port),
		Weight:      10,
		Enable:      true,
		Healthy:     true,
		Metadata:    map[string]string{},
		ClusterName: "DEMO_SERVER",
		ServiceName: serverName,
		GroupName:   "DEMO_SERVER_GROUP",
		Ephemeral:   true,
	})
	if err != nil {
		panic(err)
	}
	if !ok {
		logger.Info("注册本服务发生错误")
		panic(errors.New("注册本服务发生错误"))
	}
}

```

```go
nacosRF.NacosInstance.Register("demo", 8300)
```

### 服务发现

```go
//FindInstance 获取某个服务器连接信息
func (this *nacosRF) FindInstance(serverName string) (*ServerInfo, error) {
	ins, err := InitNacos().client.SelectOneHealthyInstance(vo.SelectOneHealthInstanceParam{
		Clusters:    []string{"DEMO_SERVER"},
		ServiceName: serverName,
		GroupName:   "DEMO_SERVER_GROUP",
	})
	if err != nil {
		return nil, err
	}
	return &ServerInfo{
		Ip:         ins.Ip,
		Port:       ins.Port,
		ServerName: ins.ServiceName,
	}, nil
}
```

```go
func GetDemoOrderService(serviceName string) pb.RpcDemoServiceClient {
	info, err := nacosRF.NacosInstance.FindInstance(serviceName)
	if err != nil {
		return nil
	}
	c, err := grpc.Dial(fmt.Sprintf("%s:%d", info.Ip, info.Port), grpc.WithInsecure())
	if err != nil {
		return nil
	}
	p := pb.NewRpcDemoServiceClient(c)
	return p
}
```

### 客户端调用

```go
func main() {

	demoClient := service.GetDemoOrderService("demo")
	result, err := demoClient.GetUserInfo(context.Background(), &pb.GetUserInfoRequest{
		Id: 10,
	})
	//
	if err != nil {
		panic(err)
	}

	fmt.Println(result)
}
```

