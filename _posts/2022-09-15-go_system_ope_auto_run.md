---
layout: post
title: "Go编写开机自启动服务"
date:    2022-09-15
tags: [Go]
comments: true
author: mazezen
---


##### 需要用到的包

```shell
github.com/kardianos/service
```

```go
package main

import (
	"fmt"
	"os"
	"sync"

	"github.com/kardianos/service"
)

type program struct {
	log service.Logger
	cfg *service.Config
}

func (p *program) Start(s service.Service) error {
	go p.run()
	return nil
}

func (p *program) run() {

	fmt.Println("fsdfasdfsd")

	//这里写运行时的代码
	wg.Done()
}

func (p *program) Stop(s service.Service) error {
	return nil
}

var wg sync.WaitGroup

func main() {
	wg.Add(1)
	svcConfig := &service.Config{
		Name:        "test-service",
		DisplayName: "test-service",
		Description: "test-service",
	}
	prg := &program{}
	s, err := service.New(prg, svcConfig)
	if err != nil {
		fmt.Println(err.Error())
	}
	if len(os.Args) > 1 {
		if os.Args[1] == "install" {
			x := s.Install()
			if x != nil {
				fmt.Println("error:", x.Error())
				return
			}
			fmt.Println("服务安装成功")
			return
		} else if os.Args[1] == "uninstall" {
			x := s.Uninstall()
			if x != nil {
				fmt.Println("error:", x.Error())
				return
			}
			fmt.Println("服务卸载成功")
			return
		}
	}
	err = s.Run()
	if err != nil {
		fmt.Println(err.Error())
	}
	wg.Wait()
}

```


### Mac

```shell
go build

# 需要权限
sudo go ./main	install  # 安装服务 

sudo go ./main uninstall # 卸载服务

cd /Library/LaunchDaemons
ls # 查看安装的服务

# 启动服务
launchctl load test-service.plist

# 停止服务
launchctl unload test-service.plist

```

#### Linux

```shell
# 将上面代码交叉编译成linux可执行文件 扔到服务器上


./main install

cd /lib/systemd/system/或 /etc/systemd/system（中）
ls

# 启动
systemctl start test-service.service


# 停止
systemctl stop test-service.service


systemctl enable test-service.service        # 设置服务开机自启动            
systemctl is-enabled test-service.service    # 查询是否自启动服务             
systemctl disable test-service.service       # 取消服务器开机自启动 
systemctl list-units --type=service          # 列出正在运行的服务


# 查看系统日志
journalctl
journalctl -o short -n 50 # 精简模式查看最新50行
```

