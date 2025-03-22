---
layout: post
title: "微服务 – Spring Cloud – Eureka"
date:    2023-6-21
tags: [SpringCloud]
comments: true
author: mazezen
---


### Eureka 服务管理

Eureka是Netflix开发的服务发现框架，本身是一个基于REST的服务，主要用于定位运行在AWS域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。

SpringCloud将它集成在其子项目spring-cloud-netflix中，以实现SpringCloud的服务发现功能

##### Eureka服务注册与发现

Eureka 采用了CS的设计架构，Eureka Server作为服务注册功能的服务器，它是服务注册的中心。而系统中的其他微服务，使用 Eureka 的客户端链接到
Eureka server并维持心跳连接。使开发人员可以通过Eureka serve来监控系统中各个微服务的运行状态.

在服务注册与发现中，有一个注册中心。当服务启动的时候，会把当前自己的服务器的信息，比如 服务地址、通讯地址等以别名方式注册到注册中心。另一方
（消费者｜服务提供者），以该别名的方式去注册中心上获取到实际的服务通讯地址，然后再实现本地RPC调用远程RPC。远程调用框架的核心设计思想：在于注册中心
，因为使用注册中心管理每个服务于服务之间的一个依赖关系（服务治理概念）。在任何RPC远程框架中，都会有一个注册中心（存放服务地址相关信息）

### Eureka 三种角色

* Eureka Server ：提供服务注册和发现等
* Service Provider：服务提供者：自身注册到Eureka Server，供消费端调用
* Service Consumer：服务消费方：从Eureka获取注册服务列表，从而能够消费服务



# ureka Server搭建

**在父工程中创建module 'server7001' 引入 spring-cloud ureka 依赖**

```yaml
<dependencies>

        <!-- eureka server依赖包 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

    </dependencies>
```

**配置文件**

```yaml
server:
  port: 7001

eureka:
  instance:
    # eureka 服务端的实例名称
    hostname: localhost
  client:
    # 表示是否将自己注册到Eureka Server，默认为true。
    register-with-eureka: false
    # 表示是否从Eureka Server获取注册信息，默认为true。
    fetch-registry: false
    service-url:
      # 设置与Eureka Server交互的地址
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```



**配置启动类**

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication7001 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication7001.class, args);
    }

}
```

**启动服务** `浏览器访问:  localhost:7001`



## 服务注册到 Eureka 服务中心

在需要注册的服务中

**引入pom依赖**

```xml
<!-- eureka-client -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

```

**配置文件**

```yaml
eureka:
  client:
    # 表示是否将自己注册到Eureka Server，默认为true。
    register-with-eureka: true
    # 表示是否从Eureka Server获取注册信息，默认为true。
    fetch-registry: true
    service-url:
      # 设置与Eureka Server交互的地址
      defaultZone: http://localhost:7001/eureka
```

**启动类**

```java
@SpringBootApplication
@EnableEurekaClient
public class PaymentMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8001.class, args);
    }
}
```

**测试**启动，然后去 Eureka服务中心查看服务是否注册进来。

在  Eureka服务中心 的 DS Replicas中就会发现注册进来的服务

```
DS Replicas
Instances currently registered with Eureka
Application	AMIs	Availability Zones	Status
CLOUD-PAYMENT-SERVICE	n/a (1)	(1)	UP (1) - 192.168.2.61:cloud-payment-service:8001

```



# eureka Server 集群搭建

**在父工程中创建module 'server7002' 引入 spring-cloud ureka 依赖**

```xml
<dependencies>

  <!-- eureka server依赖包 -->
  <dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>

</dependencies>
```

**单机模拟eureka集群搭建，同一个ip不同端口**

**修改hosts文件**

```hosts
127.0.0.1 eureka7001.com
127.0.0.1 eureka7002.com
```

**修改配置文件**

server7002配置文件

```yaml
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
```

server7002配置文件

```yaml
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
```

**启动类**

server7001无需修改

新建server7002启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication7002 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication7002.class, args);
    }

}
```

**启动服务** `浏览器访问:  localhost:7001`

**启动服务** `浏览器访问:  localhost:7002`



### **服务注册到 Eureka 集群服务中心**

在需要注册的服务中。只需要修改application.yaml文件即可。还以上面的服务注册为例

```yaml
eureka:
  client:
    # 表示是否将自己注册到Eureka Server，默认为true。
    register-with-eureka: true
    # 表示是否从Eureka Server获取注册信息，默认为true。
    fetch-registry: true
    service-url:
      # 设置与Eureka Server交互的地址
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
```

**测试**启动，然后去 Eureka  server7001 和 server7002 两个 服务中心查看服务是否注册进来。

在  Eureka  server7001 和 server7002 两个 服务中心  的 DS Replicas中就会发现注册进来的服务

```
DS Replicas
Instances currently registered with Eureka
Application	AMIs	Availability Zones	Status
CLOUD-PAYMENT-SERVICE	n/a (1)	(1)	UP (1) - 192.168.2.61:cloud-payment-service:8001

```




**背景：**

* 服务注册用的是 Eureka集群。

* 服务调用用的是注解 @LoadBalanced 和  RestTemplate

* 服务数量两个： order服务 和 pyment服务 （**order服务是调用者。 payment 服务是被调用者**）



1. **首先将 order服务 和 payment服务注册 Eureka集群中。通过order调用 payment服务**

2. **Eureka集群 的搭建 和 rder服务 和 payment服务注册 Eureka集群中 可以查看上一篇文章，有详细的步骤。**

3. **order服务调用payment服务用的http方式。用的是 RestTemplate.后续会使用RPC方式**



**实现代码**

整合<font color="red">`Ribbon`</font>，引入依赖。在eureka依赖中，已经潜入了 <font color="red">`Ribbon`</font>

```yaml
<!-- eureka-client -->
  <dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```



配置文件 ApplicationContextConfig.java

RestTemplate 可以结合 Eureka 来动态发现服务并进行负载均衡的调用。

修改 RestTemplate 的配置，增加能够让 RestTemplate 具备负载均衡能力的注解 @LoadBalanced

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

控制器 OrderController.java

`CLOUD-PAYMENT-SERVICE` 为payment服务注册到  Eureka集群中的服务名。

```java
@RestController
public class OrderController {

    private static final String PROVIDER_URL = "http://CLOUD-PAYMENT-SERVICE";

    @Resource
    private RestTemplate restTemplate;

    @GetMapping("/consumer/create/payment")
    public CommonResult<Payment> create(Payment payment) {
        return restTemplate.postForObject(PROVIDER_URL + "/payment/create", payment, CommonResult.class);
    }

    @GetMapping("/consumer/payment/get/{id}")
    public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id) {
        return restTemplate.getForObject(PROVIDER_URL + "/payment/get/"+id, CommonResult.class);
    }

}
```



## @LoadBalanced 

@LoadBalanced 采用的是轮训的方式进行服务调用。



### 为什了加了 @LoadBalanced  就可用通过 Eureka集群中的服务名称，并可以实现负载均衡呢？



因为 Spring Cloud 给我们做了大量的底层工作，因为它将这些都封装好了。

这里主要的逻辑就是给 RestTemplate 增加拦截器，在请求之前对请求的地址进行替换，或者根据具体的负载策略选择服务地址，然后再去调用。




### **如何发现服务呢？**


服务注册到 Eureka 集群中。需要通过 RestTemplate和@LoadBalanced 实现服务发现调用(http) 。

order 服务 通过 estTemplate和@LoadBalanced 实现调用 payment服务. 是通过注册在  Eureka 集群中的服务名称来调用的。

**那么如何发现这些服务呢？也就是说如何知道注册在  Eureka 集群中的服务名称**

通过DiscoveryClient 和 @EnableDiscoveryClient 注解实现



<font color="red">正常来讲，服务发现应该是服务调用者的事情。这里为了方便代码写在了PaymentController类里.只是为了实现自测</font>

主启动类

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class PaymentMain8002 {

    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8002.class, args);
    }

}
```

控制器 PaymentController.java

```java
@GetMapping(value = "/payment/discovery")
    public Object discovery() {
        List<String> services = discoveryClient.getServices();
        for (String service : services) {
            log.info("===================== service: " + service);
        }

        List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
        for (ServiceInstance instance : instances) {
            log.info("********************** instance: " + instance.getServiceId()
                    + instance.getHost()
                    + instance.getPort()
                    + instance.getUri());
        }

        return this.discoveryClient;
    }
```

`访问： localhost/payment/discovery`

```json
{
  "order": 0,
  "services": [
    "cloud-payment-service",
    "cloud-order-service"
  ]
}
```

```text
2023-06-20 23:13:57.410  INFO [cloud-payment-service,a84f2b64acc1e87a,a84f2b64acc1e87a,true] 7444 --- [nio-8002-exec-1] c.j.s.controller.PaymentController       : ===================== service: cloud-payment-service
2023-06-20 23:13:57.411  INFO [cloud-payment-service,a84f2b64acc1e87a,a84f2b64acc1e87a,true] 7444 --- [nio-8002-exec-1] c.j.s.controller.PaymentController       : ===================== service: cloud-order-service
2023-06-20 23:13:57.412  INFO [cloud-payment-service,a84f2b64acc1e87a,a84f2b64acc1e87a,true] 7444 --- [nio-8002-exec-1] c.j.s.controller.PaymentController       : ********************** instance: CLOUD-PAYMENT-SERVICE192.168.2.618001http://192.168.2.61:8001
2023-06-20 23:13:57.412  INFO [cloud-payment-service,a84f2b64acc1e87a,a84f2b64acc1e87a,true] 7444 --- [nio-8002-exec-1] c.j.s.controller.PaymentController       : ********************** instance: CLOUD-PAYMENT-SERVICE192.168.2.618002http://192.168.2.61:8002

```



#### **自我保护是什么？**

自我保护是一种针对网络异常波动的安全保护措施，自我保护能使Eureka集群更加的健壮、稳定的运行。



因为 Eureka 客户端会定时的向 Eureka 服务端 发送心跳检测包， 默认30秒发送一次。发送目的是为了通知 Eureka 服务端  <font color="read">你好，老六。我还在，别删我！</font>。如果超过90秒（默认90秒），服务端没有收到 Eureka 客户端发送的心跳包，就会认为 <font color="red"> My son, you are dead! 我要把你踢出去!</font> 但是如果短时间内丢失了Eureka 客户端发送的心跳包， Eureka 服务端便会触发自我保护机制。而且 Eureka 服务端的自我保护机制是默认开启的。触发自我保护机制会出现红色字体警告： 

<font color="red">**EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.**</font>



#### 为什么要有自我保护机制?

其实说白了，如果因为网络波动的原因导致的，丢失大量心跳包，直接剔除服务，并不妥当。因为服务可能并没有挂掉。这个时候 Eureka 启动自我保护机制，就会把当前的服务注册信息保护起来。生产环境中非常有用。在生产环境中轻易的剔除某个服务，会导致服务性能下降，甚至是宕机，丢失数据。



### 场景

开发测试场景下，可以将心跳时间缩短。可以保证服务关闭后能够得到及时的剔除

生成环境下，需要开启自我保护机制。防止服务轻易的被剔除

### 关闭Eureka 的自我保护机制

Eureka 服务端的配置

```yaml
  server:
    enable-self-preservation: false # 禁用自我保护机制
    eviction-interval-timer-in-ms: 2000 # 2秒钟
```

Eureka客户端配置：

```yaml
eureka:
  instance:
    # Eureka 服务端 在收到最后一次心跳后等待的时间上限， 单位为秒（默认是90秒），超时提出服务
    lease-expiration-duration-in-seconds:2
    # Eureka 客户端向服务端发送心跳的时间间隔 单位为秒 （默认30秒）
    lease-renewal-interval-in-seconds: 1
```










































