---
layout: post
title: "Go+echo+xorm+casbin实现权限管理"
date: 2022-07-01
tags: [Go]
comments: true
author: mazezen
---


#### 权限管理介绍

一般指根据系统设置的安全规则或者安全策略 ，用户可以访问而且只能访问自己被授权的资源，不多不少



#### 权限管理分类

* 功能权限 （api权限）
* 数据权限



### Casbin

Casbin 是一个强大的、高效的开源访问控制框架，其权限管理机制支持多种访问控制模型。



#### Casbin是什么?

Casbin 可以：

1. 支持自定义请求的格式，默认的请求格式为`{subject, object, action}`。
2. 具有访问控制模型model和策略policy两个核心概念。
3. 支持RBAC中的多层角色继承，不止主体可以有角色，资源也可以具有角色。
4. 支持内置的超级用户 例如：`root` 或 `administrator`。超级用户可以执行任何操作而无需显式的权限声明。
5. 支持多种内置的操作符，如 `keyMatch`，方便对路径式的资源进行管理，如 `/foo/bar` 可以映射到 `/foo*`

Casbin 不能：

1. 身份认证 authentication（即验证用户的用户名和密码），Casbin 只负责访问控制。应该有其他专门的组件负责身份认证，然后由 Casbin 进行访问控制，二者是相互配合的关系。
2. 管理用户列表或角色列表。 Casbin 认为由项目自身来管理用户、角色列表更为合适， 用户通常有他们的密码，但是 Casbin 的设计思想并不是把它作为一个存储密码的容器。 而是存储RBAC方案中用户和角色之间的映射关系。

`以上内容来自casbin官网文档`



#### Casbin如何工作 

在 Casbin 中, 访问控制模型被抽象为基于 PERM (Policy, Effect, Request, Matcher) 的一个文件。因此，切换或升级项目的授权机制与修改配置一样简单。



#### Model语法

- Model CONF 至少应包含四个部分: `[request_definition], [policy_definition], [policy_effect], [matchers]`。
- 如果 model 使用 RBAC, 还需要添加`[role_definition]`部分。
- Model CONF 文件可以包含注释。注释以 `#` 开头， `#` 会注释该行剩余部分。



#### 举个例子 以(RBAC)为例

编写一个rbac_mode.conf配置文件:

```conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

Sub, obj, act 是啥？

官网介绍: 

`sub, obj, act` 表示经典三元组: 访问实体 (Subject)，访问资源 (Object) 和访问方法 (Action)。 但是, 你可以自定义你自己的请求表单, 如果不需要指定特定资源，则可以这样定义 `sub、act` ，或者如果有两个访问实体, 则为 `sub、sub2、obj、act`。

官网介绍的也是比较抽象的一个定义。

可以把sub当作用户角色, obj为资源对象(比如api路径), act当作请求行为方法（比如get 和 post等）



#### 这里以RBAC和功能权限为例

分类用户、角色、权限（菜单）

1.安装

```vim 
go get github.com/casbin/casbin/v2
```

2.编写brace_model.conf

3.连接mysql

```go
adapter := xormadapter.NewAdapter("mysql", "root:123456@tcp(127.0.0.1:3306)/demo?charset=utf8mb4", true)
	enforcer := casbin.NewEnforcer("./rbac_models.conf", adapter)
	err := enforcer.LoadPolicy()
	if err != nil {
		logger.RLog.Error("enforcer load policy err", zap.Error(err))
	}
```



4.添加权限

```go
if ok := enforcer.AddPolicy(roleName, url, method); !ok {
  log.Fatal("添加权限失败")
} else {
  log.Fatal("添加权限成功")
}
```

5.删除权限

```go
if ok := enforcer.RemovePolicy(roleName, url, method); !ok {
  log.Fatal("删除权限失败")
} else {
  log.Fatal("删除权限成功")
}
```

6.中间件校验权限

```go
func CheckPermission() echo.MiddlewareFunc {
	return func(handlerFunc echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			obj := c.Request().URL.RequestURI()
			act := c.Request().Method
			sub := "admin"
			
			if ok := _casbin.Enforcer.Enforce(sub, obj, act); !ok {
				return c.JSON(http.StatusOK, _auth.PermissionInterception)
			}
			return handlerFunc(c)
		}
	}
}
```

7.路由引入中间件

