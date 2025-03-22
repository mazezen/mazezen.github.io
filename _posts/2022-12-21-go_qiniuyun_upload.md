---
layout: post
title: "golang对接七牛文件上传"
date:    2022-12-21
tags: [Go]
comments: true
author: mazezen
---


### ｜ 背景简介
在项目开发中，经常涉及到文件上传的功能，文档、pdf、图片、视频、音乐等,为了减少服务器磁盘的使用率，一般会选择将文件存储在对应的第三方服务空间中。引用七牛云，支持文件上传、文件分片上传、文件断点续传。

### ｜ 官方sdk
```shell
go get github.com/qiniu/go-sdk/v7
```

### | 文档
```http
https://developer.qiniu.com/kodo/1238/go
```

### ｜ 代码实现
##### 1. 文件上传
```go
// 文件上传
func TestUploadFile(t *testing.T) {
	var ak = core.QiNiuAK
	var sk = core.QiNiuSk
	var bucket = core.QiNiuBucket
	var url = core.QiuNiuUrl

	src, err := os.ReadFile("./img/meinv.jpeg")
	if err != nil {
		t.Fatal(err)
	}
	fileSize := len(src)

	putPolicy := storage.PutPolicy{
		Scope: bucket,
	}
	mac := qbox.NewMac(ak, sk)
	upToken := putPolicy.UploadToken(mac)
	cfg := storage.Config{
		Zone:          &storage.ZoneHuanan,
		UseCdnDomains: false,
		UseHTTPS:      false,
	}
	putExtra := storage.PutExtra{}
	formUploader := storage.NewFormUploader(&cfg)
	ret := storage.PutRet{}
	key := "go-cloud-storage/meinv.jpeg"
	err = formUploader.Put(context.Background(), &ret, upToken, key, bytes.NewReader(src), int64(fileSize), &putExtra)
	if err != nil {
		t.Fatal(err)
	}
	url2 := url + ret.Key
	fmt.Println(ret)
	fmt.Println(url2)
}
```


##### 2. 分片上传
```go
// 分片上传
func TestUploadChunkFile(t *testing.T) {
	var ak = core.QiNiuAK
	var sk = core.QiNiuSk
	var bucket = core.QiNiuBucket
	var url = core.QiuNiuUrl

	putPolicy := storage.PutPolicy{
		Scope: bucket,
	}
	mac := qbox.NewMac(ak, sk)
	upToken := putPolicy.UploadToken(mac)
	cfg := storage.Config{
		Zone:          &storage.ZoneHuanan,
		UseCdnDomains: false,
		UseHTTPS:      false,
	}
	resumeUploaderV2 := storage.NewResumeUploaderV2(&cfg)
	upHost, err := resumeUploaderV2.UpHost(ak, bucket)
	if err != nil {
		t.Fatal(err)
	}
	key := "go-cloud-storage/lala.mp4"
	// 初始化分块上传
	initPartsRet := storage.InitPartsRet{}
	err = resumeUploaderV2.InitParts(context.TODO(), upToken, upHost, bucket, key, true, &initPartsRet)
	if err != nil {
		t.Fatal(err)
	}

	fileInfo, err := os.Open("./music/lala.mp4")
	if err != nil {
		t.Fatal(err)
	}
	defer fileInfo.Close()
	fileContent, err := ioutil.ReadAll(fileInfo)
	if err != nil {
		t.Fatal(err)
	}
	fileLen := len(fileContent)
	chunkSize2 := 2 * 1024 * 1024

	num := fileLen / chunkSize2
	if fileLen%chunkSize2 > 0 {
		num++
	}

	// 分块上传
	var uploadPartInfos []storage.UploadPartInfo
	for i := 1; i <= num; i++ {
		partNumber := int64(i)
		fmt.Printf("开始上传第%v片数据", partNumber)

		var partContentBytes []byte
		endSize := i * chunkSize2
		if endSize > fileLen {
			endSize = fileLen
		}
		partContentBytes = fileContent[(i-1)*chunkSize2 : endSize]
		partContentMd5 := Md5(string(partContentBytes))
		uploadPartsRet := storage.UploadPartsRet{}
		err = resumeUploaderV2.UploadParts(context.TODO(), upToken, upHost, bucket, key, true,
			initPartsRet.UploadID, partNumber, partContentMd5, &uploadPartsRet, bytes.NewReader(partContentBytes),
			len(partContentBytes))
		if err != nil {
			t.Fatal(err)
		}
		uploadPartInfos = append(uploadPartInfos, storage.UploadPartInfo{
			Etag:       uploadPartsRet.Etag,
			PartNumber: partNumber,
		})
		fmt.Printf("结束上传第%d片数据\n", partNumber)
	}

	// 完成上传
	rPutExtra := storage.RputV2Extra{Progresses: uploadPartInfos}
	comletePartRet := storage.PutRet{}
	err = resumeUploaderV2.CompleteParts(context.TODO(), upToken, upHost, &comletePartRet, bucket, key,
		true, initPartsRet.UploadID, &rPutExtra)
	if err != nil {
		t.Fatal(err)
	}

	url2 := url + comletePartRet.Key
	fmt.Println(comletePartRet.Hash)
	fmt.Println(url2)
}
```


##### 3. 断点续传
```go
// 断点续传
func TestResumeUploadFile(t *testing.T) {
	ak := core.QiNiuAK
	sk := core.QiNiuSk
	localFile := "./music/abc.mp4"
	bucket := core.QiNiuBucket
	key := "go-cloud-storage/abc.mp4"
	url := core.QiuNiuUrl
	putPolicy := storage.PutPolicy{
		Scope: bucket,
	}
	mac := qbox.NewMac(ak, sk)
	upToken := putPolicy.UploadToken(mac)
	cfg := storage.Config{
		Zone:          &storage.ZoneHuanan,
		UseCdnDomains: false,
		UseHTTPS:      false,
	}
	resumeUploaderV2 := storage.NewResumeUploaderV2(&cfg)
	ret := storage.PutRet{}
	recorder, err := storage.NewFileRecorder(os.TempDir())
	if err != nil {
		t.Fatal(err)
	}
	putExtra := storage.RputV2Extra{
		Recorder: recorder,
	}
	err = resumeUploaderV2.PutFile(context.Background(), &ret, upToken, key, localFile, &putExtra)
	if err != nil {
		fmt.Println(err)
		return
	}
	url2 := url + ret.Key
	fmt.Println(ret)
	fmt.Println(url2)
}

```