---
layout: post
title: "golang+vue3+websocket实时推送首页数据或者站内信"
date: 2022-7-7
tags: [Go]
comments: true
author: mazezen
---

Web socket 长链接。实时推送下单任务数据首页展示

客户端与后端建立连接。实时接受数据回显到页面。

 

过程：

客户端接到数据后做两件事

* 数据页面回显
* 告诉后端，我接受到了，再次发送新的数据



`话不多说上代码`



websocket包

```shell
go get github.com/gorilla/websocket
```



```go
var (
	upgrader = websocket.Upgrader{
		ReadBufferSize:  0,
		WriteBufferSize: 0,
		CheckOrigin: func(r *http.Request) bool {
			return true
		},
	}
)

// 接口 客户端与这个接口建立连接
// GetDashBoardData
func (dbh *DashBoardHandler) GetDashBoardData(c echo.Context) error {

	ws, err := upgrader.Upgrade(c.Response(), c.Request(), nil)
	if err != nil {
		logger.RLog.Error(fmt.Sprintf("websocket connection failed "), zap.Error(err))
		return c.JSON(http.StatusOK, _system.WebSocketConnectionErr)
	}
	defer ws.Close()

	for {
		// write
		err = ws.WriteMessage(websocket.TextMessage, getDashboardData())
		if err != nil {
			logger.RLog.Error("websocket write failed")
			return c.JSON(http.StatusOK, _system.WebWriteErr)
		}
		// read
		_, msg, err := ws.ReadMessage()
		if err != nil {
			logger.RLog.Error("websocket read failed")
			return nil
		}
		//logger.RLog.Info(fmt.Sprintf("------------ msg: %v", string(msg)))
		if string(msg) == "success" {
			//logger.RLog.Info("------------------ 继续发送数据 ---------------")
			err = ws.WriteMessage(websocket.TextMessage, getDashboardData())
			if err != nil {
				logger.RLog.Error("websocket write failed")
				return c.JSON(http.StatusOK, _system.WebWriteErr)
			}
		}
	}
}

func getDashboardData() []byte {
  
  // 从数据中心拿数据
}
```


代码:

```vue
<script setup>
import { ref } from 'vue';

const data = ref({})

const getBoardData = (function () {
    let  websocket = new WebSocket("ws://localhost:7830/v1/api/dashboard/get/data")

    // 链接发生错误的回调方法
    websocket.onerror = function() {
        alert('websocket链接错误')
    }

    // 链接成功建立的回调方法
    websocket.onopen = function() {
        // alert('websocket链接成功')
    }

    // 接受到消息的回调方法
    websocket.onmessage = function(event) {
        // alert(event.data)
        data.value = JSON.parse(event.data)
        console.log(data.value);
        send();
    }
    
    // 链接关闭的回调方法
    websocket.onclose = function() {
        alert('websocket链接关闭')
    }

    // 监听窗口关闭事件， 当窗口关闭时，主动去关闭websocket链接, 防止链接还没断开就关闭窗口, server端会炮异常
    window.onbeforeunload = function() {
        websocket.close();
    }

    function send() {
        var message = "success"
        websocket.send(message)
    }
})
getBoardData()
```

