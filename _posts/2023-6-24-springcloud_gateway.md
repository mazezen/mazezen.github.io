---
layout: post
title: "微服务 – Spring Cloud – Gateway"
date:    2024-6-24
tags: [SpringCloud]
comments: true
author: mazezen
---


Api 网关 （Api Gateway ）

> 微服务可能分布在不同的主机上，这样有许多缺点：前端需要硬编码调用不同地址的微服务很麻烦；存在跨域访问的问题；微服务地址直接暴露是不安全的。还有所以需要为前端提供一个统一的访问入口。Gateway 就是用于解决以上问题的框架。

## 主要功能

* 路由转发
* 负载均衡
* 安全认证
* 日志记录
* 数据转换

## 重要概念

* Filter（过滤器）

和Zuul的过滤器在概念上类似，可以使用它拦截和修改请求，并且对上游的响应，进行二次处理。过滤器为org.springframework.cloud.gateway.filter.GatewayFilter类的实例

* Route（路由）

网关配置的基本组成模块，和Zuul的路由配置模块类似。一个**Route模块**由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配，目标URI会被访问

* Predicate（断言）

 这是一个 Java 8 的 Predicate，可以使用它来匹配来自 HTTP 请求的任何内容，例如 headers 或参数。**断言的**输入类型是一个 ServerWebExchange

## Gateway 搭建

### 1、引入pomx依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

```

### 2、配置文件

http://localhost:8001 位另一个微服务payment的uri地址。

```yaml
server:
  port: 9527

Spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
        - id: payment_routh
          uri: http://localhost:8001
          predicates:
            - Path = /payment/get/**

        - id: payment_routh2
          uri: http://localhost:8001
          predicates:
            - Path = /payment/lib/**


eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    service-url:
      register-with-eureka: true
      fetch-registry: true
      defaultZone: http://eureka7001.com:7001/eureka/

```

### 3、主启动类

```java
@SpringBootApplication
@EnableEurekaClient
public class GatewayMain9527 {

    public static void main(String[] args) {
        SpringApplication.run(GatewayMain9527.class, args);
    }

}
```

4、测试

通过访问网关来访问payment服务的接口

访问 `http://localhost:9527/payment/get/1` 代替原来的 `http://localhost:8001/payment/get/1`

```json
{"code":200,"message":"查询成功, serverPort: 8001","data":{"id":1,"serial":"jeffcail00001"}}
```

