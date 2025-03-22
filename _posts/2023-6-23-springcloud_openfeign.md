---
layout: post
title: "微服务 – Spring Cloud – OpenFeign"
date:    2023-6-23
tags: [SpringCloud]
comments: true
author: mazezen
---

OpenFeign 简介

> OpenFeign 提供了一种**声明式的远程调用接口**。

## OpenFeign 能做什么

目的是为了简易HTTP客户端的编写。

之前在 笔记中介绍了 Ribbon + RestTemplate 的使用。Ribbon + RestTemplate 是多http请求做了封装处理，形成了模版化的调用。但是在实际的开发中，由于对服务依赖的调用可能不止一处，往往一个接口被多处调用，所以需要对每个微服务进行封装。鉴于此 Feign 在此基础上为我们提供了封装操作，由Feign 帮我门定义和实现依赖服务接口的定义。因此简化了我们的操作，只需要创建一个接口并使用注解的形式来配置它（比如Mapper接口上标注@Mapper注解，现在是在一个微服务接口上标注一个@FeignClient注解），就可以完成对服务接口的绑定，简化了Spring cloud Ribbon使用时候封装客户端的开发量

## 如何使用

Declarative REST Client: Feign creates a dynamic implementation of an interface decorated with JAX-RS or Spring MVC annotations

在主启动类上通过注解 @EnableFeignClients、 接口上通过注解 @FeignClient 实现

```java
@SpringBootApplication
@EnableFeignClients
public class WebApplication {

	public static void main(String[] args) {
		SpringApplication.run(WebApplication.class, args);
	}

	@FeignClient("name")
	static interface NameService {
		@RequestMapping("/")
		public String getName();
	}
}
```

## OpenFeign 使用Example

* eureka服务集群 `127.0.0.1:7001` `127.0.0.1:7002`
* 微服务 - 消费者服务 `consumner-order`， 服务注册 eureka
* 微服务 - 支付服务 `payment8001`, `payment8002`, 服务注册进eureka。服务名叫 `CLOUD-PAYMENT-SERVICE` 

1、新建子模块 consumner-order

依赖pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2、配置文件application.yaml'

```yaml
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
```

3、主启动类

@EnableFeignClients 开启 Feign 客户端

```java
@SpringBootApplication
@EnableFeignClients
public class FeignMain80 {

    public static void main(String[] args) {
        SpringApplication.run(FeignClient.class, args);
    }

}

```

4、接口层 service/PaymentFeignService.java

```java
@Component
@FeignClient(value = "CLOUD-PAYMENT-SERVICE")
public interface PaymentFeignService {

    @GetMapping(value = "/payment/get/{id}")
    CommonResult<Payment> getPaymentById(@PathVariable("id") Long id);

}
```

5、业务控制器 controller/OrderController.java

```java
@RestController
public class OrderController {

    @Resource
    private PaymentFeignService paymentFeignService;

    @GetMapping("/consumer/payment/get/{id}")
    public CommonResult<Payment> paymentLb(@PathVariable("id") Long id) {
         return paymentFeignService.getPaymentById(id);
    }

}
```

6、启动eureka7001、eureka7002、payment8001、payment8002、consumer-order

访问 `127.0.0.1:80//consumer/payment/get/4`

```json
# 第一次

{"code":200,"message":"查询成功, serverPort: 8001","data":{"id":4,"serial":"jeffcail00004"}}

# 第二次

{"code":200,"message":"查询成功, serverPort: 8002","data":{"id":4,"serial":"jeffcail00004"}}

# 第三次

{"code":200,"message":"查询成功, serverPort: 8001","data":{"id":4,"serial":"jeffcail00004"}}

# 第四次

{"code":200,"message":"查询成功, serverPort: 8002","data":{"id":4,"serial":"jeffcail00004"}}
```

## OpenFeign 超时控制 （OpenFeign默认超时时间为1秒）

在payment8001、payment8002中新增接口

```java
@GetMapping(value = "/payment/feign/timeout")
    public String paymentFeignTimeout() {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return serverPort;
    }
```

在order 服务中新增调用接口

service/PaymentFeignService.jva

```java
@GetMapping(value = "/payment/feign/timeout")
String paymentFeignTimeout();
```

业务控制器 controller/OrderController.java

```java
@GetMapping(value = "/consumer/payment/feign/timeout")
public String paymentFeignTimeout() {
  return paymentFeignService.paymentFeignTimeout();
}
```

访问: `http://localhost/consumer/payment/feign/timeout`

超时错误

```html
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Thu Jun 22 13:39:17 CST 2023
There was an unexpected error (type=Internal Server Error, status=500).
Read timed out executing GET http://CLOUD-PAYMENT-SERVICE/payment/feign/timeout
```

YAML文件中开启超时配置

```yaml
# 设置feigin 客户端超时时间（OpenFeign默认支持ribbon）
ribbon:
  # 读超时时间 5秒
  ReadTimeout: 5000
  # 建立连接 超时时间 5秒
  ConnectTimeout: 5000
```

访问: `http://localhost/consumer/payment/feign/timeout`

等待3秒后

```html
8001
```





