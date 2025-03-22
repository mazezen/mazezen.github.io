---
layout: post
title: "Go微服务 Dockerfile ENTRYPOINT的使用 - 接受参数 读取nacos对应的配置文件"
date: 2022-3-9
tags: [Go]
comments: true
author: mazezen
---

**现在通过命令行参数传参的形式，根据传入的参数加载nacos上对应的配置文件。**

**首先要先理解nacos的使用。其次懂得使用Go语言中的flag包来解析命令行参数。获取命令行参数的形式有多种，这里只以flag包为例。**

**可以通flag.String、flag.Int等等获取对应类型的命令行参数。方法中包含三个字name、value、usage 分别是参数名、参数值、参数说明.如果忘记传什么参数可以通过--help 查看参数说明。** 

**这里以从nacos读取配置文件连接mysql和redis为例浅谈。**

话不多说直接上Demo

### 1.  打开nacos新建一个demo.yaml的配置文件

```yaml
demo:
  datasource:
    dsn: root:root@tcp(127.0.0.1:3306)/test
    showSql: true
  redis:
    host: 127.0.0.1:6379
    password: root
    db: 0
```

### 2. Demo程序

```go
var (
	viperConfig *viper.Viper
)

var (
	nacosIP2 = flag.String("ip", "127.0.0.1", "The nacos of ip address")
	port2    = flag.Int("p", 8848, "The nacos of port")
	group2   = flag.String("g", "demo", "The nacos of Group")
	cfg2     = flag.String("c", "demo.yml", "The path of configuration file")
)

type MysqlConfig struct {
	DbDsn   string
	ShowSQL bool
}

type RedisConfig struct {
	RedisAddr string
	Password  string
	RedisDb   int
}

func init() {
	flag.Parse()
}

func main() {
	ParseRemoteConfig(*nacosIP2, *port2, *group2, *cfg2)
	fmt.Println(GetMysqlConfig())
	fmt.Println(GetRedisConfig())
}

//ParseRemoteConfig 解析nacos配置文件
func ParseRemoteConfig(ip2 string, port2 int, group2 string, cfg2 string) *viper.Viper {

	viperConfig = viper.New()
	viperConfig.SetConfigType("yaml")
	viperConfig.SetConfigName(cfg2)
	serverConfigs := []constant.ServerConfig{
		{IpAddr: ip2, Port: uint64(port2)},
	}
	nacosClient, err := clients.NewConfigClient(vo.NacosClientParam{
		ClientConfig:  &constant.ClientConfig{TimeoutMs: 5000},
		ServerConfigs: serverConfigs,
	})
	if err != nil {
		logger.OmpLog.Fatal("nacos init failed :", err)
	}
	content, err := nacosClient.GetConfig(vo.ConfigParam{
		DataId: cfg2,
		Group:  group2,
	})
	if err != nil {
		logger.OmpLog.Fatal("read from nacos failed:" + content)
	}

	viperConfig.ReadConfig(strings.NewReader(content))
	return viperConfig
}

//GetMysqlConfig mysql配置
func GetMysqlConfig() MysqlConfig {
	var mc MysqlConfig
	mc.DbDsn = viperConfig.GetString("demo.datasource.dsn")
	mc.ShowSQL = viperConfig.GetBool("demo.datasource.showSql")
	return mc
}

//GetRedisConfig redis配置
func GetRedisConfig() RedisConfig {
	var rrc RedisConfig
	rrc.RedisAddr = viperConfig.GetString("demo.redis.host")
	rrc.Password = viperConfig.GetString("demo.redis.password")
	rrc.RedisDb = viperConfig.GetInt("demo.redis.db")
	return rrc
}

```

执行: 

```shell
  go run main.go -ip 127.0.0.1 -p 8848 -g demo -c demo.yaml
  
  # 或者先编译 
  go build
  ./main -ip 127.0.0.1 -p 8848 -g demo -c demo.yaml
```

即可看到接受到的命令行参数和读取到的配置信息

![image-20220309210227887](http://images.caixiaoxin.cn//image-20220309210227887.png)



![image-20220309210312996](http://images.caixiaoxin.cn//image-20220309210312996.png)

### 3. 链接mysql和redis, 修改main函数

```go
ParseRemoteConfig(*nacosIP2, *port2, *group2, *cfg2)
	//  链接mysql
	db, err := sql.Open("mysql", GetMysqlConfig().DbDsn)
	if err != nil {
		panic(err)
	}
	err = db.Ping()

	if err != nil {
		panic(err)
	}
	fmt.Println("mysql connect success...")

	// 链接redis
	rdb := redis.NewClient(&redis.Options{
		Addr:     GetRedisConfig().RedisAddr,
		Password: GetRedisConfig().Password,
		DB:       GetRedisConfig().RedisDb,
	})
	ping := rdb.Ping()

	err = ping.Err()
	if err != nil {
		panic(err)
	}
	fmt.Println("redis connect success...")
```

执行: 

```shell
  go run main.go -ip 127.0.0.1 -p 8848 -g demo -c demo.yaml
  
  # 或者先编译 
  go build
  ./main -ip 127.0.0.1 -p 8848 -g demo -c demo.yaml
```

即可看到数据库和redis连接成功

![image-20220309210510838](http://images.caixiaoxin.cn//image-20220309210510838.png)



###  四. 结合之前谈到的Dockerfile进行命令行传参

因为部署项目是结合docker进行部署的，不可能把代码上传到服务器，去执行go run 或者go build 。采用shell脚本，一键完成自动服务重载。那么就要通过dockerfile来将上面的参数传递过来。



这个脚本采用upx来帮我们对go项目进行打包

```shell
#!/usr/bin/env bash
echo "开始编译文件"
buildFileName="main"
BuildTime=`date +'%Y.%m.%d.%H:%M:%S'`
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "-w -s" -o ${buildFileName}
#go build -o main main.go
echo "编译完成, 编译时间${BuildTime}"
echo "开始压缩文件"
upx -9 ${buildFileName}
echo "完成压缩"
```



Dcokerfile

```dockerfile
FROM golang:1.17.2-alpine

# 设置工作目录
WORKDIR /root/demo

# 添加可执行文件
ADD ./main $WORKDIR


# 添加
ADD config /root/omp-go/omp-router/config
ADD log /root/omp-go/omp-router/log


# 声明服务端口
EXPOSE 9999

# 启动容器时运行的命令
ENTRYPOINT ["./demo", "-ip","127.0.0.1","-p","8848","-g","demo","-c","demo.yml"]
```



start.sh

```shell
#! /bin/bash

if [ ! -f 'main' ]; then
  echo 文件不存在! 待添加的安装包: 'main'
  exit
fi

echo "main-demo..."
sleep 3
docker stop main-demo

sleep 2
docker rm main-demo

docker rmi main-demo
echo ""

echo "main-demo packing..."
sleep 3
docker build -t main-demo .
echo ""

echo "main-demo running..."
sleep 3
docker run --name main-demo \
  -p 9804:9804 \
  -d main-demo \

docker logs -f main-demo | sed '/Started main-demoApplication/q'

echo ""


```

先进行打包，然后启动即可...