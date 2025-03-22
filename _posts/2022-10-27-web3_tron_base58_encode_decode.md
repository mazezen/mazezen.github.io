---
layout: post
title: "Go 实现Base58编码与解码(区块链)"
date:    2022-10-27
tags: [区块链]
comments: true
author: mazezen
---


### 编码 
base58(区块链)：去掉6个容易混淆的，去掉0，大写的O、大写的I、小写的L、/、+/、+影响双击选择

### 实现
```go
var base58 = []byte("123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz")

func ReverseByte(bytes []byte) []byte {
	for i := 0; i < len(bytes)/2; i++ {
		bytes[i], bytes[len(bytes)-1-i] = bytes[len(bytes)-1-i], bytes[i]
	}
	return bytes
}

func Base58Encoding(str string) string {
	bytes := []byte(str)
	l := big.NewInt(0).SetBytes(bytes)
	var mod []byte
	for l.Cmp(big.NewInt(0)) > 0 {
		m := big.NewInt(0)
		ten58 := big.NewInt(58)
		l.DivMod(l, ten58, m)
		mod = append(mod, base58[m.Int64()])
	}

	for _, v := range bytes {
		if v != 0 {
			break
		} else if v == 0 {
			mod = append(mod, byte('1'))
		}
	}
	return string(ReverseByte(mod))
}

func Base58Decoding(str string) string {
	sBytes := []byte(str)
	r := big.NewInt(0)
	for _, v := range sBytes {
		i := bytes.IndexByte(base58, v)
		r.Mul(r, big.NewInt(58))
		r.Add(r, big.NewInt(int64(i)))
	}
	return string(r.Bytes())
}

func main() {
	src := "blog.caixiaoxin.cn"
	res := Base58Encoding(src)
	fmt.Println(res)
	decoding := Base58Decoding(res)
	fmt.Println(decoding)
}


```