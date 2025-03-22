---
layout: post
title: "微服务 – Spring Cloud – consul 安装、服务注册、服务发现"
date:    2023-06-22
tags: [SpringCloud]
comments: true
author: mazezen
---

## what is consul?

HashiCorp Consul is a service networking solution that enables teams to manage secure network connectivity between services and across on-prem and multi-cloud environments and runtimes. Consul offers service discovery, service mesh, traffic management, and automated updates to network infrastructure device. You can use these features individually or together in a single Consul deployment.

Consul 是一套开源的分布式服务发现和配置管理系统。由 HashiCorp 公司使用Go语言开发。

提供了微服务系统中的服务治理、配置中心、控制总线等。这些功能即可以单独使用，也可以一起使用构建全方位的服务网格。



## 功能

* 服务发现               提供http 和 dns 两种发现方式
* 监控检测              支持多方式， http、tcp、docker、shell脚本定制化
* KV存储                  key、value 的存储方式
* 多数据中心           consul支持多数据中心
* 可视化web界面



## 安装 Consul

1、拉取镜像

```shell
docker search consul
docker pull consul
```

2、运行镜像

```shell
docker run -d -p 8500:8500 --restart=always --name=consul consul:latest agent -server -bootstrap -ui -node=1 -client='0.0.0.0'

docker logs containId
```

- agent: 表示启动 Agent 进程。
- server：表示启动 Consul Server 模式
- client：表示启动 Consul Cilent 模式。
- bootstrap：表示这个节点是 Server-Leader ，每个数据中心只能运行一台服务器。技术角度上讲 Leader 是通过 Raft 算法选举的，但是集群第一次启动时需要一个引导 Leader，在引导群集后，建议不要使用此标志。
- ui：表示启动 Web UI 管理器，默认开放端口 8500，所以上面使用 Docker 命令把 8500 端口对外开放。
- node：节点的名称，集群中必须是唯一的，默认是该节点的主机名。
- client：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服务，如果你要对外提供服务改成0.0.0.0
- join：表示加入到某一个集群中去。 如：-json=192.168.0.11。

3、测试访问

`localhost:8500`
![](http://images.caixiaoxin.cn/consul.jpg)

## 服务注册

1、新建 子模块 payment 服务

2、引入consul pom依赖

```xml
<!-- 整合 consul客户端 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
```

3、application.yaml配置文件

```yaml
server:
  port: 8006

spring:
  application:
    name: consul-provier-payment

  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
```

4、主启动类 ConsulMain8006.java

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsulMain8006 {

    public static void main(String[] args) {
        SpringApplication.run(ConsulMain8006.class, args);
    }

}
```



4、编写业务控制体，提供一个接口，供消费者服务通过consul服务发现调用

PaymentController.java

```java
@RestController
public class PaymentController {

    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/cs")
    public String paymentConsul() {
        return "SpringCloud with consul: " + serverPort + "\t" + UUID.randomUUID().toString();
    }

}
```

5、启动 然后去consul注册中心查看 `consul-provier-payment` 服务是否注册进来
![](http://images.caixiaoxin.cn/consul-service1.jpg)



## 服务发现

采用还是RestTemplate

1、新建子模块order 服务，并把 order服务注册到 consul 服务中心 

2、order模块pom引入consul 依赖

```xml
<!-- 整合 consul客户端 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
```

2、配置文件 application.yaml

```yaml
server:
  port: 80

spring:
  application:
    name: consul-consumer-order

  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
```

3、RestTemplate配置类文件 ApplicationContextConfig.java

```java
@Configuration
public class ApplicationContextConfig {

    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

}

```

 4、业务接口 控制器 负责 通过consul 发现 payment服务,并调用payment服务中的 `/payment/cs` 接口

```java
@RestController
public class OrderController {

    private final static String PAYMENT_SERVICE = "http://consul-provier-payment";

    @Resource
    private RestTemplate restTemplate;

    @GetMapping(value = "/comsumer/payment/cs")
    public String paymentInfo() {
        String result = restTemplate.getForObject(PAYMENT_SERVICE + "/payment/cs", String.class);
        return result;
    }

}

```

5、主启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsulOrderMain80 {

    public static void main(String[] args) {
        SpringApplication.run(ConsulOrderMain80.class, args);
    }

}
```

6、运行order服务，会发现 order 服务已经注册到 consul注册中心

访问 `http://localhost/comsumer/payment/cs` 接口 实现了服务发现调用

![](http://images.caixiaoxin.cn/consul-service2.jpg)

![](http://images.caixiaoxin.cn/consul-res.jpg)