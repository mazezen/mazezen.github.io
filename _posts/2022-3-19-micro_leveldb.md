---
layout: post
title: "Go 微服务十四 Go 使用leveldb"
date: 2022-3-19
tags: [Go]
comments: true
author: mazezen
---


## 简介

**LevelDB**是一个由Google公司所研发的键-值存储嵌入式数据库管理系统编程库



leveldb是一个写性能十分优秀的存储引擎，是典型的LSM树(Log Structured-Merge Tree)实现。LSM树的核心思想就是放弃部分读的性能，换取最大的写入能力

比较使用读少写多的一种场景.以太坊、区块链



## 特点

1. key和value都是任意长度的字节数据
2. 提供了基本的增删改查接口
3. 自动使用Snappy压缩数据
4. 通过向前或者向后迭代器遍历数据
5. 不支持sql语句， 不支持索引
6. 一次只允许一个进程访问一个特定的数据库



## 安装

由于leveldb依赖Snapy压缩，所以需要下载snappy库

小编基于goleveldb封装了一个使用<a href="https://github.com/jeffcail/leveldb" target="_blank" rel="noopener">leveldb的包</a>,放在了github上，需有兴趣可以点击左侧连接下载使用。包括Put、Get、Has、Delete、SelectAll

在项目中引用

```shell
go get github.com/jeffcail/leveldb 
```

## 使用

1 .创建leveldb连接，会在项目目录下生成一个level_data文件夹，用来保存数据

```go
var (
	db  *leveldb1.LevelDB
	err error
)

func init() {
	db, err = leveldb1.CreateLevelDB("./level_data")
	if err != nil {
		panic(err)
	}
}

```



2.保存

```go
func main() {
	save()
}

func save() {
	type User struct {
		ID   int
		Name string
		Age  int
		Sex  string
	}

	u := &User{}
	u.ID = 1
	u.Name = "jeffcail"
	u.Age = 18
	u.Sex = "男"

	db.Put("user-1", u)
  db.Put("test", "太阳上的雨天 blog.caixiaoxin.cn")
}
```

3.获取条数据

```go
func main() {
	getUser()
}
func getUser() {
	user, err := db.Get("user-1")
	if err != nil {
		panic(err)
	}

	fmt.Printf("user=====%s\n", user)
}

```

执行结果:

 user====={"ID":1,"Name":"jeffcail","Age":18,"Sex":"男"}

4.查询全部

```go
func main() {
	selectAll()
}

func selectAll() {
	iter := db.SelectAll()
	for iter.Next() {
		k := iter.Key()
		v := iter.Value()
		fmt.Printf("k: %s\n", k)
		fmt.Printf("v: %s\n", v)
	}
	iter.Release()
	err = iter.Error()
	if err != nil {
		panic(err)
	}
}
```

执行结果:

k: test
v: "太阳上的雨天 blog.caixiaoxin.cn"
k: user-1
v: {"ID":1,"Name":"jeffcail","Age":18,"Sex":"男"}



5.查询key是否存在

```go
func main() {
	has()
}

func has() {
	if ok, _ := db.Has("test"); !ok {
		fmt.Printf("key 为%s\n不存在", "test")
	}
	fmt.Printf("key 为%s\n存在", "test")
}
```

执行结果:

key 为test
存在



6.删除

```go
func main() {
	del()
}

func del() {
	err = db.Delete("test")
	if err != nil {
		panic(err)
	}
}
```

