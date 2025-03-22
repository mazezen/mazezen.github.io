---
layout: post
title: "安利一款自己开发的命令行翻译工具。command-fanyi"
date:    2024-11-05
tags: [开源项目]
comments: true
author: mazezen
---

> 安利一款自己开发的命令行翻译工具

在日常开发中，尤其是在国际化（i18n）和跨语言的技术交流中，翻译工具往往成为开发者的重要助手。为了更高效、更便捷地进行语言翻译，特别是对于程序员来说，command-fanyi 是一款非常实用的命令行翻译工具。它简洁、高效，专为开发者设计，旨在提供快速翻译体验，避免在开发过程中频繁切换应用或打开网页。

[![](https://pic1.zhimg.com/70/v2-09c39759ae39b66f3180f78ec9a6166e_1440w.avis?source=172ae18b&biz_tag=Post)](https://pic1.zhimg.com/70/v2-09c39759ae39b66f3180f78ec9a6166e_1440w.avis?source=172ae18b&biz_tag=Post)

### 工具背景与介绍
command-fanyi 是一个由 Golang 开发的命令行翻译工具。它能帮助开发者通过命令行快速获取翻译结果，支持多种语言对的实时翻译。不同于传统的图形化翻译工具，command-fanyi 采用了命令行界面，使得开发者能够在终端中迅速完成翻译任务，提升开发效率。

使用了百度翻译 API，进一步提升了翻译的速度和准确性。

### 功能特点
支持命令行翻译

通过简洁的命令行输入，用户可以轻松将需要翻译的文本传入，command-fanyi 会迅速返回翻译结果，避免了传统翻译工具的繁琐操作。
为了提高翻译的速度与质量，command-fanyi 默认使用了百度翻译 API，具有较高的翻译精度和响应速度，尤其适合中文翻译需求。
command-fanyi 是一款开源工具，开发者可以根据自身需求对其进行定制和扩展，比如添加其他翻译服务或改进某些翻译模块，完全不需要担心工具的使用限制。


安装

```shell
curl -sSL https://raw.githubusercontent.com/jeffcail/command-fanyi/refs/heads/master/install.sh | bash
```

设置百度翻译的appid 和 secret
```shell
export APPID="**********"
export SECRETKEY="**********"
```


使用示例
安装完成后，可以通过以下命令使用 command-fanyi 进行翻译：
```shell
fanyi go mod -h 
fanyi tldr git push
fanyi tldr curl
```

总结
command-fanyi 是一个功能简洁却非常高效的命令行翻译工具，特别适合程序员和开发者使用。它不仅支持多语言翻译，且通过百度翻译 API 提升了翻译的速度和准确性。无论是在开发过程中需要快速查看外文文档，还是在处理多语言项目时需要进行实时翻译，command-fanyi 都能够为开发者带来极大的便利。

如果你是一位需要频繁进行语言翻译的开发者，command-fanyi 将会是你非常值得尝试的工具。