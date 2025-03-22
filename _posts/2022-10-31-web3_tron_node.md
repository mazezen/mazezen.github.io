---
layout: post
title: "搭建Tron 节点"
date:    2022-10-31
tags: [区块链]
comments: true
author: mazezen
---

1 **下载最新tron代码及修改配置文件**

https://github.com/tronprotocol/java-tron/releases

2. **下载配置文件**

https://github.com/tronprotocol/tron-deployment

3. 修改 rpc.port = 50051，修改node.trustNode = “0.0.0.0:50051”，修改node.listen.port = 18889，修改vm.supportConstant = true

4. **直接启动以下命令（注：一定要安装java运行环境，如果节点内容不同步请重新下载最新版本FullNode.jar）**
```shell
nohup java -Xmx14g -XX:+UseConcMarkSweepGC -jar FullNode.jar --witness -c main_net_config.conf >> /dev/null  2>&1  &
```

5. 查询同步块高度
```shell
curl -X POST http://127.0.0.1:8090/wallet/getnowblock
```
```json
{"blockID":"000000000001ba67157878c8f3a5258c9335c6de93beab40083cb8e68d66710c","block_header":{"raw_data":{"number":113255,"txTrieRoot":"0000000000000000000000000000000000000000000000000000000000000000","witness_address":"414593d27b70d21454b39ab60bf13291dae8dc0326","parentHash":"000000000001ba663ad02c7009a0602f9af39a3f0138d4e2cd81a64714cca6b7","timestamp":1530231399000},"witness_signature":"7125e312718c0a9ee7e6da3d7c80fb044869f8fc6328d056686da97eeaaaed7979953b1a15026cf48784e05fa49e9b062d1747d0107fbbc5544cd4eaf59b673701"}}
```

6. 查看同步数据
```shell
tail -f -n 100 logs/tron.log
```
