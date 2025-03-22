---
layout: post
title: "Go编写开机自启动服务"
date:    2022-08-01
tags: [Go]
comments: true
author: mazezen
---

### 1. 基于redis 的 setnx

```go
package main

import (
	"fmt"
	"sync"
	"time"

	"github.com/go-redis/redis"
)

func incr() {
	client := redis.NewClient(&redis.Options{
		Addr:     "127.0.0.1:6379",
		Password: "root",
		DB:       0,
	})

	var lockKey = "counter_lock"
	var counterKey = "counter"

	resp := client.SetNX(lockKey, 1, time.Second*5)
	ok, err := resp.Result()
	if err != nil || !ok {
		fmt.Println(err, "lock result: ", ok)
		return
	}

	getResp := client.Get(counterKey)
	cntValue, err := getResp.Int64()
	if err == nil || err == redis.Nil {
		cntValue++
		resp := client.Set(counterKey, cntValue, 0)
		_, err := resp.Result()
		if err != nil {
			println("set value error!")
		}
	}
	println("current counter is ", cntValue)

	delResp := client.Del(lockKey)
	unlockSuccess, err := delResp.Result()
	if err == nil && unlockSuccess > 0 {
		println("unlock  success!")
	} else {
		println("unlock failed", err)
	}
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			incr()
		}()
	}
	wg.Wait()
}

```

### 2. 基于ZooKeeper

```go
package main

import (
	"time"

	"github.com/samuel/go-zookeeper/zk"
)

func main() {
	c, _, err := zk.Connect([]string{"127.0.0.1"}, time.Second)
	if err != nil {
		panic(err)
	}
	l := zk.NewLock(c, "/lock", zk.WorldACL(zk.PermAll))
	err = l.Lock()
	if err != nil {
		panic(err)
	}
	println("lock succ, do your business login")

	time.Sleep(time.Second * 5)
	l.Unlock()
	println("unlock succ, finish business logic")
}

```

### 3. 基于etcd

```go
package main

import (
	"log"

	"github.com/zieckey/etcdsync"
)

func main() {
	m, err := etcdsync.New("/mylock", 10, []string{"http://127.0.0.1:2379"})
	if m == nil || err != nil {
		panic(err)
	}
	err = m.Lock()
	if err != nil {
		log.Printf("etcdsync.lock failed")
	} else {
		log.Printf("etcd sync.lock succ")
	}

	log.Printf("get the lock, do something here.")

	err = m.Unlock()
	if err != nil {
		log.Printf("etcdsync.unlock failed")
	} else {
		log.Printf("etcdsync unlock succ")
	}

	log.Printf("unlock the lock, do something here.")
}
```

