---
layout: post
title: "Telegram 群组防机器人神器：我用 Go 写了一个开源加群验证机器人"
date:   2026-03-27
tags: [编程随笔]
comments: true
author: mazezen


---

### Telegram 群越来越难管了，尤其是加群机器人满天飞。

每天都有各种推广号、广告 bot 疯狂进群，发垃圾消息、刷屏、引流。手动踢人？累死人。找现成的验证机器人？很多要收费、要加官方群、隐私不放心，或者功能太复杂。
于是我自己用 Go 写了一个轻量级的开源加群验证机器人 —— verify-bot，现在已经开源在 GitHub 上，欢迎大家免费使用和 Star。
项目地址： https://github.com/mazezen/verify-bot

### 这个机器人到底能干什么？

当新成员加入群组时，verify-bot 会自动：

- 立刻**临时禁言**新成员，防止他们立刻发广告
- 私聊或群内向新成员发送一道**简单数学验证题**（例如：32 + 67 = ?）
- 验证**完全独立**：每个人看到的题目和按钮只有自己能点击，其他人看不到也点不了，避免作弊
- **验证通过** → 立即恢复发言权限，正常聊天
- **验证失败或超时** → 根据配置次数踢出群组

整个流程全自动，极大降低垃圾机器人和恶意用户的进入概率，同时对真实用户几乎无打扰。



为什么推荐这个项目？

- **极简轻量**：纯 Go 语言编写，二进制体积小，性能高，支持全平台编译运行。
- **部署友好**：强烈推荐使用 **Docker + docker-compose**，一行命令就能跑起来。我还写了 Makefile，Mac（M 系列）和 Ubuntu Linux 都能丝滑使用：
  - 首次启动：make docker-compose
  - 修改代码后重启：make restart
- **配置简单**：只需要准备 Bot Token，填入 config.yaml 即可。
- **安全可靠**：支持错误次数限制、超时自动踢出，验证图片（waiting_verify.jpg / verifyed.jpg）也可自定义。
- **完全开源免费**：代码透明，你可以自行修改、部署私有化，不用担心数据泄露或第三方服务突然跑路。



### 如何快速部署？

**推荐 Docker 方式**（最省事）：

1. git clone [https://github.com/mazezen/verify-bot.git](https://link.zhihu.com/?target=https%3A//github.com/mazezen/verify-bot.git)
2. 进入目录，配置好 config.yaml（填入你的 Bot Token）
3. 执行 make docker-compose

搞定！机器人就跑起来了。

想看实时日志？make logs 想重启？make restart 想停止？make down

源码部署也非常简单，几行命令就能跑。



**使用前准备**：

- 在 @BotFather 创建机器人，获取 Token
- 把机器人拉进群，并设为管理员（赋予相应权限）
- 建议群组为超级群



### 写在最后

目前项目还在持续迭代中，如果你有好的想法（比如增加更多验证题型、支持图片验证码、自定义欢迎语等），欢迎提交 Issue 或 Pull Request，一起完善它。

**GitHub 地址（求 Star ⭐）：** [https://github.com/mazezen/verify-bot](https://link.zhihu.com/?target=https%3A//github.com/mazezen/verify-bot)

有部署问题的朋友可以在评论区留言，或者直接去 GitHub 提 Issue，我会尽量帮忙。

如果你也在运营 Telegram 群，欢迎试试这个机器人。干净、简单、高效，才是好工具。

