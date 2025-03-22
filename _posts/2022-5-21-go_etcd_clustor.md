---
layout: post
title: "Go操作etcd集群"
date: 2022-5-21
tags: [Go]
comments: true
author: mazezen
---

docker搭建etcd集群
### yml
```yaml
version: "3.0"
networks:
  etcd-net:
    driver: bridge

volumes:
  etcd1_data:
    driver: local
  etcd2_data:
    driver: local
  etcd3_data:
    driver: local

services:
  etcd1:
    image: bitnami/etcd:latest
    container_name: etcd1
    restart: always
    networks:
      - etcd-net
    ports:
      - "30000:2379"
      - "30001:2380"
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes                        # 运训不用密码登陆
      - ETCD_NAME=etcd1                                      # 名字
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd1:2380   # 列出这个成员的伙伴 URL 以便通告给集群的其他成员
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380            # 用于监听伙伴通讯的URL列表
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379         # 用于监听客户端通讯的URL列表
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd1:2379        # 列出这个成员的客户端URL，通告给集群中的其他成员
      - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster             # 在启动期间用于 etcd 集群的初始化集群记号
      - ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 # 为启动初始化集群配置
      - ETCD_INITIAL_CLUSTER_STATE=new                      # 初始化集群状态
    volumes:
      - "./data/etcd1_data:/bitnami/etcd"                           # 挂在的数据卷

  etcd2:
    image: bitnami/etcd:latest
    container_name: etcd2
    restart: always
    networks:
      - etcd-net
    ports:
      - "30002:2379"
      - "30003:2380"
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_NAME=etcd2
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd2:2380
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd2:2379
      - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
      - ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - ETCD_INITIAL_CLUSTER_STATE=new
    volumes:
      - "./data/etcd2_data:/bitnami/etcd"

  etcd3:
    image: bitnami/etcd:latest
    container_name: etcd3
    restart: always
    networks:
      - etcd-net
    ports:
      - "30004:2379"
      - "30005:2380"
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_NAME=etcd3
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd3:2380
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd3:2379
      - ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
      - ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      - ETCD_INITIAL_CLUSTER_STATE=new
    volumes:
      - "./data/etcd3_data:/bitnami/etcd"

```

```shell
docker-compose up -d
```

### 测试
etcd1 写入
```shell
docker exec -it etcd1 bash

I have no name!@77912e04a985:/opt/bitnami/etcd$ etcdctl put name "test"
OK
```

etcd2 读取
```shell
docker exec -it etcd2 bash
I have no name!@5972f7c68c5e:/opt/bitnami/etcd$ etcdctl get name
name
test
```




### 安装
<a href="https://github.com/etcd-io/etcd" target="_blank" rel="noopener">etcd/client/v3</a>
```shell
go get go.etcd.io/etcd/client/v3
```

```go
func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"http://127.0.0.1:30000", "http://127.0.0.1:30002", "http://127.0.0.1:30004"},
		DialTimeout: 5 * time.Second,
	})

	if err != nil {
		fmt.Printf("connect to etcd failed, err: %v\n", err)
		return
	}

	defer cli.Close()
	fmt.Println("connect to etcd success...")

	go listen(cli)

	put(cli)
	get(cli)
	update(cli)
	del(cli)
}

func put(cli *clientv3.Client) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	_, err := cli.Put(ctx, "name", "test")
	cancel()
	if err != nil {
		fmt.Printf("put to etcd failed, err:%v\n", err)
		return
	}
}

func get(cli *clientv3.Client) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	res, err := cli.Get(ctx, "name")
	cancel()
	if err != nil {
		fmt.Printf("get to etcd failed, err:%v\n", err)
		return
	}
	for _, v := range res.Kvs {
		fmt.Printf("type: %s key: %s value: %s\n", "GET", v.Key, v.Value)
	}
}

func update(cli *clientv3.Client) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	_, err := cli.Put(ctx, "name", "test2")
	cancel()
	if err != nil {
		fmt.Printf("put to etcd failed, err:%v\n", err)
		return
	}
}

func del(cli *clientv3.Client) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	_, err := cli.Delete(ctx, "name")
	cancel()
	if err != nil {
		fmt.Printf("delete from etcd failed, err:%v\n", err)
		return
	}
}

func listen(cli *clientv3.Client) {
	wat := cli.Watch(context.Background(), "name")
	for w := range wat {
		for _, v := range w.Events {
			fmt.Printf("type: %s key: %s value: %s", v.Type, v.Kv.Key, v.Kv.Value)
		}
	}
	fmt.Println("out")
}

```