---
layout: post
title: "微服务 二十一 Mycat中间件 实现主从读写分离"
date: 2022-05-17
tags: [Go]
comments: true
author: mazezen
---

# mycat
**下载地址: http://dl.mycat.org.cn/**
### 简介
MyCat 是目前最流行的基于 java 语言编写的数据库中间件，是一个实现了 MySQL 协议的服务器，用户可以把它看作是一个数据库代理，用 MySQL 客户端工具和命令行访问，而其后端可以用 MySQL 原生协议与多个 MySQL 服务器通信，也可以用 JDBC 协议与大多数主流数据库服务器通信，其核心功能是分库分表。配合数据库的主从模式还可实现读写分离

MyCat 是基于阿里开源的 Cobar 产品而研发，Cobar 的稳定性、可靠性、优秀的架构和性能以及众多成熟的使用案例使得 MyCat 变得非常的强大。
MyCat 发展到目前的版本，已经不是一个单纯的 MySQL 代理了，它的后端可以支持MySQL、SQL Server、Oracle、DB2、PostgreSQL 等主流数据库。而在最终用户看来，无论是那种存储方式，在 MyCat 里，都是一个传统的数据库表，支持标准的 SQL 语句进行数据的操作，对业务系统来说，可以大幅降低开发难度，提升开发速度

### mycat 原理
Mycat的原理中最重要的一个动词是“拦截”.
它拦截了用户发送过来的SQL语句，首先对SQL语句做了一些特定的分析：如分片分析、路由分析、读写分离分析、缓存分析等，然后将此SQL发往后端的真实数据库，并将返回的结果做适当的处理，最终再返回给用户

### 如何使用
1. 一主一从
2. 双主一从
3. docker方式

# 一主一从
###  机器规划

| 名字          | IP   | 配置           |
| ------------- | ---- | -------------- |
| master        | 172.16.129.151     | 2CPU 2G 20硬盘 |
| slave         |  172.16.129.152    | 2CPU 2G 20硬盘 |
| Mysql - proxy |   172.16.129.153   | 2CPU 2G 20硬盘 |

**三台机器都执行的操作**

```shell
1. 联网
 vim /etc/sysconfig/network-scripts/ifcfg-ens33
 ONBOOT=yes
 service network restart

2. yum -y install vim

3. yum -y install wget

4. yum -y install net-tools

5. service firewalld stop && systemctl disable firewalld.service && service firewalld status
```

### 安装 mysql

查看我的个人博客<a href="http://blog.caixiaoxin.cn/?p=654" target="_blank" rel="noopener">微服务十八 搭建 mysql 主从同步</a>

### 配置主从

查看我的个人博客<a href="http://blog.caixiaoxin.cn/?p=654" target="_blank" rel="noopener">微服务十八 搭建 mysql 主从同步</a>



### 安装mycat

#### 1. 安装 JDK1.8

https://www.oracle.com/java/technologies/downloads/

```shell
1. 下载 jdk 并上传

mkdir /usr/java
tar xf jdk-8u333-linux-x64.tar.gz -C /usr/java
```

#### 2. 配置环境变量

```shell
vim /etc/profile.d/java.sh                        # /etc/profile.d/目录下创建java.sh 文件并定入如下内容

JAVA_HOME=/usr/java/jdk1.8.0_333
PATH=$JAVA_HOME/bin:$PATH 
CLASSPATH=$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar
export PATH JAVA_HOME CLASSPATH
```

```shell
chmod +x /etc/profile.d/java.sh
source /etc/profile.d/java.sh
java -version
```

#### 3. 下载mycat

```shell
# 废弃
tar xf Mycat-server-1.5.1-RELEASE-20161130213509-linux.tar.gz -C /usr/local/

# 备份 以防改错方便恢复
cp schema.xml schema.xml.bak  # 定义逻辑库，表、分片节点等内容
cp rule.xml rule.xml.bak      # 定义分片规则
cp server.xml server.xml.bak  # 定义用户以及系统相关变量
```

#### 4. 配置环境变量

```shell
vim /etc/profile.d/mycat.h   # 在/etc/profile.d 目录下创建mycat.sh 文件，并写入如下

MYCAT_HOME=/usr/local/mycat 
PATH=$MYCAT_HOME/bin:$PATH
```

```shell
source /etc/profile.d/mycat.h                   # 使环境变量生效
```

#### 5. mycat

* 账号授权

在master上执行即可（slave已做主从同步）

```mysql
-- mysql8 之前版本 执行
GRANT ALL PRIVILEGES ON *.* TO 'mycat'@"%" IDENTIFIED BY "123456";


-- mysql8 执行
create user mycat@'%' identified by '123456';
grant all privileges on *.* to mycat@'%' with grant option;

ALTER USER 'mycat'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
```

* 修改mycat的用户账号和授权信息

```shell
vim /usr/local/mycat/conf/server.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://org.opencloudb/">
        <system>
        <property name="defaultSqlParser">druidparser</property>
        </system>
        <!-- 以下设置为应用访问帐号权限 -->
        <user name="mycat">                                   <!-- 定义管理员用户，也就是连接 Mycat 的用户名 -->
                <property name="password">123456</property>  <!-- 密码 -->
                <property name="schemas">ha</property>       <!-- 定义一个逻辑库，与schema 配置文件对应 -->
        </user>

        <!-- 以下设置为应用只读帐号权限 -->
        <user name="user">
                <property name="password">123456</property>
                <property name="schemas">ha</property>       <!-- 定义一个逻辑库，与schema 配置文件对应 -->
                <property name="readOnly">true</property>
        </user>

</mycat:server>    
```

* 修改schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://org.opencloudb/">

        <!--下面schema的name为逻辑数据库 -->
        <schema name="ha" checkSQLschema="false" sqlMaxLimit="100" dataNode='dn1'>
        </schema>

        <!--dataNode的database对应物理数据库（实际存在） -->
        <dataNode name="dn1" dataHost="dthost" database="test"/>
        <dataHost name="dthost" maxCon="500" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="-1" slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="master" url="172.16.129.151:3306" user="mycat" password="123456">
                <readHost host="hostS1" url="172.16.129.152:3306" user="root" password="123456" />
        </writeHost>
        </dataHost>

</mycat:schema>
```

* 启动

```shell
/usr/local/mycat/bin/mycat start
cat /usr/local/mycat/logs/wrapper.log

# 停止
/usr/local/mycat/bin/mycat stop
# 重启
/usr/local/mycat/bin/mycat restart
```

* 结果

```shell
STATUS | wrapper  | 2022/05/18 16:18:56 | --> Wrapper Started as Daemon
STATUS | wrapper  | 2022/05/18 16:18:56 | Launching a JVM...
INFO   | jvm 1    | 2022/05/18 16:18:56 | Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=64M; support was removed in 8.0
INFO   | jvm 1    | 2022/05/18 16:18:56 | Wrapper (Version 3.2.3) http://wrapper.tanukisoftware.org
INFO   | jvm 1    | 2022/05/18 16:18:56 |   Copyright 1999-2006 Tanuki Software, Inc.  All Rights Reserved.
INFO   | jvm 1    | 2022/05/18 16:18:56 | 
INFO   | jvm 1    | 2022/05/18 16:18:57 | log4j 2022-05-18 16:18:57 [./conf/log4j.xml] load completed.
INFO   | jvm 1    | 2022/05/18 16:18:57 | MyCAT Server startup successfully. see logs in logs/mycat.log
```



* 测试

```shell
# client （我是本机跑了一个docker - mysql）
docker start my-mysql
docker exec -it my-mysql /bin/bash
```

```mysql
mysql -uroot -p123456 -h172.16.129.153 -P8066

show databases;
use ha;
show tables;

insert into demo values("php");
select * from demo;
```


# 双主一从 
### 机器规划

| 名字    | IP             | 配置           |
| ------- | -------------- | -------------- |
| master1 | 172.16.129.158 | 2CPU 2G 20硬盘 |
| master2 | 172.16.129.160 | 2CPU 2G 20硬盘 |
| slave   | 172.16.129.159 | 2CPU 2G 20硬盘 |
| mycat   | 172.16.129.161 | 2CPU 2G 20硬盘 |

**四台机器都执行的操作**

```shell
1. 联网
 vim /etc/sysconfig/network-scripts/ifcfg-ens33
 ONBOOT=yes
 service network restart

2. yum -y install vim

3. yum -y install wget

4. service firewalld stop && systemctl disable firewalld.service && service firewalld status
```

### 安装mysql8

查看我的个人博客[微服务二十一 Mysql8多主一从搭建](http://blog.caixiaoxin.cn/?p=667)

### 配置双主一从

查看我的个人博客[微服务二十一 Mysql8多主一从搭建](http://blog.caixiaoxin.cn/?p=667)



### 安装mycat

#### 1. 安装 JDK1.8

https://www.oracle.com/java/technologies/downloads/

```shell
1. 下载 jdk 并上传

mkdir /usr/java
tar xf jdk-8u333-linux-x64.tar.gz -C /usr/java
```

#### 2. 配置环境变量

```shell
vim /etc/profile.d/java.sh                        # /etc/profile.d/目录下创建java.sh 文件并定入如下内容

JAVA_HOME=/usr/java/jdk1.8.0_333
PATH=$JAVA_HOME/bin:$PATH 
CLASSPATH=$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar
export PATH JAVA_HOME CLASSPATH
```

```shell
chmod +x /etc/profile.d/java.sh
source /etc/profile.d/java.sh
java -version
```

#### 3. 下载mycat

```shell
# 废弃
tar xf Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz -C /usr/local/

# 备份 以防改错方便恢复
cp schema.xml schema.xml.bak  # 定义逻辑库，表、分片节点等内容
cp rule.xml rule.xml.bak      # 定义分片规则
cp server.xml server.xml.bak  # 定义用户以及系统相关变量
```

#### 4. 配置环境变量

```shell
vim /etc/profile.d/mycat.sh   # 在/etc/profile.d 目录下创建mycat.sh 文件，并写入如下

MYCAT_HOME=/usr/local/mycat 
PATH=$MYCAT_HOME/bin:$PATH
```

```shell
source /etc/profile.d/mycat.sh                   # 使环境变量生效
```

#### 5. mycat

* 账号授权

在master上执行即可（slave已做主从同步）

```shell
-- mysql8 之前版本 执行
GRANT ALL PRIVILEGES ON *.* TO 'mycat'@"%" IDENTIFIED BY "123456";


-- mysql8 执行
create user mycat@'%' identified by '123456';
grant all privileges on *.* to mycat@'%' with grant option;

ALTER USER 'mycat'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
```

* 修改mycat的用户账号和授权信息

```shell]
vim /usr/local/mycat/conf/server.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
        <system>
        <property name="defaultSqlParser">druidparser</property>
        </system>

        <user name="mycat">                                   <!-- 定义管理员用户，也就是连接 Mycat 的用户名 -->
                <property name="password">123456</property>  <!-- 密码 -->
                <property name="schemas">ha</property>       <!-- 定义一个逻辑库，与schema 配置文件对应 -->
        </user>


        <user name="user">
                <property name="password">123456</property>
                <property name="schemas">ha</property>
                <property name="readOnly">true</property>
        </user>
</mycat:server>
```

* 修改schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

        <schema name="ha" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
        </schema>
        <dataNode name="dn1" dataHost="host1" database="test" />
        <dataHost name="host1" maxCon="1000" minCon="10" balance="1"
                          writeType="1" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="172.16.129.158:3306" user="mycat"
                                   password="123456">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS1" url="172.16.129.159:3306" user="mycat" password="123456" />
                </writeHost>

                <writeHost host="hostM2" url="172.16.129.160:3306" user="mycat" password="123456"/>
        </dataHost>

</mycat:schema>
```

```md
balance 属性
负载均衡类型，目前的取值有 3 种：

balance=“0”, 不开启读写分离机制，所有读操作都发送到当前可用的 writeHost 上。
balance=“1”，全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡，简单的说，当双
主双从模式(M1->S1，M2->S2，并且 M1 与 M2 互为主备)，正常情况下，M2,S1,S2 都参与 select 语句的负载
均衡。
balance=“2”，所有读操作都随机的在 writeHost、readhost 上分发。
balance=“3”，所有读请求随机的分发到 wiriterHost 对应的 readhost 执行，writerHost 不负担读压力，
注意 balance=3 只在 1.4 及其以后版本有，1.3 没有。
```

```md
writeType 属性
负载均衡类型，目前的取值有 3 种：

writeType=“0”, 所有写操作发送到配置的第一个 writeHost，第一个挂了切到还生存的第二个 writeHost，
重新启动后已切换后的为准，切换记录在配置文件中:dnindex.properties .
writeType=“1”，所有写操作都随机的发送到配置的 writeHost，1.5 以后废弃不推荐。switchType 属
性
-1 表示不自动切换。
1 默认值，自动切换。
2 基于 MySQL 主从同步的状态决定是否切换。
```

```md
switchType 属性
-1 表示不自动切换
1 默认值，自动切换
2 基于 MySQL 主从同步的状态决定是否切换
心跳语句为 show slave status
3 基于 MySQL galary cluster 的切换机制（适合集群）（1.4.1）
心跳语句为 show status like ‘wsrep%’

修改dataHost的 balance=“1”
```



* ​	启动	

```shell
/usr/local/mycat/bin/mycat start
cat /usr/local/mycat/logs/wrapper.log

# 停止
/usr/local/mycat/bin/mycat stop
# 重启
/usr/local/mycat/bin/mycat restart

```

* 结果

```shell
STATUS | wrapper  | 2022/05/18 16:18:56 | --> Wrapper Started as Daemon
STATUS | wrapper  | 2022/05/18 16:18:56 | Launching a JVM...
INFO   | jvm 1    | 2022/05/18 16:18:56 | Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=64M; support was removed in 8.0
INFO   | jvm 1    | 2022/05/18 16:18:56 | Wrapper (Version 3.2.3) http://wrapper.tanukisoftware.org
INFO   | jvm 1    | 2022/05/18 16:18:56 |   Copyright 1999-2006 Tanuki Software, Inc.  All Rights Reserved.
INFO   | jvm 1    | 2022/05/18 16:18:56 | 
INFO   | jvm 1    | 2022/05/18 16:18:57 | log4j 2022-05-18 16:18:57 [./conf/log4j.xml] load completed.
INFO   | jvm 1    | 2022/05/18 16:18:57 | MyCAT Server startup successfully. see logs in logs/mycat.log
```



* 测试

```shell
# client （我是本机跑了一个docker - mysql）
docker start my-mysql
docker exec -it my-mysql /bin/bash
```

```mysql
mysql -umycat -p123456 -h172.16.129.161 -P8066

show databases;
use ha;
show tables;

insert into demo values("php");
insert into demo values("php");
insert into demo values("php");
insert into demo values("php");
select * from demo;
```



# docker 方式 安装mycat
### 机器规划

| 名字    | IP             | 配置           |
| ------- | -------------- | -------------- |
| master1 | 172.16.129.158 | 2CPU 2G 20硬盘 |
| master2 | 172.16.129.160 | 2CPU 2G 20硬盘 |
| slave   | 172.16.129.159 | 2CPU 2G 20硬盘 |
| mycat   | 172.16.129.162 | 2CPU 2G 20硬盘 |

### 安装mycat
```shell
1. mkdir -p /root/mycat-env
2. mkdir -p /root/mycat-env/conf
3. mkdir -p /root/mycat-env/logs
4. cp rule.xml /root/mycat-env/conf && cp server.xml /root/mycat-env/conf && cp schema.xml /root/mycat-env/conf && cp wrapper.conf /root/mycat-env/conf
5. vim /root/mycat-env/Dockerfile

FROM centos:7

RUN mkdir -p /usr/java

ADD  jdk-8u333-linux-x64.tar.gz /usr/java

ENV JAVA_HOME=/usr/java/jdk1.8.0_333
ENV PATH=$JAVA_HOME/bin:$PATH
ENV CLASSPATH=$JAVA_HOME/jre/lib/ext:$JAVA_HOME/lib/tools.jar


ADD Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz /usr/local

ENV MYCAT_HOME=/usr/local/mycat
ENV PATH=$MYCAT_HOME/bin:$PATH

WORKDIR /usr/local/mycat

ENV TZ Asia/Shanghai

EXPOSE 8066 9066

CMD ["/usr/local/mycat/bin/mycat", "console","&"]

6. vim /root/mycat-env/mycat.yml

version: '3'
services:
  mycat:
    build:
      context: .
      dockerfile: Dockerfile
    restart: always
    privileged: true
    image: mycat:1.6
    container_name: mycat
    hostname: mycat
    ports:
     - '8066:8066'
     - '9066:9066'
    volumes:
     - /etc/localtime:/etc/localtime
     - /root/mycat-env/conf/schema.xml:/usr/local/mycat/conf/schema.xml
     - /root/mycat-env/conf/rule.xml:/usr/local/mycat/conf/rule.xml
     - /root/mycat-env/conf/server.xml:/usr/local/mycat/conf/server.xml
     - /root/mycat-env/conf/wrapper.conf:/usr/local/mycat/conf/wrapper.conf
     - /root/mycat-env/logs:/usr/local/mycat/logs

7. docker-compose -f ./mycat.yml up -d

```
### 配置同上

### 测试方式同上



#### 问题汇总
#### 1. java.lang.RuntimeException: Unknown charsetIndex:255
```shell
1. 打开 …\ conf \ index_to_charset.properties
2. 加入255=utf8mb4
```

#### 2. java.lang.IllegalArgumentException: Invalid DataSource:0
```shell
打开schema.xml查看配置是否正确
```

#### 3. WARN [$_NIOREACTOR-0-RW] (io.mycat.backend.mysql.nio.handler.SingleNodeHandler.backConnectionErr(SingleNodeHandler.java:249)) - execute sql err : errno:1366 Incorrect string value: '\xF0\x9F\x98\x98' for column 'wifi_ssid' at row 1 con:MySQLConnection [id=3530, lastTime=1487755676098,
```shell
升级mycat 到 1.6.7
```




