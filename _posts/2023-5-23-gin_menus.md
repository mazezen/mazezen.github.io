---
layout: post
title: "Go 将无限分类扁平化数据转换为树状结构 ,可用于RBAC权限控制等"
date:    2023-5-23
tags: [Gin]
comments: true
author: mazezen
---


#### 案例用到的包

```go
go get github.com/gin-gonic/gin
go get -u gorm.io/gorm
go get gorm.io/driver/mysql
```



#### 数据库结构

```mysql
CREATE TABLE `menu` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `pid` int DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `url` varchar(255) DEFAULT NULL,
  `icon` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;


INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (1, 0, '系统首页', '', NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (2, 0, '系统管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (3, 2, '用户管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (4, 2, '角色管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (5, 2, '菜单管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (6, 2, '接口管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (7, 0, '商品管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (8, 7, '鞋子管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (9, 7, '衣服管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (10, 9, '连衣裙管理', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (11, 1, '数据面板', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (12, 10, '碎花连衣裙', NULL, NULL);
INSERT INTO `demo`.`menu` (`id`, `pid`, `name`, `url`, `icon`) VALUES (13, 10, '星星连衣裙', NULL, NULL);
```

### main.go

```go
func main() {

	r := gin.Default()

	r.GET("/menu", handler.List)

	r.Run(":9999")

}
```



#### go struct 结构

model/menu.go

```go
type Menu struct {
	Id   int    `json:"id"`
	Pid  int    `json:"pid"`
	Name string `json:"name"`
	Url  string `json:"url"`
	Icon string `json:"icon"`
}
```



#### 用法

handler/menu.go

```go
type TreeList struct {
	model.Menu
	Children []*TreeList
}

func List(c *gin.Context) {

	db := mysql.InitDb()
	menus := make([]*model.Menu, 0)
	db.Find(&menus)

	var nodes []*TreeList
	for _, item := range menus {
		node := &TreeList{
			Menu: model.Menu{
				Id:   item.Id,
				Pid:  item.Pid,
				Name: item.Name,
				Url:  item.Url,
				Icon: item.Icon,
			},
			Children: nil,
		}
		nodes = append(nodes, node)
	}

	treeLists := Tree(nodes, 0)

	c.JSON(http.StatusOK, gin.H{
		"msg":  "success",
		"data": treeLists,
	})
}

func Tree(node []*TreeList, pid int) []*TreeList {
	res := make([]*TreeList, 0)
	for _, v := range node {
		if v.Pid == pid {
			v.Children = Tree(node, v.Id)
			res = append(res, v)
		}
	}
	return res
}
```



#### db

```go
func InitDb() *gorm.DB {
	dsn := "root:123456@tcp(127.0.0.1:3306)/demo?charset=utf8mb4&parseTime=True&loc=Local"
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		SkipDefaultTransaction: false,
		NamingStrategy: schema.NamingStrategy{
			SingularTable: true, 
		},
		Logger:                                   logger.Default.LogMode(logger.Info),
		DisableAutomaticPing:                     false,
		DisableForeignKeyConstraintWhenMigrating: true,
	})
	if err != nil {
		return nil
	}

	return db
}
```



#### 效果

Localhost:9999/menu

```json
{
    "data":[
        {
            "id":1,
            "pid":0,
            "name":"系统首页",
            "url":"",
            "icon":"",
            "Children":[
                {
                    "id":11,
                    "pid":1,
                    "name":"数据面板",
                    "url":"",
                    "icon":"",
                    "Children":[

                    ]
                }
            ]
        },
        {
            "id":2,
            "pid":0,
            "name":"系统管理",
            "url":"",
            "icon":"",
            "Children":[
                {
                    "id":3,
                    "pid":2,
                    "name":"用户管理",
                    "url":"",
                    "icon":"",
                    "Children":[

                    ]
                },
                {
                    "id":4,
                    "pid":2,
                    "name":"角色管理",
                    "url":"",
                    "icon":"",
                    "Children":[

                    ]
                },
                {
                    "id":5,
                    "pid":2,
                    "name":"菜单管理",
                    "url":"",
                    "icon":"",
                    "Children":[

                    ]
                },
                {
                    "id":6,
                    "pid":2,
                    "name":"接口管理",
                    "url":"",
                    "icon":"",
                    "Children":[

                    ]
                }
            ]
        },
        {
            "id":7,
            "pid":0,
            "name":"商品管理",
            "url":"",
            "icon":"",
            "Children":[
                {
                    "id":8,
                    "pid":7,
                    "name":"鞋子管理",
                    "url":"",
                    "icon":"",
                    "Children":[

                    ]
                },
                {
                    "id":9,
                    "pid":7,
                    "name":"衣服管理",
                    "url":"",
                    "icon":"",
                    "Children":[
                        {
                            "id":10,
                            "pid":9,
                            "name":"连衣裙管理",
                            "url":"",
                            "icon":"",
                            "Children":[
                                {
                                    "id":12,
                                    "pid":10,
                                    "name":"碎花连衣裙",
                                    "url":"",
                                    "icon":"",
                                    "Children":[

                                    ]
                                },
                                {
                                    "id":13,
                                    "pid":10,
                                    "name":"星星连衣裙",
                                    "url":"",
                                    "icon":"",
                                    "Children":[

                                    ]
                                }
                            ]
                        }
                    ]
                }
            ]
        }
    ],
    "msg":"success"
}
```



