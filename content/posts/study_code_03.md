---
title: "从别人的代码中学习golang系列--03"
date: 2020-07-31T10:08:55+08:00
draft: false
author: "fan"
subtitle: "Go"
tags: ["Go"]
categories: ["Go"]
toc:
  enable: true
  auto: true
---

这篇博客还是整理从https://github.com/LyricTian/gin-admin 这个项目中学习的golang相关知识。

作者在项目中使用了 `github.com/casbin/casbin` 进行权限控制的，这个库自己之前也没有用过，正好可以通过这个项目学习一下使用。 当然这篇博客并不会对casbin的使用做非常详细的说明，感兴趣的可以去官网看具体的使用文档。

## 关于casbin

-------

### 常见访问控制模型

**ABAC**: 基于属性的访问控制。

**DAC**: 自主访问控制模型（DAC，Discretionary Access Control）是根据自主访问控制策略建立的一种模型，允许合法用户以用户或用户组的身份访问策略规定的客体，同时阻止非授权用户访问客体。拥有客体权限的用户，可以将该客体的权限分配给其他用户。

**ACL**: 　ACL是最早也是最基本的一种访问控制机制，它的原理非常简单：每一项资源，都配有一个列表，这个列表记录的就是哪些用户可以对这项资源执行CRUD中的那些操作。当系统试图访问这项资源时，会首先检查这个列表中是否有关于当前用户的访问权限，从而确定当前用户可否执行相应的操作。总得来说，ACL是一种面向资源的访问控制模型，它的机制是围绕“资源”展开的。

**RBAC**: 基于角色的访问控制(RBAC, Role Based Access Control)在用户和权限之间引入了“角色（Role）”的概念，角色解耦了用户和权限之间的关系。


### casbin

casbin使用配置文件来设置访问控制模型。在 Casbin 中, 访问控制模型被抽象为基于 PERM (Policy, Effect, Request, Matcher) 的一个文件。

Casbin中最基本、最简单的model是ACL。ACL中的model CONF为:

```
# Request definition
[request_definition]
r = sub, obj, act

# Policy definition
[policy_definition]
p = sub, obj, act

# Policy effect
[policy_effect]
e = some(where (p.eft == allow))

# Matchers
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

**Request definition**: 代表请求,上面的配置中 `r = sub, obj, act` 代表一个请求有三个标准元素：请求主体，请求对象，请求操作。

**Policy definition**: 代表策略，表示具体的权限定义的规则是什么，上面配置中 `p = sub, obj, act` 

**Policy effect**: Effect 用来判断如果一个请求满足了规则，是否需要同意请求

**Matchers**: 有请求，有规则，那么请求是否匹配某个规则，则是matcher进行判断的


#### ACL with superuser 栗子

model的配置：

```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act || r.sub == "root"
```

Policey 配置：

```
p, alice, data1, read
p, bob, data2, write
```

当我们请求：alice,data1,read
根据匹配规则，匹配的结果就是true
当我们请求：alice,data1,write
根据匹配规则，匹配的规则就是false


#### RESful (KeyMatch2) 栗子

model的配置：

```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && keyMatch2(r.obj, p.obj) && regexMatch(r.act, p.act)
```

Policy 定义：

```
p, alice, /alice_data/:resource, GET
p, alice, /alice_data2/:id/using/:resId, GET
```

当我们请求 `alice, /alice_data/hello, GET` 根据matchers规则匹配了 `p, alice, /alice_data/:resource, GET` 所以返回true
当我们请求 `alice, /alice_data/hello, POST`  根据matchers规则没有匹配到，所以返回false
 
#### RBAC 栗子

model的配置：
```
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
在这里引入了`role_definition` 角色定义，  g 用于判断哪个用户是否属于哪个角色

Policy 配置：

```
p, alice, data1, read
p, bob, data2, write
p, data2_admin, data2, read
p, data2_admin, data2, write

g, alice, data2_admin
```

当我们请求 `alice, data2, read` 根据matchers 匹配了alice 是data2_admin角色，并且 `r.obj == p.obj  && r.act == p.act` 所以返回true



## 在gin-admin项目中的使用

这里先梳理一下作者代码中对casbin的使用，因为之前看了casbin在其他几个项目中的使用，感觉都是有点乱，在在gin-admin这个项目的时候一开始也是感觉有点懵 ，没有理解怎么用，不过当把代码梳理清楚之后，感觉gin-admin作者的使用还是非常好的。

gin-admin项目中关于casbin的使用分为

1. 定义CasbinAdapter
2. 初始化casbin
3. 异步加载casbin权限


### 定义了CasbinAdapter

作者在gin-admin/internal/module/adapter/casbin.go 中定义了CasbinAdapter：

```
// CasbinAdapter casbin适配器
type CasbinAdapter struct {
	RoleModel         model.IRole
	RoleMenuModel     model.IRoleMenu
	MenuResourceModel model.IMenuActionResource
	UserModel         model.IUser
	UserRoleModel     model.IUserRole
}
```

这里的CasbinAdapter是实现了`casbin`中的`Adapter`接口，即CasbinAdapter实现了LoadPolicy，SavePolicy，AddPolicy，RemovePolicy方法，并且作者通过在LoadPolicy将用户权限和角色权限从数据库中进行加载。

### 初始化 

权限的初始化是通过下面代码：

```
func InitCasbin(adapter persist.Adapter) (*casbin.SyncedEnforcer, func(), error) {
	cfg := config.C.Casbin
	if cfg.Model == "" {
		return new(casbin.SyncedEnforcer), nil, nil
	}

	e, err := casbin.NewSyncedEnforcer(cfg.Model)
	if err != nil {
		return nil, nil, err
	}
	e.EnableLog(cfg.Debug)
	err = e.InitWithModelAndAdapter(e.GetModel(), adapter)
	if err != nil {
		return nil, nil, err
	}
	e.EnableEnforce(cfg.Enable)

	cleanFunc := func() {}
	if cfg.AutoLoad {
		e.StartAutoLoadPolicy(time.Duration(cfg.AutoLoadInternal) * time.Second)
		cleanFunc = func() {
			e.StopAutoLoadPolicy()
		}
	}

	return e, cleanFunc, nil
}
```


### 异步加载casbin权限

这个部分主要是当我们通过页面进行权限的配置后，我们需要将权限重新进行加载，这部分代码在gin-admin/internal/app/bll/impl/bll/b_casbin.go中：

```
var chCasbinPolicy chan *chCasbinPolicyItem

type chCasbinPolicyItem struct {
	ctx context.Context
	e   *casbin.SyncedEnforcer
}

func init() {
	chCasbinPolicy = make(chan *chCasbinPolicyItem, 1)
	go func() {
		for item := range chCasbinPolicy {
			err := item.e.LoadPolicy()
			if err != nil {
				logger.Errorf(item.ctx, "The load casbin policy error: %s", err.Error())
			}
		}
	}()
}

// LoadCasbinPolicy 异步加载casbin权限策略
func LoadCasbinPolicy(ctx context.Context, e *casbin.SyncedEnforcer) {
	if !config.C.Casbin.Enable {
		return
	}

	if len(chCasbinPolicy) > 0 {
		logger.Infof(ctx, "The load casbin policy is already in the wait queue")
		return
	}

	chCasbinPolicy <- &chCasbinPolicyItem{
		ctx: ctx,
		e:   e,
	}
}
```

而在我们的更改权限的接口中都会通过调用LoadCasbinPolicy将权限策略进行加载。


## 总结

关于这个项目整理了三篇文章，也学习到了很多东西，其实到这篇文章，作者整体代码自己已经树立清楚了，很多人会觉得作者的项目目录过于复杂，还有一些重复代码，在你刚开始梳理代码逻辑的时候还会感到一脸懵，但是当你耐心梳理完之后，你会发现，这样写原来会有这样或者那样的好处。作者剩余的代码就是关于web接口中的逻辑了，就不在做整理。

后面的计划是通过这次对这次代码的学习，写一个blog的web项目。同时也会找下一个开源项目代码进行学习


## 延伸阅读

* https://casbin.org/zh-CN/editor
* https://casbin.org/docs/zh-CN/overview
* https://www.cnblogs.com/yjf512/p/12200206.html
