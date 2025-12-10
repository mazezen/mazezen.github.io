---
layout: post
title: Jenkins安装和使用
date: 2025-12-10
tags: [编程随笔]
comments: true
author: mazezen
---

## 背景

> 项目需要,重新搭建一套 Jenkins 并使用. 借此记录下来

## Jenkins

官方地址: https://www.jenkins.io/

## 一. 搭建 (Docker)

> 文档: https://www.jenkins.io/doc/book/installing/docker/

> 系统为 Linux(MacOS)

### 1. 编写 Dockerfile, 参考官方文档

```Dockerfile
FROM jenkins/jenkins:2.528.2-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
```

### 2. 编写脚本

```shell
#! /bin/bash

docker network create jenkins || true

docker build -t myjenkins-blueocean:2.528.2-1 .

docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.528.2-1
```

> 完整脚本请查看: https://github.com/mazezen/env-deploy/tree/master/Jenkins

### 3. 运行脚本

> 如果出现 502. 可以尝试多试几次或者更换稳定镜像源
>
> 查看初始密码:
> sudo docker exec ${CONTAINER_ID or CONTAINER_NAME} cat /var/jenkins_home/secrets/initialAdminPassword
>
> 浏览器打开 http://localhost:8080/
>
> Jenkins 2.400+（当前版本是 2.528.2）开始，官方 彻底移除了对 /index.html 的支持，解锁页面强制改为 http://localhost:8080/
>
> 跳过 自定义插件安装 , 推荐插件安装

## 二. 使用

### 1. 部署 Vue 项目

> 这里已 github 为例. 实际按需使用

#### ✅ 流程: 从 GitHub 拉取 Vue 项目 → 构建 → 将 dist 部署到目标服务器目录

> 构建: 在 jenkins 内构建

#### ✅ 需要安装的插件

- Publish Over SSH
- NodeJS Plugin
- Git plugin (默认已安装)
  > 插件安装完进行配置

#### 📌 配置

1. 配置 ssh (此处采用 username+password 形式)
   Jenkins → Manage Jenkins → System → Publish over SSH → SSH Servers → Add

- Passphrase: 目标服务器密码
- Name: 起个名字, 等下 Pipeline → Script 会用到
- Hostname: 目标服务器 IP
- Username: 目标服务器 登录账号
- Remote Directory: 可为空, Pipeline → Script 里填写
  > 配置 Test Configuration 验证是否能成功连上目标服务器

2. 配置 Node 版本 (此处可以配置多个 Node 版本,适应不用的项目需要)
   Jenkins → Manage Jenkins → Tools → NodeJS installations → Add NodeJS

- Name: 起个名字, 等下 Pipeline → Script 会用到 (我的: node18)
- Version: 选择打包你项目的 Node 版本 (我的: NodeJS 18.19.0)

#### ⭐ 创建 Item 开始构建部署.

- Description: Item 名称描述
- Github 项目: 填写 git 仓库地址 (我采用的 https 如果 用 git@ 需要配置 github key)
- Pipeline Script

```Pipeline
pipeline {
    agent any

    tools {
        nodejs "node18"   // 这里是你前面配置的 NodeJS Name
    }

    environment {
        GIT_REPO = "https://github.com/xxx/admin-template.git"
        TARGET_PATH = "env/nginx/html/admin-template"        // 目标服务器部署路径 (从连上服务所在的位置为起始位置: 比如我的完整路径就是 /root/env/nginx/html/admin-template)
        SERVER_HOST = "xxx.xxx.xx.xx" // 目标服务器IP
        SERVER_USER_CREDENTIAL = "18server"    // 上面配置 SSH 时的名字
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: "${GIT_REPO}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build' // 如果你的项目中为 npm run build-only 此处需要保持一致
            }
        }

        stage('Deploy to target server') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: SERVER_USER_CREDENTIAL,
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "dist/**",
                                    removePrefix: "dist",
                                    remoteDirectory: TARGET_PATH,
                                    cleanRemote: true
                                )
                            ]
                        )
                    ]
                )
            }
        }
    }
}

```

#### 🚀 完成后构建流程

以后你只要点击 Jenkins 的 Build Now ：

1. 从 GitHub 拉代码
2. 自动 npm install
3. 自动 build
4. 自动同步 dist 到目标服务器目录

### 2. 一键打包编译 + 上传 + 部署 Go 项目

> 这里已 github 为例. 实际按需使用

#### ✅ 流程: 从 GitHub 拉取 Go 项目 → 打包编译 → 将 执行程序 部署到目标服务器目录

> 构建: 在 jenkins 内打包编译

#### ✅ 需要安装的插件

- Go Plugin

#### 📌 配置 Go 版本

配置 Go 版本 (此处可以配置多个 Go 版本,适应不用的项目需要)
Jenkins → Manage Jenkins → Tools → Go installations → Add Go

- Name: 起个名字, 等下 Pipeline → Script 会用到 (我的: Go1.23-scheduler)
- Version: 选择编译你项目的 Go 版本 (我的: Go 1.23.0)

#### ⭐ 创建 Item 开始构建部署.

- Description: Item 名称描述
- 选择 Github 项目: 填写 git 仓库地址 (我采用的 https 如果 用 git@ 需要配置 github key) // 可不填
- Pipeline Script

```pipeline
pipeline {
    agent any

    tools {
        go 'Go1.23-scheduler'   // 上面步骤配置Name
    }

    options {
        timeout(time: 10, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    environment {
        BINARY_NAME = "bbb"
        REMOTE_DIR  = "bbb_dir"
        SSH_SERVER  = "18server"
    }

    stages {
        stage('拉取代码') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/master']],
                    userRemoteConfigs: [[url: 'https://github.com/xxx/xxx.git']]
                ])
            }
        }

        stage('编译') {
            steps {
                script {
                    def buildTime = sh(script: "date +'%Y.%m.%d.%H:%M:%S'", returnStdout: true).trim()

                    sh """
                        echo "开始编译 elf（Go 版本：\$(go version)）"
                        echo "构建时间: ${buildTime}"

                        CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath \\
                            -ldflags "-w -s -X 'energy/conf.BuildTime=${buildTime}'" \\
                            -o ${BINARY_NAME} .

                        echo "编译完成！"
                        echo "二进制大小："
                        ls -lh ${BINARY_NAME}
                    """
                }
            }
        }

        stage('上传到服务器') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: "${SSH_SERVER}",
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "${BINARY_NAME}", // Dockerfile, deploy.sh, config.yaml 根据你项目实际情况加
                                    remoteDirectory: "${REMOTE_DIR}",
                                    flatten: true,   // 直接覆盖旧文件
                                    cleanRemote: false
                                )
                                sshTransfer(
                                    execCommand: """
                                        cd ${REMOTE_DIR}
                                        chmod +x ${BINARY_NAME} deploy.sh
                                        echo "开始自动部署..."
                                        sudo ./deploy.sh
                                        echo "部署完成！"
                                    """,
                                    execTimeout: 120000   // 超时2分钟，防止卡住
                                )
                            ]
                        )
                    ]
                )
            }
        }
    }

    post {
        success {
            echo "========================================"
            echo "编译 + 上传 + 自动部署 全部成功！"
            echo "服务已重启，无需手动操作"
            echo "========================================"
        }
        failure {
            echo "构建或部署失败，请查看上面日志"
        }
    }
}
```

#### 🚀 完成后构建流程

以后你只要点击 Jenkins 的 Build Now ：

1. 从 GitHub 拉代码
2. 自动 go build
3. 自动上传同步
4. 自动部署
