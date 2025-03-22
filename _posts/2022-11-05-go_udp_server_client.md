---
layout: post
title: "Go实现udp服务端和客户端"
date:    2022-11-05
tags: [Go]
comments: true
author: mazezen
---


#### UDP

画dp 被称为用户数据报协议（UDP，User Datagram Protocol。UDP 为应用程序提供了一种无需建立连接就可以发送封装的 IP 数据包的方法。



#### 使用场景

音视频通话、游戏、工业物联网传感器等



#### Example

udp_server:

```go
func main() {
	listen, err := net.ListenUDP("udp", &net.UDPAddr{
		IP:   net.IPv4(0, 0, 0, 0),
		Port: 8888,
	})
	if err != nil {
		fmt.Printf("Listen udp err: %v", err)
		return
	}

	for {
		var data [1024]byte
		n, addr, err := listen.ReadFromUDP(data[:])
		if err != nil {
			fmt.Printf("Read failed from addr: %v, err: %v\n", addr, err)
			break
		}

		go func() {
			fmt.Printf("addr: %v data: %v count: %v\n", addr, string(data[:]), n)
			_, err := listen.WriteToUDP([]byte("received success!"), addr)
			if err != nil {
				fmt.Printf("write failed, err: %v\n", err)
			}
		}()
	}
}

```

udp_client: 

```go
func main() {
	conn, err := net.DialUDP("udp", nil, &net.UDPAddr{
		IP:   net.IPv4(127, 0, 0, 1),
		Port: 8888,
	})
	if err != nil {
		fmt.Printf("Connect failed, err: %v\n", err)
		return
	}

	for i := 0; i < 100; i++ {
		_, err := conn.Write([]byte("Hello server!"))
		if err != nil {
			fmt.Printf("Send data failed, err: %v\n", err)
			return
		}

		result := make([]byte, 1024)
		n, addr, err := conn.ReadFromUDP(result)
		if err != nil {
			fmt.Printf("Receive data err: %v", err)
			return
		}
		fmt.Printf("Receive from addr: %v, data: %v\n", addr, string(result[:n]))
	}
}

```

