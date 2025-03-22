---
layout: post
title: "使用copier提升开发效率"
date:    2024-8-05
tags: [GO]
comments: true
author: mazezen
---

I am a copier, I copy everything from one to another

在我们使用Go语言做开发时，经常会遇到将一个源结构体复制到另一个相似的目标结构体中。

例如，从数据库查询出来用户信息model结构体，在进行一些业务逻辑处理后，需要转换成返给前端的用户信息api结构体。没遇到这个库之前的操作就是将一个结构体的每个字段复制到另一个结构体中。而这两种数据结构有很多的字段是相同的。每碰到这种相似场景，我们都需要一个一个地去复制字段。然而在实际业务中，这样的场景往往很多，如果可以减少这种重复的，琐碎的工作。将很好的提高我们的工作效率！copier库就可以做到。copier的作者jinzhu也是大名鼎鼎gorm库的作者

<a href="https://github.com/jinzhu/copier" target="_blank" rel="noopener">copier地址</a>

支持的复制类型

- Field-to-field and method-to-field copying based on matching names
- Support for copying data:
  - From slice to slice
  - From struct to slice
  - From map to map
- Field manipulation through tags:
  - Enforce field copying with `copier:"must"`
  - Override fields even when `IgnoreEmpty` is set with `copier:"override"`
  - Exclude fields from being copied with `copier:"-"`



### 安装

```shell
go get -u github.com/jinzhu/copier
```



### 可用标签

| Tag                 | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| `copier:"-"`        | Explicitly ignores the field during copying.                 |
| `copier:"must"`     | Forces the field to be copied; Copier will panic or return an error if the field is not copied. |
| `copier:"nopanic"`  | Copier will return an error instead of panicking.            |
| `copier:"override"` | Forces the field to be copied even if `IgnoreEmpty` is set. Useful for overriding existing values with empty ones |
| `FieldName`         | Specifies a custom field name for copying when field names do not match between structs. |

### 用法

#### 1. 结构体to结构体

##### 1.1 `copier:"-"` - Ignoring Fields(忽略某个字段) 

忽略了`password`字段

```go
type SqlUserSource struct {
	Id       int64  `json:"id"`
	Name     string `json:"name"`
	Address  string `json:"address"`
	Password string `json:"password"`
}

type FrontViewTarget struct {
	Id      int64  `json:"id"`
	Name    string `json:"name"`
	Address string `json:"address"`
}

func main() {

	sqlUserSource := &SqlUserSource{Id: 1, Name: "jinzhu", Address: "jinzhu.address", Password: "1212112"}
	frontViewTarget := new(FrontViewTarget)
	if err := copier.Copy(frontViewTarget, sqlUserSource); err != nil {
		panic(err)
	}
	fmt.Printf("front view target: %v\n", frontViewTarget)
}
```

##### copier方式

```go
type SqlUserSource struct {
	Id       int64  `json:"id"`
	Name     string `json:"name"`
	Address  string `json:"address"`
	Password string `json:"password"`
}

type FrontViewTarget struct {
	Id       int64  `json:"id"`
	Name     string `json:"name"`
	Address  string `json:"address"`
	Password string `json:"password" copier:"-"` // 中横线
}

func main() {

	sqlUserSource := &SqlUserSource{Id: 1, Name: "jinzhu", Address: "jinzhu.address", Password: "1212112"}
	frontViewTarget := new(FrontViewTarget)
	if err := copier.Copy(frontViewTarget, sqlUserSource); err != nil {
		panic(err)
	}
	fmt.Printf("front view target: %v\n", frontViewTarget)
}
```

##### 1.2`copier:"must"` - Enforcing Field (强制执行字段复制)

The `copier:"must"` tag forces a field to be copied, resulting in a panic or an error if the field cannot be copied.
`copier：“must”`标签强制复制字段，如果无法复制字段，则会导致崩溃或错误。

```go
type SqlUserSource struct {
	Name string `json:"name"`
}

type FrontViewTarget struct {
	Id string `json:"id" copier:"must"` // This field must be copied, or it will panic/error.
}

func main() {

	sqlUserSource := &SqlUserSource{Name: "jinzhu"}
	frontViewTarget := new(FrontViewTarget)
  
  // This will result in a panic or an error since ID is a must field but is empty in source.
	if err := copier.Copy(frontViewTarget, sqlUserSource); err != nil {
		panic(err)
	}
	fmt.Printf("front view target: %v\n", frontViewTarget)
}
```

```shell
panic: Field Id has must tag but was not copied
```

##### 1.3`copier:"must,nopanic"` - Enforcing Field Copy Without Panic (无恐慌地执行字段复制)

Similar to `copier:"must"`, but Copier returns an error instead of panicking if the field is not copied.
(但如果字段未被复制，Copier 会返回错误而不是恐慌)，经测试并没有报错

```go
type SqlUserSource struct {
	Name string `json:"name"`
}

type FrontViewTarget struct {
	Id int64 `json:"id" copier:"must,nopanic"`
}

func main() {

	sqlUserSource := &SqlUserSource{Name: "jinzhu"}
	frontViewTarget := &FrontViewTarget{}
	if err := copier.Copy(frontViewTarget, sqlUserSource); err != nil {
		log.Fatalln("Error: ", err)
	}
	fmt.Printf("front view target: %v\n", frontViewTarget)
}
```



 ##### 1.4 Specifying Custom Field Names

Use field tags to specify a custom field name when the source and destination field names do not match.
当源字段名称和目标字段名称不匹配时，使用字段标签指定自定义字段名称。

```go
type SqlUserSource struct {
	Identifier string `json:"identifier"`
}

type FrontViewTarget struct {
	ID string `json:"id" copier:"Identifier"`
}

func main() {

	sqlUserSource := &SqlUserSource{Identifier: "78651"}
	frontViewTarget := &FrontViewTarget{}
	if err := copier.Copy(frontViewTarget, sqlUserSource); err != nil {
		log.Fatalln("Error: ", err)
	}
	fmt.Printf("front view target: %v\n", frontViewTarget)
}
```



#### 2. copy struct to slice

```go
type SqlUserSource struct {
	Name    string `json:"name"`
	Address string `json:"address"`
}

type FrontViewTarget struct {
	Name    string `json:"name"`
	Address string `json:"address"`
}

func main() {

	sqlUserSource := &SqlUserSource{Name: "78651", Address: "jinzhu.address"}
	var target []FrontViewTarget
	if err := copier.Copy(&target, sqlUserSource); err != nil {
		log.Fatalln("Error: ", err)
	}
	fmt.Printf("front view target: %v\n", target)
}
```



#### 3. copy slice to slice

```go
type SqlUserSource struct {
	Name    string `json:"name"`
	Address string `json:"address"`
}

type FrontViewTarget struct {
	Name    string `json:"name"`
	Address string `json:"address"`
}

func main() {

	sqlUserSource := []SqlUserSource{
		{Name: "78651", Address: "jinzhu.address"}, 
		{Name: "786512", Address: "jinzhu.address2"},
		}
	var target []FrontViewTarget
	if err := copier.Copy(&target, sqlUserSource); err != nil {
		log.Fatalln("Error: ", err)
	}
	fmt.Printf("front view target: %v\n", target)
}
```

##### 4. copy map to map

```go
func main() {

	source := map[int]int64{3: 4, 5: 9}
	target := map[int64]int64{}
  // target必须为指针，否则报错 copy destination must be non-nil and addressable
	if err := copier.Copy(&target, source); err != nil {
		log.Fatalln(err)
	}
	fmt.Printf("%#v\n", target)
}
```

