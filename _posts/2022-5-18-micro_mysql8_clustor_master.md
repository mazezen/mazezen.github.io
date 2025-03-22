---
layout: post
title: "微服务Mysql8多主一从搭建"
date: 2022-5-18
tags: [Go]
comments: true
author: mazezen
---

## 机器规划



| 名字    | IP             | 配置           |
| ------- | -------------- | -------------- |
| master1 | 172.16.129.155 | 2CPU 2G 20硬盘 |
| master2 | 172.16.129.156 | 2CPU 2G 20硬盘 |
| slave   | 172.16.129.157 | 2CPU 2G 20硬盘 |

**三台机器都执行的操作**

```shell
1. 联网
 vim /etc/sysconfig/network-scripts/ifcfg-ens33
 ONBOOT=yes
 service network restart

2. yum -y install vim

3. yum -y install wget

4. service firewalld stop && systemctl disable firewalld.service && service firewalld status
```

### 安装 mysql8

* master1

```mysql
--server_id=1
```

* master2

```mysql
--server_id=2
```

* slave

```mysql
--server_id=10000
```

### master1 和 master2分别创建一个test数据库 demo数据表

注意: **demo 表 ID 不要设置为自增**



### 配置主从

```mysql
-- master1
-- 设置同步状态
set global sync_binlog=0;
-- 设置同步flush
set global innodb_flush_log_at_trx_commit = 2;

CHANGE MASTER TO 
MASTER_HOST='172.16.129.155',   -- 修改成主库真实内网ip
MASTER_PORT=3306,
MASTER_USER='root',
MASTER_PASSWORD='123456', -- 修改成主库真实密码
MASTER_LOG_FILE='binlog.000002',
MASTER_LOG_POS=395
for channel '01';    -- 通道编号

start slave;


-- master2

CHANGE MASTER TO 
MASTER_HOST='172.16.129.156',   -- 修改成主库真实内网ip
MASTER_PORT=3306,
MASTER_USER='root',
MASTER_PASSWORD='123456', -- 修改成主库真实密码
MASTER_LOG_FILE='binlog.000002',
MASTER_LOG_POS=395
for channel '02';    -- 通道编号

start slave;

show slave STATUS;
```

### 测试

```mysql
-- master1 test->demo 创建一条数据
-- master2 test->demo 创建一条数据
-- slave test->demo 显示两条数据
-- 即成功
```


