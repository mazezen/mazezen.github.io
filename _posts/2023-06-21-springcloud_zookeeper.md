---
layout: post
title: "微服务 – Spring Cloud –zookeeper安装以及服务注册、发现"
date:    2023-06-21
tags: [SpringCloud]
comments: true
author: mazezen
---


## zookeeper 简介

ZooKeeper是一个集中式服务，用于维护配置信息、命名、提供分布式同步、提供组服务. 支持高度可靠的分布式协调.

### zookeeper 数据模型和分层命名空间

zookeeper 数据模型: 其实就是用来存储和处理数据的。类似于数据库系统。不过 zookeeper 的数据模型更像电脑中的文件系统，有一个根文件夹（固定的根节点 / ），下面有很多字文件夹（可以在根节点创建多个子节点，支持逐级创建）

![](http://images.caixiaoxin.cn/zookeeper.png)

### zookeeper 节点和临时节点

与标准文件系统不同，ZooKeeper 命名空间中的每个节点都可以具有与其关联的数据以及子节点。这就像拥有一个允许文件也成为目录的文件系统。（ZooKeeper 被设计用来存储协调数据：状态信息、配置、位置信息等，因此每个节点存储的数据通常很小，在字节到千字节范围内。）我们使用术语 znode 来*明确*我们正在谈论 ZooKeeper 数据节点。

Znode 维护一个统计结构，其中包括数据更改的版本号、ACL 更改和时间戳，以允许缓存验证和协调更新。每次 znode 的数据更改时，版本号都会增加。例如，每当客户端检索数据时，它也会收到数据的版本。

存储在命名空间中每个 znode 的数据是原子读取和写入的。读取获取与 znode 关联的所有数据字节，写入替换所有数据。每个节点都有一个访问控制列表 (ACL)，用于限制谁可以做什么。

ZooKeeper 也有临时节点的概念。只要创建 znode 的会话处于活动状态，这些 znode 就会存在。当会话结束时，znode 将被删除。

### zookeeper API

* *create*：在树中的某个位置创建一个节点
* *删除*：删除一个节点
* *存在*：测试某个位置是否存在节点
* *获取数据*：从节点读取数据
* *设置数据*：将数据写入节点
* *get children*：检索节点的子节点列表
* *sync*：等待数据传播

## zookeeper 安装以及可视化界面安装

1、**创建 zookeeper 安装目录，保存 zookeeper 相关内容**

```shell
mkdir $HOME/docker/zookeeper
```

2、**拉取 zookeeper 镜像**

```shell
docker search zookeeper    
docker pull zookeeper:latest 
```

3、**创建zookeeper 数据挂在目录**

```shell
mkdir -p $HOME/docker/zookeeper/data
mkdir -p $HOME/docker/zookeeper/conf
mkdir -p $HOME/docker/zookeeper/logs
```

4、**安装，并挂载数据卷到宿主机**

```shell
docker run -d \
--name zookeeper \
--privileged=true \
-p 2181:2181 \
--restart=always \
-v $HOME/docker/zookeeper/data:/data \
-v $HOME/docker/zookeeper/conf:/conf \
-v $HOME/docker/zookeeper/logs:/datalog \
zookeeper
```

5、**查看安装结果**

```shell
docker logs containerId

docker exec -it zookeeper /bin/bash ./bin/zkServer.sh status

docker exec -it zookeeper zkCli.sh
```

6、安装可视化界面

<a href="https://github.com/vran-dev/PrettyZoo/releases" target="_blank" rel="noopener">PrettyZoo</a>

下载自己对应的版本即可 

我的为 mac 版本。如果下载下来 显示文件以损坏。

解决：

```shell
sudo spctl --master-disable
sudo xattr -rc /Applications/prettyZoo.app
```

参考： https://github.com/vran-dev/PrettyZoo/blob/master/README_CN.md

![](http://images.caixiaoxin.cn/prettyzoo.jpg)

## spring cloud 注册服务到zookeeper

1、 新建子模块 cloud-payment

2、加入pom依赖

```xml
<!-- 整合 zookeeper客户端 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>

```

3、配置文件 application.yaml

```yaml
spring:
  application:
    name: cloud-payment-service
  cloud:
    zookeeper:
      connect-string: 127.0.0.1:2181
```

4、启动类

该注解用于向consul或者zookeeper注册中心注册服务使用

@EnableDiscoveryClient  

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentMain8004 {

    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8004.class, args);
    }

}
```

5、查看服务是否注册进去

![](http://images.caixiaoxin.cn/zookeeper-service.jpg)



6、自测

编写接口

PaymentController.java

```java
@RestController
public class PaymentController {

    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/zk")
    public String paymentZk() {
        return "SpringCloud with zookeeper: " + serverPort + "\t" + UUID.randomUUID().toString();
    }

}
```

`访问 localhost:8004/payment/zk`

SpringCloud with zookeeper: 8004 e40a3dc5-0599-40de-9405-c588d24c98fe

## Zookeeper 服务发现调用



新建一个子模块，order 服务。使用order 服务调用 payment服务的 /payment/zk 接口

pom.xml 引入 zookeeper依赖

```xml
<!-- 整合 zookeeper客户端 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
        </dependency>

```

配置文件 application.yaml

```yaml
server:
  port: 80

spring:
  application:
    name: cloud-consumenr-order-service

  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: org.gjt.mm.mysql.Driver
    url: jdbc:mysql://localhost:3306/db2023?useUnicde=true&characterEncoding=utf-8&useSSL=false
    username: root
    password: 123456

  cloud:
    zookeeper:
      connect-string: 127.0.0.1:2181
```

启动类 ConsumnerZkOderMain80.java

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsumnerZkOderMain80 {

    public static void main(String[] args) {
        SpringApplication.run(ConsumnerZkOderMain80.class, args);
    }

}
```

配置文件类 ApplicationContextBean.java

```java
@Configuration
public class ApplicationContextBean {

    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }


}
```

具体服务调用控制器 OderZkController.java

```java
@RestController
@Slf4j
public class OderZkController {

   // cloud-payment-service 为 payment 服务在 zookeeper service 中的名称 
    public static final String INVOKE__URL = "http://cloud-payment-service";

    @Resource
    private RestTemplate restTemplate;

    @RequestMapping(value = "/consumer/payment/zk")
    public String paymentInfo() {
        String result = restTemplate.getForObject(INVOKE__URL + "/payment/zk", String.class);

        return result;
    }

}

```

访问 order 服务的 `/consumer/payment/zk` 在 order 服务的接口中，通过RestTemplate  来调用payement服务中的 `/payment/zk`接口

`http://localhost/consumer/payment/zk`

```html
SpringCloud with zookeeper: 8004 39f4e027-ca57-40c5-a9f6-a26e1ca7e342
```



