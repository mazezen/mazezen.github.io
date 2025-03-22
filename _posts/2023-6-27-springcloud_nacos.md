---
layout: post
title: "微服务 – Spring Cloud – Nacos 配置中心"
date:    2024-6-27
tags: [SpringCloud]
comments: true
author: mazezen
---

打开nacos面板新建配置

**Data ID**: nacos-config-client-dev.yaml

**Group:** DEV-CLOUD2023

```
config:
    info: config info lalalal 小魔仙~~~~
```

## 引入依赖

```xml
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```

## 配置文件

bootstrap.yaml

```yaml
server:
  port: 9999

spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: localhost:7848 #服务注册中心地址
      config:
        server-addr: localhost:7848 #配置中心地址
        file-extension: yaml #指定yaml格式的配置
        group: DEV-CLOUD2023

```

application.yaml

```yaml
spring:
  profiles:
    active: dev
```

## 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class NacosConfigClientMain {

    public static void main(String[] args) {
        SpringApplication.run(NacosConfigClientMain.class, args);
    }

}
```

## 业务类

```java
@RestController
@RefreshScope
public class NacosConfigController {

    @Value("${config.info}")
    private String configInfo;

    @GetMapping("/config/info")
    public String getConfigInfo() {
        return configInfo;
    }

}

```

`访问 : http://localhost:3377/config/info`

```html
config info lalalal 小魔仙~~~~
```




## 服务注册

[TOC]

### 1、引入依赖

父pom依赖

```xml
<!--spring cloud alibaba 2.1.0.RELEASE-->
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-alibaba-dependencies</artifactId>
  <version>2.1.0.RELEASE</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>


```

子pom依赖

```xml

<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

### 2、配置文件

```yaml
server:
  port: 9001

spring:
  application:
    name: nacos-payment-provider
  cloud:
    nacos:
      discovery:
        server-addr: localhost:7848

management:
  endpoints:
    web:
      exposure:
        include: '*'
```

### 3、主启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AlibabaPaymentMain9001 {

    public static void main(String[] args) {
        SpringApplication.run(AlibabaPaymentMain9001.class, args);
    }

}

```

`第三部完成` 打开nacos 在服务列表即可看到注册进来的服务.

### 4、业务类 写一个接口供服务发现者使用

```java
@RestController
public class PaymentController {

    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/nacos/{id}")
    public String getPayment(@PathVariable("id") Integer id) {
        return "nacos registery , server port: " + serverPort + "\t id: " + id;
    }

}
```

## 服务发现

### 1、引入依赖

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

### 2、配置文件

```yaml
server:
  port: 9002

spring:
  application:
    name: nacos-payment-consumner
  cloud:
    nacos:
      discovery:
        server-addr: localhost:7848

management:
  endpoints:
    web:
      exposure:
        include: '*'
```

### 3、主启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AlibabaConsumnerMain9002 {

    public static void main(String[] args) {
        SpringApplication.run(AlibabaConsumnerMain9002.class, args);
    }

}
```

` 打开nacos 在服务列表即可看到注册进来的服务.`

### 4、发现第一个服务 并调用第一个服务提供的接口

nacos默认支持负载,因为底层nacos包集成了Ribbon

服务调用用的还是`RestTemplate`

在第二个服务中，配置RestTemplate

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

业务类中调用第一个服务的接口

```java
@RestController
public class ConsumnerController {

    public static final String SERVER_URL = "http://nacos-payment-provider";

    @Resource
    private RestTemplate restTemplate;

    @GetMapping("/consumer/{id}")
    public String paymentInfo(@PathVariable("id") Long id) {
        return restTemplate.getForObject(SERVER_URL + "/payment/nacos/" + id, String.class);
    }

}

```

访问`http://localhost:9002/consumer/1` 

```html
nacos registery , server port: 9001 id: 1
```



