---
layout: post
title: "Go微服务搭建 mysql 主从同步"
date: 2022-4-18
tags: [Go]
comments: true
author: mazezen
---

在实际项目，当访问量大，并发量高或者业务较复杂的时候。为了优化性能，减轻一个主库mysql服务的压力，提升用户体验，会考虑分库分表或者主从模式。 项目中会存在大量的读写操作，而且读的操作可能会占很大的比例，如果写的同时使用了锁机制，那么会导致查询等待，也就导致查询很慢。为了减少这种情况的发生，可以使用从库来处理读，主库负责写。

主从又分为几种模式

* 一主一从
* 一主多从
* 多主一从
* 多主多从



这里采用一主多从的模式，实现主从数据同步。同步会有时间延迟，最终确保数据一致性。



### 原理

1. master将变动记录到二进制日志文件(binary log)中
2. master主库将二进制日志文件发送给订阅者slave
3. slave通过I/O线程读取日志文件中的内容写到relay日志中
4. slave执行relay日志中的事件，将数据存储到本地



### 注意

* 主从服务操作系统版本和位数一致
* 主库和从库 数据库版本要一致
* master要开启二进制日志
* master和slave 的 server_id 在局域网内必须是唯一的





### 环境

* 虚拟机 
* centos7
* docker
* mysql8



| 机器名字     | 配置          |
| ------------ | ------------- |
| master-msql  | 1核2G 20G硬盘 |
| slave1-mysql | 1核2G 20G硬盘 |
| slave2-mysql | 1核2G 20G硬盘 |

Note: 为了方便，这里将每个机器重新命名. master为主库机器 salve1 位从库1的机器 slave2 位从库2的机器



### 安装

1. 使用vmware fusion安装centos7 过程省略...
2. 联网、修改机器名

```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33 (名字可能会有不同)

修改
ONBOOT=NO
改为
ONBOOT=yes

systemctl restart network

ip addr 查看ip 或者ping www.baidu.com （查看是否联网）

yum -y install wget && yum -y install vim

hostnamectl set-hostname master-mysql

hostname

```

slave1-mysql  slave2-mysql 执行以上操作



### 搭建mysql

1. master-mysql

1.1 创建env目录

```shell
mkdir env

pwd
/root/env
```

1.2 进入env目录 创建init_docker.sh 、 start.sh和 yml目录

```shell
touch init_docker.sh
chmod +x init_docker.sh

touch  start.sh
chmod +x start.sh

mkdir yml
cd yml
touch mysql.yml

```

init_docker.sh

```shell
#! /bin/bash

yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io -y

echo "=====>>>>> 启动 docker..." && sleep 1
systemctl start docker

echo "=====>>>>> 启动成功..." && sleep 1

echo "=====>>>>> test docker..." && sleep 1
docker pull hello-world

docker rmi hello-world

echo "=====>>>>> install docker-compose..." && sleep 1
curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

echo "=====>>>>> test docker-compose..." && sleep 1
docker-compose --version

echo "=====>>>>> 安装成功" && sleep 1

exit
```



 start.sh

```shell
#! /bin/bash

# chmod +x ./init_docker.sh

echo "=====>>>>> 开始安装 docker..." && sleep 1 && echo ""
./init_docker.sh

echo "=====>>>>> docker安装完毕 " && sleep 1

echo "=====>>>>> 准备安装并启动 mysql " && sleep 1 && echo ""
docker-compose -f yml/mysql.yml up -d

sleep 3 && docker ps && echo ""

echo "=====>>>>> mysql启动成功 "
echo "=====>>>>> 浏览器访问7880端口，数据库密码请查询 yml/mysql.yml " && sleep 2

```



mysql.yml

```yml
version: '2'
services:
  mysql:
    container_name: mysqld
    image: mysql:8.0.19
    environment:
      MYSQL_ROOT_PASSWORD: root
      TZ: Asia/Shanghai
    command:
      --server_id=1
      --binlog_format=MIXED
      --slow_query_log='ON'
      --long_query_time=20
      --character-set_server=utf8mb4
      --collation-server=utf8mb4_general_ci
    ports:
      - 3306:3306
    volumes:
      - /ext/data/mysql:/var/lib/mysql

  adminer:
    container_name: adminer
    image: adminer:4.8.1
    restart: always
    environment:
      ADMINER_DEFAULT_SERVER: 172.17.0.1
    ports:
      - 7880:8080
```

1.3 安装

```shell
./start.sh

docker 查看

浏览器访问
ip:7880
root / root
```

1.4 navicat客户端连接





2. slave1-mysql

2.1 创建env目录

```shell
mkdir env

pwd
/root/env
```

2.2 进入env目录 创建init_docker.sh 、 start.sh和 yml目录

```shell
touch init_docker.sh
chmod +x init_docker.sh

touch  start.sh
chmod +x start.sh

mkdir yml
cd yml
touch mysql.yml

```

init_docker.sh

```shell
#! /bin/bash

yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io -y

echo "=====>>>>> 启动 docker..." && sleep 1
systemctl start docker

echo "=====>>>>> 启动成功..." && sleep 1

echo "=====>>>>> test docker..." && sleep 1
docker pull hello-world

docker rmi hello-world

echo "=====>>>>> install docker-compose..." && sleep 1
curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

echo "=====>>>>> test docker-compose..." && sleep 1
docker-compose --version

echo "=====>>>>> 安装成功" && sleep 1

exit
```



 start.sh

```shell
#! /bin/bash

# chmod +x ./init_docker.sh

echo "=====>>>>> 开始安装 docker..." && sleep 1 && echo ""
./init_docker.sh

echo "=====>>>>> docker安装完毕 " && sleep 1

echo "=====>>>>> 准备安装并启动 mysql " && sleep 1 && echo ""
docker-compose -f yml/mysql.yml up -d

sleep 3 && docker ps && echo ""

echo "=====>>>>> mysql启动成功 "
echo "=====>>>>> 浏览器访问7880端口，数据库密码请查询 yml/mysql.yml " && sleep 2

```



mysql.yml

```yml
version: '2'
services:
  mysql:
    container_name: mysqld
    image: mysql:8.0.19
    environment:
      MYSQL_ROOT_PASSWORD: root
      TZ: Asia/Shanghai
    command:
      --server_id=1000
      --binlog_format=MIXED
      --slow_query_log='ON'
      --long_query_time=20
      --character-set_server=utf8mb4
      --collation-server=utf8mb4_general_ci
    ports:
      - 3306:3306
    volumes:
      - /ext/data/mysql:/var/lib/mysql

  adminer:
    container_name: adminer
    image: adminer:4.8.1
    restart: always
    environment:
      ADMINER_DEFAULT_SERVER: 172.17.0.1
    ports:
      - 7880:8080
```

2.3 安装

```shell
./start.sh

docker 查看

浏览器访问
ip:7880
root / root
```

2.4 navicat客户端连接





3. slave2-mysql

3.1 创建env目录

```shell
mkdir env

pwd
/root/env
```

3.2 进入env目录 创建init_docker.sh 、 start.sh和 yml目录

```shell
touch init_docker.sh
chmod +x init_docker.sh

touch  start.sh
chmod +x start.sh

mkdir yml
cd yml
touch mysql.yml

```

init_docker.sh

```shell
#! /bin/bash

yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io -y

echo "=====>>>>> 启动 docker..." && sleep 1
systemctl start docker

echo "=====>>>>> 启动成功..." && sleep 1

echo "=====>>>>> test docker..." && sleep 1
docker pull hello-world

docker rmi hello-world

echo "=====>>>>> install docker-compose..." && sleep 1
curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

echo "=====>>>>> test docker-compose..." && sleep 1
docker-compose --version

echo "=====>>>>> 安装成功" && sleep 1

exit
```



 start.sh

```shell
#! /bin/bash

# chmod +x ./init_docker.sh

echo "=====>>>>> 开始安装 docker..." && sleep 1 && echo ""
./init_docker.sh

echo "=====>>>>> docker安装完毕 " && sleep 1

echo "=====>>>>> 准备安装并启动 mysql " && sleep 1 && echo ""
docker-compose -f yml/mysql.yml up -d

sleep 3 && docker ps && echo ""

echo "=====>>>>> mysql启动成功 "
echo "=====>>>>> 浏览器访问7880端口，数据库密码请查询 yml/mysql.yml " && sleep 2

```



mysql.yml

```yml
version: '2'
services:
  mysql:
    container_name: mysqld
    image: mysql:8.0.19
    environment:
      MYSQL_ROOT_PASSWORD: root
      TZ: Asia/Shanghai
    command:
      --server_id=2000
      --binlog_format=MIXED
      --slow_query_log='ON'
      --long_query_time=20
      --character-set_server=utf8mb4
      --collation-server=utf8mb4_general_ci
    ports:
      - 3306:3306
    volumes:
      - /ext/data/mysql:/var/lib/mysql

  adminer:
    container_name: adminer
    image: adminer:4.8.1
    restart: always
    environment:
      ADMINER_DEFAULT_SERVER: 172.17.0.1
    ports:
      - 7880:8080
```

3.3 安装

```shell
./start.sh

docker 查看

浏览器访问
ip:7880
root / root
```

3.4 navicat客户端连接





### 主从搭建

1. 使用客户端连接slave1-mysql

```mysql
set global sync_binlog=0;

set global innodb_flush_log_at_trx_commit = 2;

CHANGE MASTER TO 
MASTER_HOST='172.16.129.135', // 修改成主库真实内网ip
MASTER_PORT=3306,
MASTER_USER='root',
MASTER_PASSWORD='root',// 修改成主库真实密码
MASTER_LOG_FILE='binlog.000001',
MASTER_LOG_POS=155
for channel '01';  // 通道编号

start slave;

show slave STATUS;
```

```shell
docker restart mysqld
```



 看到Slave_IO_Running 和 Slave_SQL_Running 为yes 说明成功

去主库创建test库，创建user表，添加数据，刷新slave1-mysql从库即可发现已同步



2. 使用客户端连接slave2-mysql

```mysql
set global sync_binlog=0;

set global innodb_flush_log_at_trx_commit = 2;

CHANGE MASTER TO 
MASTER_HOST='172.16.129.135', // 修改成主库真实内网ip
MASTER_PORT=3306,
MASTER_USER='root',
MASTER_PASSWORD='root',// 修改成主库真实密码
MASTER_LOG_FILE='binlog.000001',
MASTER_LOG_POS=155
for channel '02';  // 通道编号

start slave;

show slave STATUS;
```

```shell
docker restart mysqld
```

看到Slave_IO_Running 和 Slave_SQL_Running 为yes 说明成功

刷新即可看到刚才创建的test库已同步



### mysql 主从卡住问题
```shell
STOP slave for channel '01';
show slave STATUS;
set global sql_slave_skip_counter=1;
start slave for channel '01';
```

### nysql 主从延迟问题
```shell
-- 设置同步状态  # 重启之后 重新设置
set global sync_binlog=0;
-- 设置同步flush
set global innodb_flush_log_at_trx_commit = 2;
```

