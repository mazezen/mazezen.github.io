---
layout: post
title: "undefined: secp256k1.RecoverPubkey"
date:    2024-09-18
tags: [GO]
comments: true
author: mazezen
---


在使用 gotron-sdk开发过程中，跨平台打包编译的时候，遇到了这个问题 
```shell
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "-w -s" -o app
```

```shell
# github.com/fbsobreira/gotron-sdk/pkg/keystore
../../gopath/pkg/mod/github.com/fbsobreira/gotron-sdk@v0.0.0-20230907131216-1e824406fe8c/pkg/keystore/recover.go:17:33: undefined: secp256k1.RecoverPubkey
```

于是去网站找答案，看了两个也没有真正的解决办法。于是去提了个<a href="https://github.com/fbsobreira/gotron-sdk/issues/139" target="_blank" rel="noopener">issue139</a>
[![issue139](http://images.caixiaoxin.cn/gotron-sdk-issue139.jpg "issue139")](http://images.caixiaoxin.cn/gotron-sdk-issue139.jpg "issue139")
在提issue的时候，发现已经有三个同样的issue问题了。虽然还是 ·OPEN·状态，但还是抱着试试的心态进去看了看。在里面还真发现了解决的办法。
<a href="https://github.com/fbsobreira/gotron-sdk/pull/107" target="_blank" rel="noopener">issue107</a>
[![issue107](http://images.caixiaoxin.cn/gotron-sdk.jpg)]

其实就是版本问题。
在mod添加以下代码即可
```go
replace github.com/fbsobreira/gotron-sdk v0.0.0-20230907131216-1e824406fe8c => github.com/sunbankio/gotron-sdk v0.0.0-20231003155243-a269b0d040c3
```
