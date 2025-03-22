---
layout: post
title: "Go使用qrcode包解析微信和支付宝二维码，生成一个链接（前端拿到链接即可解析成对应的支付二维码）"
date:    2022-9-15
tags: [区块链]
comments: true
author: mazezen
---



```go
github.com/makiuchi-d/gozxing
```



```go
// uploadFile
func uploadFile(c echo.Context) (error, string) {
	file, err := c.FormFile("qr_code")
	if err != nil {
		return err, ""
	}

	lastIndex := strings.LastIndex(file.Filename, ".")
	ext := file.Filename[lastIndex:]

	ext_list := []string{".png", ".jpg", ".jpeg"}
	if !utils.In(ext, ext_list) {
		return errors.New("文件只能是png|jpg|jpeg合适的图片"), ""
	}

	if file.Size*2 > MaxFileSize {
		return errors.New("文件太大，不符合要求"), ""
	}

	fi, err := file.Open()
	if err != nil {
		return err, ""
	}
	defer fi.Close()
	str := GetPaymentStr(fi).String()
	return nil, str
}


// GetPaymentStr
func GetPaymentStr(fi io.Reader) (paymentCodeUrl *gozxing.Result) {
	img, _, err := image.Decode(fi)
	if err != nil {
		ubzer.HLog.Error("解析二维码出错")
	}
	bmp, _ := gozxing.NewBinaryBitmapFromImage(img)
	qrReader := qrcode.NewQRCodeReader()
	result, err := qrReader.Decode(bmp, nil)
	if err != nil {
		ubzer.HLog.Error("解析二维码出错")
	}
	return result
}

```

