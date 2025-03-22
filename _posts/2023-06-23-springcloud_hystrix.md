---
layout: post
title: "微服务 – Spring Cloud – Hystrix"
date:    2023-06-23
tags: [SpringCloud]
comments: true
author: mazezen
---

## 一、Hystrix 简介

hystrix是Netlifx开源的一款容错框架，防雪崩利器，具备[服务降级](https://so.csdn.net/so/search?q=服务降级&spm=1001.2101.3001.7020)，服务熔断，依赖隔离，监控(Hystrix Dashboard)等功能。

<font color="red">Hystrix is no longer in active development, and is currently in maintenance mode.</font>

<font color="red">Hystrix 已经停更</font>

----

## 二、Hystrix 的作用

* 服务降级
* 服务熔断
* 服务限流

----

## 三、Hystrix使用场景

1. 服务超时
2. 服务宕机（服务崩掉、机房断电、服务故障等）
3. 线程打满
4. 高并发场景
5. 等等

----

## 四、功能点简介

### 1、服务降级 

服务降级是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行

### 2、服务熔断

某服务出现不可用或响应超时的情况时，为了防止整个系统出现雪崩，暂时停止对该服务的调用.

### 3、服务降级VS服务熔断

相同点：

- 目标一致 都是从可用性和可靠性出发，为了防止系统崩溃；
- 用户体验类似 最终都让用户体验到的是某些功能暂时不可用；

不同点：

- 触发原因不同 服务熔断一般是某个服务（下游服务）故障引起，而服务降级一般是从整体负荷考虑

### 4、服务限流

只允许指定数量的事务进入系统处理，超过的部分将被拒绝服务，排队或者降级处理

----

## 五、功能点使用

 ### 1、服务降级

`eureka服务` `payment8001服务` `order服务`

将payment8001注册到eureka服务中心.以供order服务调用

**1.1、新建hysteria-payment服务**

pom依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

配置文件application.yaml

```yaml
server:
  port: 8001

spring:
  application:
    name: cloud-provider-hystrix-payment

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
```

主启动类 HystrixMain8001.java

```java
@SpringBootApplication
@EnableEurekaClient
public class HystrixMain8001 {

    public static void main(String[] args) {
        SpringApplication.run(HystrixMain8001.class, args);
    }

}
```

servicel逻辑层

```java
@Service
public class PaymentService {

    /**
     * 正常访问 ok
     * @param id
     * @return
     */
    public String paymentInfoOk(Integer id) {
        return "线程池: " + Thread.currentThread().getName() + " paymentInfoOk, id: " + id + "\t" + "ok~~~";
    }

    /**
     * 有问题的访问 超时、异常
     * @param id
     * @return
     */
    public String paymentInfoTimeout(Integer id) {
        int timeNumber = 3;
        try {
            TimeUnit.SECONDS.sleep(timeNumber);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "线程池: " + Thread.currentThread().getName() + " paymentInfoTimeout, id: " + id + "\t" + "error~~~" + "耗时： "+ timeNumber;
    }

}
```

控制器层

```java
@RestController
@Slf4j
public class PaymentController {

    @Resource
    private PaymentService paymentService;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/payment/hystrix/ok/{id}")
    public String paymentInfoOk(Integer id) {
        String res = paymentService.paymentInfoOk(id);
        log.info("===================== result ok : " + res);
        return res;
    }

    @GetMapping("/payment/hystrix/timeout/{id}")
    public String paymentInfoTimeout(Integer id) {
        String res = paymentService.paymentInfoTimeout(id);
        log.info("==================== result timeout : " + res);
        return res;
    }

}
```

访问: 

`http://localhost:8001/payment/hystrix/ok/3`

```html
线程池: http-nio-8001-exec-1 paymentInfoOk, id: null ok~~~
```

`http://localhost:8001/payment/hystrix/timeout/3` 模拟卡顿3秒后

```html
线程池: http-nio-8001-exec-1 paymentInfoTimeout, id: null error~~~耗时： 3
```

**1.2 服务降级**

1.2.1 、服务自身降级

<font color="red">**将服务逻辑层睡眠时间提升为6秒。设置超时时间峰值为3秒，超过3秒接口没响应，服务自身降级处理**</font>

通过配置@Hystrixcommand

设置服务接口自身超时时间的峰值，在峰值内视为正常。超出峰值进行fallback。

主启动类添加注解 @EnableCircuitBreaker

```java
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
public class HystrixMain8001 {

    public static void main(String[] args) {
        SpringApplication.run(HystrixMain8001.class, args);
    }

}
```

业务逻辑层添加注册实现服务自身降级

```java
 /**
     * 超时时间 3秒 演示降级
     * @param id
     * @return
     */
    @HystrixCommand(fallbackMethod = "paymentInfo_TimeoutHandler", commandProperties = {
            @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value = "3000")
    })
    public String paymentInfoTimeout(Integer id) {
        int timeNumber = 6;
        try {
            TimeUnit.SECONDS.sleep(timeNumber);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "线程池: " + Thread.currentThread().getName() + " paymentInfoTimeout, id: " + id + "\t" + "error~~~" + "耗时： "+ timeNumber;
    }

    public String paymentInfo_TimeoutHandler(Integer id) {
        return "~～～啦啦啦 服务开小差儿了: " + "\t" + "当前线程名字: " + Thread.currentThread().getName();
    }
```

访问: `http://localhost:8001/payment/hystrix/timeout/3`

```html
~～～啦啦啦 服务开小差儿了: 当前线程名字: HystrixTimer-1
```

1.2.1 、调用方服务降级

新建调用方服务。 服务提供方，模拟了6秒睡眠时间。超过3秒服务提供方会自身降级处理，调用方设置的超时峰值为1.5秒，超过峰值，调用方服务降级处理。

配置文件 application.yaml

```yaml
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
feign:
  hystrix:
    enabled: true
```

主启动类添加 @EnableHystrix

```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
public class ConsumerHystrixMain80 {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerHystrixMain80.class, args);
    }

}
```

控制器或者业务逻辑层

```java
@GetMapping("/consumer/payment/hystrix/timeout/{id}")
    @HystrixCommand(fallbackMethod = "paymentInfo_TimeoutHandler", commandProperties = {
            @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value = "1500")
    })
    public String paymentInfoTimeout(@PathVariable("id") Long id){
        String res = paymentHystrixService.paymentInfoTimeout(id);
        return res;
    }

    public String paymentInfo_TimeoutHandler(@PathVariable("id") Long id) {
        return "~～～啦啦啦 服务提供方的服务开小差儿了: " + "\t" + "当前线程名字: " + Thread.currentThread().getName();
    }
```

访问： `http://localhost/consumer/payment/hystrix/timeout/3`

```html
~～～啦啦啦 服务提供方的服务开小差儿了: 当前线程名字: hystrix-PaymentHystrixController-1
```



----

### 2、服务熔断



```java
// ===== 服务熔断
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback", commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled", value = "true"), // 是否开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"), // 请求次数
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"), // 时间窗口期
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"), // 失败率
    })
    public String paymentCircuitBreaker(@PathVariable("id") Integer id) {
        if (id < 0) {
            throw new RuntimeException("******** id 不能为负数");
        }
        String serialNumber = IdUtil.simpleUUID();

        return Thread.currentThread().getName() + "\t" + "调用成功,流水号：" + serialNumber;
    }

    public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id) {
        return "id 不能为负数， 请稍后再试. id: " + id;
    }

```

