---
layout: post
title: "微服务 – Spring Cloud – Stream"
date:    2024-6-25
tags: [SpringCloud]
comments: true
author: mazezen
---



Stream 是什么？ 为什么要用Stream？
> SpringCloud Stream是一个构建消息驱动微服务的框架，应用程序通过inputs或者 outputs来与SpringCloud Stream中的binder进行交互。其实就是为了适配底层消息队列的一个抽象出来的中间件。
> 使用 Stream 是为了 解决使用不同的消息队列技术所造成技术结构上的不同所带来的困扰。减少底层消息队列学习的一个成本，方便消息队列技术的迁移



## 如何使用 案列 -  Rabbitmq

结合RabbitMQ的使用案例

Rabbitmq的安装不再赘述。小编采用的docker安装方式

### **1、消息生产者**

引入pom依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
        </dependency>
```

配置文件

```yaml
server:
  port: 8001

spring:
  application:
    name: cloud-stream-provider
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
  cloud:
    stream:
      binders:
        defaultRabbit:
          type: rabbit
      bindings:
        output:
          destination: HelloExchange
          content-type: application/json
          default-binder: defaultRabbit

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka/
  instance:
    lease-renewal-interval-in-seconds: 2
    lease-expiration-duration-in-seconds: 5
    instance-id: sender-8001.com
    prefer-ip-address: true
```

业务类

```java
public interface IMessageProvider {

    String send();

}
```

```java
@EnableBinding(Source.class)
public class IMessageProviderImpl implements IMessageProvider {

    @Resource
    private MessageChannel output;

    @Override
    public String send() {
        String serial = UUID.randomUUID().toString();
        output.send(MessageBuilder.withPayload(serial).build());
        System.out.println("===========serial: " + serial);
        return null;
    }

}

```

```java
@RestController
public class SendMessageController {

    @Resource
    private IMessageProvider messageProvider;

    @GetMapping(value = "/sendMessage")
    public String sendMessage() {
        return messageProvider.send();
    }

}
```

主启动类

```java
@SpringBootApplication
public class RabbitProviderMain {

    public static void main(String[] args) {
        SpringApplication.run(RabbitProviderMain.class, args);
    }

}
```



### **2、消息消费者1 **

引入pom依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
        </dependency>
```

配置文件 

group: 就是queue name 。 同一个队列中多个消费者处于竞争关系，多个消费者属于不同的组，会出现重复消费的问题。 g roup也是消息持久化的配置 

```yaml
server:
  port: 8003

spring:
  application:
    name: cloud-stream-consumer2
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
  cloud:
    stream:
      binders:
        defaultRabbit:
          type: rabbit
      bindings:
        input:
          destination: HelloExchange
          content-type: application/json
          default-binder: defaultRabbit
          group: consumer1

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka/
  instance:
    lease-renewal-interval-in-seconds: 2
    lease-expiration-duration-in-seconds: 5
    instance-id: receiver-8003.com
    prefer-ip-address: true
```

业务类

```java
@RestController
@Component
@EnableBinding(Sink.class)
public class ReceiveMessageController {

    @Value("${server.port}")
    private String serverPort;

    @StreamListener(Sink.INPUT)
    public void  input(Message<String> message) {
        System.out.println("消费者2号, ============ 接收到的消息: " + message.getPayload() + "\t port: " + serverPort);
    }

}
```

主启动类

```java
@SpringBootApplication
public class RabbitConsumerMain {

    public static void main(String[] args) {
        SpringApplication.run(RabbitConsumerMain.class, args);
    }

}
```



### **3、消息消费者2**

引入pom依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
        </dependency>
```

配置文件

group: 就是queue name 。 同一个队列中多个消费者处于竞争关系，多个消费者属于不同的组，会出现重复消费的问题。 g roup也是消息持久化的配置 

```yaml
server:
  port: 8002

spring:
  application:
    name: cloud-stream-consumer2
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
  cloud:
    stream:
      binders:
        defaultRabbit:
          type: rabbit
      bindings:
        input:
          destination: HelloExchange
          content-type: application/json
          default-binder: defaultRabbit
          group: consumer1

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:7001/eureka/
  instance:
    lease-renewal-interval-in-seconds: 2
    lease-expiration-duration-in-seconds: 5
    instance-id: receiver-8002.com
    prefer-ip-address: true
```

业务类

```java
@RestController
@Component
@EnableBinding(Sink.class)
public class ReceiveMessageController {

    @Value("${server.port}")
    private String serverPort;

    @StreamListener(Sink.INPUT)
    public void  input(Message<String> message) {
        System.out.println("消费者3号, ============ 接收到的消息: " + message.getPayload() + "\t port: " + serverPort);
    }

}
```

主启动类

```java
@SpringBootApplication
public class RabbitConsumerMain {

    public static void main(String[] args) {
        SpringApplication.run(RabbitConsumerMain.class, args);
    }

}
```

 ## 测试

同时运行 消息生产者 和两个消费者

访问: `http://127.0.0.1:8801/sendMessage`

在两个消费者控制台即可看到消费信息







