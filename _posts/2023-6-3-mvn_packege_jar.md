---
layout: post
title: "mvn 打包jar包。 Docker 部署 jar 包程序"
date:    2023-6-3
tags: [Java]
comments: true
author: mazezen
---


默认你已经安装了jdk和maven 并且配置了环境变量. 这里贴出自己的环境配置(mac)

```shell
# Maven3.6.3
export M2_HOME=/Users/cc/maven3.6.3/apache-maven-3.6.3
export M2=$M2_HOME/bin
export PATH=$M2:$PATH

# ======================= java8 ====================
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_361.jdk/Contents/Home
export PATH=$PATH:$JAVA_HOME/bin
```



### 一、pom引入打包依赖 . 然后 Reaload

```xml
<build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <executions>
          <execution>
            <goals>
              <goal>repackage</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.1</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>

    </plugins>
</build>
```



### 二、打包

```shell
cd 到项目跟目录
执行 mvn clean package

会在target 下生成两个文件：

项目名-0.0.1-SNAPSHOT.jar
项目名-0.0.1-SNAPSHOT.jar.original
```

### 三. 执行  查看jar是否能跑起来

```shell
java -jar 项目名-0.0.1-SNAPSHOT.jar
```



## 服务器上 docker 部署jar包

默认你服务上已经安装了docker

最好建一个项目专属目录。 我的 `/root/otter-exam`

Dockerfile文件、jar包、application.yml文件、run.sh启动脚本文件最好都在 `/root/otter-exam` 具体里面层级可自己划分

#### 一、编写Dockerfile文件

```dockerfile
# 基础镜像使用java
FROM openjdk:8-jdk-alpine
# 作者 名字 邮箱 （可以不写）
MAINTAINER jeffcail <cljcailianjie@163.com>
# 将jar包添加到容器中并更名为app.jar
ADD otter-exam-0.0.1-SNAPSHOT.jar exam.jar

# 运行端口
EXPOSE 8099
# 启动命令，加载指定路径的配置文件
ENTRYPOINT ["java","-jar","exam.jar","--spring.config.location=application.yml"]

## 设置所属时区
ENV TZ=Asia/Shanghai
## 创建本地和容器的连接
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

```

* ADD otter-exam-0.0.1-SNAPSHOT.jar exam.jar 将jar包添加到容器中，也可以使用 `COPY` 

* "--spring.config.location=application.yml" 加上指定的配置文件。比如 一个镜像多个客户端公用

  



#### 二、编写run.sh脚本

脚本的作用。节省手动操作。比如每次发布，都要先打包镜像、停止容器、删除容器、删除镜像、运行镜像。

使用脚本一键执行，让脚本来完成这些动作

```shell
#!/usr/bin/env bash

echo "停止容器"

docker stop otter-exam-0.0.1

echo "删除容器"

docker rm otter-exam-0.0.1

echo "删除旧的镜像"

docker rmi otter-exam-0.0.1

echo "开始构建镜像"

docker build -t otter-exam-0.0.1 .

echo "开始运行镜像"

docker run -d --restart=always --name otter-exam-0.0.1 -p 8099:8099 \
  -v /root/otter-exam/application.yml:/application.yml \
  -v /etc/localtime:/etc/localtime \
  otter-exam-0.0.1:latest \

docker logs -f otter-exam-0.0.1 | sed '/Started otter-exam-0.0.1 Application/q'

echo ""
```

* `docker build -t otter-exam-0.0.1 .` 打包镜像
* `docker run -d --restart=always --name otter-exam-0.0.1 -p 8099:8099 -v /root/otter-exam/application.yml:/application.yml -v /etc/localtime:/etc/localtime otter-exam-0.0.1:lates` 运行打包好的镜像，并把配置文件挂载到容器中，Dockfile中 `--spring.config.location=application.yml` 就可以使用容器中的配置文件
* `docker logs -f otter-exam-0.0.1 | sed '/Started otter-exam-0.0.1 Application/q'`启动之后打印日志



#### 三、 云服务器记得在安全组开放对应端口 这里 `8099`

#### 四、 部署好了之后验证是否能访问

这里访问这套程序的swagger文档地址

![](http://images.caixiaoxin.cn/swagger3.jpg)