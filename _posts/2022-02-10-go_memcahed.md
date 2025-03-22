---
layout: post
title: "Go操作Memcached"
date: 2022-02-10
tags: [Go]
comments: true
author: mazezen
---


## 简介

[Memcached]是一个自由开源的，高性能，分布式内存对象缓存系统。
[Memcached]是一种基于[内存]的key-value存储，用来存储小块的任意数据（字符串、对象）。这些数据可以是数据库调用、API调用或者是页面渲染的结果

## docker-compose安装Memcached

1. docker-compose.yml

````dockerfile
version: '3'
services: 
  memcached:
    image: memcached:1.6.14
    container_name: my-memcached
    volumes:
      - ./data:/data
    ports: 
      - '11211:11211'
````

2. docker-compose up - 安装并启动

## go操作memcached

```shell
go get github.com/bradfitz/gomemcache/memcache
```

1. set 操作

   ```go
   var (
   	server = "127.0.0.1:11211"
   )
   
   func main() {
   	var err error
   
   	m := memcache.New(server)
   	if m == nil {
   		fmt.Printf("memcache new failed")
   		return
   	}
   
   	err = m.Set(&memcache.Item{
   		Key:   "car",
   		Value: []byte("blue car"),
   	})
   
   	if err != nil {
   		fmt.Printf("write to memcache failed., %s", err)
   		return
   	}
   }
   ```

2. get 操作

   ```go
   var (
   	server2 = "127.0.0.1:11211"
   )
   
   func main() {
   	var err error
   
   	m := memcache.New(server2)
   	if err != nil {
   		fmt.Printf("memcache new failed, %s", err)
   		return
   	}
   
   	it, _ := m.Get("car")
   	if string(it.Key) == "car" {
   		fmt.Println("value is ", string(it.Value))
   	} else {
   		fmt.Println("Get failed")
   	}
   }
   
   ```

3. add 操作

   ```go
   var (
   	server3 = "127.0.0.1:11211"
   )
   
   func main() {
   
   	m := memcache.New(server3)
   	if m == nil {
   		fmt.Printf("memcache new failed")
   		return
   	}
   
   	m.Add(&memcache.Item{Key: "food", Value: []byte("rice")})
   	it, err := m.Get("food")
   	if err != nil {
   		fmt.Println("Add failed")
   	} else {
   		if string(it.Key) == "food" {
   			fmt.Println("Add value is ", string(it.Value))
   		} else {
   			fmt.Println("Get failed")
   		}
   	}
   }
   ```

4. replace 操作

   ```go
   var (
   	server4 = "127.0.0.1:11211"
   )
   
   func main() {
   
   	m := memcache.New(server4)
   
   	if m == nil {
   		fmt.Printf("create memcache failed")
   		return
   	}
   
   	m.Replace(&memcache.Item{
   		Key:   "food",
   		Value: []byte("momo"),
   	})
   
   	it, err := m.Get("food")
   	if err != nil {
   		fmt.Println("replace failed")
   		return
   	}
   
   	if string(it.Key) == "food" {
   		fmt.Println("replace value is : ", string(it.Value))
   		return
   	}
   
   	fmt.Println("read failed")
   }
   ```

5. delete 操作

   ```go
   var (
   	server6 = "127.0.0.1:11211"
   )
   
   func main() {
   
   	m := memcache.New(server6)
   	if m == nil {
   		fmt.Println("create new memcached failed")
   		return
   	}
   
   	if err := m.Delete("food"); err != nil {
   		fmt.Println("delete food failed")
   		return
   	}
   
   	fmt.Println("delete success")
   
   }
   ```

6. increment 操作

   ```go
   var (
   	server7 = "127.0.0.1:11211"
   )
   
   func main() {
   	var err error
   
   	m := memcache.New(server7)
   	if m == nil {
   		fmt.Println("create new memcached failed")
   		return
   	}
   
   	if err = m.Set(&memcache.Item{
   		Key:   "number",
   		Value: []byte("1"),
   	}); err != nil {
   		fmt.Printf("set failed, %s", err)
   		return
   	}
   
   	newValue, err := m.Increment("number", 5)
   	if err != nil {
   		fmt.Printf("incr failed, %s", err)
   		return
   	}
   	fmt.Println("new value is ", newValue)
   
   }
   
   ```

7. decrement 操作

   ```go
   var (
   	server8 = "127.0.0.1:11211"
   )
   
   func main() {
   
   	var err error
   
   	m := memcache.New(server8)
   	if m == nil {
   		fmt.Println("create new memcached failed")
   		return
   	}
   
   	if err = m.Set(&memcache.Item{
   		Key:   "number2",
   		Value: []byte("5"),
   	}); err != nil {
   		fmt.Printf("set failed, %s", err)
   		return
   	}
   
   	newValue, err := m.Decrement("number2", 3)
   	if err != nil {
   		fmt.Printf("decr failed, %s", err)
   		return
   	}
   
   	fmt.Println("new value is ", newValue)
   }
   
   ```

   

   