---
layout: post
title: "下载YouTube视频和音频"
date: 2025-7-25
tags: [编程随笔]
comments: true
author: mazezen
---

## yt-dlp

### 安装 yt-dlp

```shell
pip3 install yt-dlp

or

brew install yt-dlp
```

### 直接下载 (自动将视频和音频合并)

```
yt-dlp https://www.youtube.com/watch?v=ML743nrkMHw&t=1s
```

### 可以加参数去下载视频或者音频

```shell
yt-dlp -f 248 https://www.youtube.com/watch?v=ML743nrkMHw&t=1s
```

> 248 为序号. 通过 yt-dlp -F url 查看

### 下载的对应分辨率的视频和音频

```shell
yt-dlp -f 248+140 https://www.youtube.com/watch?v=ML743nrkMHw&t=1s
```

### 下载最高分辨率的视频

```shell
yt-dlp -f best https://www.youtube.com/watch?v=ML743nrkMHw&t=1s
```
