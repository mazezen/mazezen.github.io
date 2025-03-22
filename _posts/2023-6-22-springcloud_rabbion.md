---
layout: post
title: "微服务 – Spring Cloud – Rabbion"
date:    2023-6-22
tags: [SpringCloud]
comments: true
author: mazezen
---

## Ribbon 简介

Ribbon是Netflix发布的开源项目，主要目的是为客户端提供负载均衡算法和服务调用。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。简单地说，就是在配置文件中列出Load Balancer(简称LB)后面所有机器，Ribbon会自动的帮助你基于某种规则(如简单轮询，随机连接等)去连接这些机器。我们很容易使用Ribbon实现自定义的负载均衡算法。

## 负载均衡 + RestTemplate 示例

### 一、Ribbon默认的负载规则 - 轮询

1、**启动eureka 集群**

localhost:7001

localhost:7002

pom依赖

```yaml
 <!-- eureka server依赖包 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

配置文件 application.yaml

```yaml
-- 7001
server:
  port: 7001

eureka:
  instance:
    # eureka 服务端的实例名称
    hostname: eureka7001.com
  client:
    # 表示是否将自己注册到Eureka Server，默认为true。
    register-with-eureka: false
    # 表示是否从Eureka Server获取注册信息，默认为true。
    fetch-registry: false
    service-url:
      # 设置与Eureka Server交互的地址
      defaultZone: http://eureka7002.com:7002/eureka/
  server:
    enable-self-preservation: false # 禁用自我保护机制
    eviction-interval-timer-in-ms: 2000 # 2秒钟
    
    

-- 7002
server:
  port: 7002

eureka:
  instance:
    # eureka 服务端的实例名称
    hostname: eureka7002.com
  client:
    # 表示是否将自己注册到Eureka Server，默认为true。
    register-with-eureka: false
    # 表示是否从Eureka Server获取注册信息，默认为true。
    fetch-registry: false
    service-url:
      # 设置与Eureka Server交互的地址
      defaultZone: http://eureka7001.com:7001/eureka/

  server:
    enable-self-preservation: false # 禁用自我保护机制
    eviction-interval-timer-in-ms: 2000 # 2秒钟


```

主启动类 7001 、7002

```java'
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication7001 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication7001.class, args);
    }

}

@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication7002 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication7002.class, args);
    }

}
```



![](http://images.caixiaoxin.cn/eureka7001.jpg)

![](http://images.caixiaoxin.cn/eureka7002.jpg)



2、**两个服务提供者 payment 8001 和 payment 8002。并注册到eureka注册中心**

启动类 8001 和 8002

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class PaymentMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8001.class, args);
    }
}

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class PaymentMain8002 {

    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8002.class, args);
    }

}
```

提供者业务类 8001 和 8002 部分代码

```java
@RestController
@Slf4j
public class PaymentController {

    @Autowired
    private PaymentService paymentService;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/get/{id}")
    public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id) {
        Payment res = paymentService.getPaymentById(id);
        log.info("************************** 查询结果: " + res);
        if (res != null) {
            return new CommonResult(200, "查询成功, serverPort: " + serverPort, res);
        }
        return new CommonResult(444, "查询失败",null);
    }

}

```

![](http://images.caixiaoxin.cn/eureka-payment8001.jpg)

![](http://images.caixiaoxin.cn/eureka-payment8002.jpg)



3、**消费者服务 `order` 以轮询的方式 通过 `eureka` 调用 `payment8001` 和`payment8002`**

启动类

```java
@SpringBootApplication
@EnableEurekaClient
public class OrderMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class, args);
    }

}
```

消费者服务部分代码

```java
@RestController
public class OrderController {

    private static final String PROVIDER_URL = "http://CLOUD-PAYMENT-SERVICE";

    @Resource
    private RestTemplate restTemplate;

    @GetMapping("/consumer/payment/get/{id}")
    public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id) {
        return restTemplate.getForObject(PROVIDER_URL + "/payment/get/"+id, CommonResult.class);
    }

}
```

访问`localhost/consumner/payment/get/6`

第一次访问结果： 调用的 8002 

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8002",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```

第二次访问结果： 调用的 8001

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8001",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```

第三次访问结果： 调用的 8002

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8002",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```

第四次访问结果： 调用的 8001

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8001",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```

因为Rabbion 默认的负载均衡机制为 `轮询`。



### 二、Ribbon负载规则 - 随机

在主主启动类所在的package的同级目录下，新建ribbonrule package。 新建MyselfRule.java 类

![在这里插入图片描述](http://images.caixiaoxin.cn/ribbonrule.jpg)

MyselfRule.java类

```java
@Configuration
public class MyselfRule {

    @Bean
    public IRule myRule() {
        return new RandomRule(); // 定义为随机规则
    }

}
```

在主启动类上添加 @RibbonClient 注解

```java
@SpringBootApplication
@EnableEurekaClient
@RibbonClient(name = "CLOUD-PAYMENT-SERVICE", configuration = MyselfRule.class)
public class OrderMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class, args);
    }

}
```

重启 `order`服务 ，然后访问 `localhost/consumner/payment/get/4`

第一次结果: 

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8001",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```

第二次结果: 

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8001",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```

第三次结果: 

```json
{
"code": 200,
"message": "查询成功, serverPort: 8002",
"data": {
"id": 4,
"serial": "jeffcail00004"
}
}
```

第四次结果:

```json
{
  "code": 200,
  "message": "查询成功, serverPort: 8002",
  "data": {
  "id": 4,
  "serial": "jeffcail00004"
  }
}
```


