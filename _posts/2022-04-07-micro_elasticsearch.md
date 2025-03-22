---
layout: post
title: "Go 微服务十六 ElasticSearch使用"
date: 2022-04-07
tags: [Go]
comments: true
author: mazezen
---

### 简介

**Elasticsearch 是一个分布式的免费开源搜索和分析引擎，适用于包括文本、数字、地理空间、结构化和非结构化数据等在内的所有类型的数据**



### 用途

1. 应用程序搜索
2. 网站搜索
3. 企业搜索
4. 日志处理和分析
5. 基础设施指标和容器检测
6. 应用程序性能检测
7. 地理空间数据分析和可视化
8. 安全分析
9. 业务分析



## 安装 - docker方式

1. 拉取镜像 

```shell
docker pull elasticsearch:7.2.0
```

2. 启动

```shell
docker run --name my-elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -d elasticsearch:7.2.0
```

3. 检查是否安装成功

```shell
curl http://localhost:9200 
# 或者
浏览器访问 localhost:9200
```

出现一下结果表示安装成功

```md
{
  "name" : "c36009ff24b3",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "VwskUpuuQgW0djgyx4r1ZA",
  "version" : {
    "number" : "7.2.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "508c38a",
    "build_date" : "2019-06-20T15:54:18.811730Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

4. 解决跨域

```shell
docker exec -it my-elasticsearch /bin/bash
vi /usr/share/elasticsearch/config/elasticsearch.yml

文件追加
http.cors.enabled: true
http.cors.allow-origin: "*"
```

5. 重启

```shell
docker restart my-elasticsearch
```

6. 安装分词器

```shell
docker exec -it my-elasticsearch /bin/bash

cd /usr/share/elasticsearch/plugins

elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.2.0/elasticsearch-analysis-ik-7.2.0.zip

exit && docker restart my-elasticsearch
```

## 安装 kibana - docker方式

1. 拉取镜像

```shell
docker pull kibana:7.2.0
```

2. 启动

```shell
docker run --name kibana --link=my-elasticsearch:test  -p 5601:5601 -d kibana:7.2.0
```

--link  链接elasticsearch容器

问题: Kibana server is not ready yet

```shell
docker exec -it kibana /bin/bash
vi /usr/share/kibana/config/kibana.yml

# 将
[ "http://elasticsearch:9200" ] 替换成 [ "http://172.17.0.2:9200" ]


#
# sticsearch** THIS IS AN AUTO-GENERATED FILE **
#

# Default Kibana configuration for docker target
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://172.17.0.2:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true
````
重启
```shell
docker restart kibana
```
172.17.0.2 为 my-elasticsearch容器的ip
通过docker inspect  my-elasticsearch即可查看


