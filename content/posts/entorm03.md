---
title: "ent orm笔记2---schema使用(下)"
date: 2020-08-27T22:37:34+08:00
draft: false
author: "fan"
subtitle: "ent orm笔记2"
tags: ["orm","go"]
categories: ["go"]
toc:
  enable: true
  auto: true
---



## Indexes 索引



在前两篇的文章中，其实对于索引也有一些使用， 这里来详细看一下关于索引的使用

Indexes方法可以在一个或者多个字段上设置索引，以提高数据检索的速度或者定义数据的唯一性



在下面这个例子中，对user表的`field1` 和`field2` 字段设置了联合索引;对`first_name`和`last_name`设置了联合唯一索引; 对`field3` 设置了唯一索引。

这里需要注意对于单独的字段设置唯一索引，在Fields中定义字段的时候通过Unique方法即可

```go
package schema

import (
	"github.com/facebook/ent"
	"github.com/facebook/ent/schema/field"
	"github.com/facebook/ent/schema/index"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("field1"),
		field.String("field2"),
		field.String("field3").Unique(),
		field.String("first_name"),
		field.String("last_name"),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return nil
}

func Indexes() []ent.Index {
	return []ent.Index{
		// non-unique index.
		index.Fields("field1", "field2"),
		// unique index
		index.Fields("first_name", "last_name").Unique(),
	}
}

```



查看一下生成的表的信息：

```sql
CREATE TABLE `users` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `field1` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `field2` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `field3` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `first_name` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `last_name` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `field3` (`field3`),
  UNIQUE KEY `user_first_name_last_name` (`first_name`,`last_name`),
  KEY `user_field1_field2` (`field1`,`field2`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
```



### Index On Edges

在建立表关系的时候也可以对相应的字段设置索引，主要实在特定关系下设置字段的唯一性

![er-city-streets](https://entgo.io/assets/er_city_streets.png)



在这个例子中，我们有一个有许多街道的城市，我们希望在每个城市下设置街道名称是唯一的。



`ent/schema/city.go`

```go
package schema

import (
   "github.com/facebook/ent"
   "github.com/facebook/ent/schema/edge"
   "github.com/facebook/ent/schema/field"
)

// City holds the schema definition for the City entity.
type City struct {
   ent.Schema
}

// Fields of the City.
func (City) Fields() []ent.Field {
   return []ent.Field{
      field.String("name"),
   }
}

// Edges of the City.
func (City) Edges() []ent.Edge {
   return []ent.Edge{
      edge.To("streets", Street.Type),
   }
}
```



`ent/schema/street.go`

```go
package schema

import (
   "github.com/facebook/ent"
   "github.com/facebook/ent/schema/edge"
   "github.com/facebook/ent/schema/field"
)

// Street holds the schema definition for the Street entity.
type Street struct {
   ent.Schema
}

// Fields of the Street.
func (Street) Fields() []ent.Field {
   return []ent.Field{
      field.String("name"),
   }
}

// Edges of the Street.
func (Street) Edges() []ent.Edge {
   return []ent.Edge{
      edge.From("city", City.Type).
         Ref("streets").
         Unique(),
   }
}
```



在上一篇文章中这种用法我们已经见过，我们看一下这样创建的表信息，主要是看streets这个表：

```sql
CREATE TABLE `streets` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `city_streets` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `streets_cities_streets` (`city_streets`),
  CONSTRAINT `streets_cities_streets` FOREIGN KEY (`city_streets`) REFERENCES `cities` (`id`) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
```



在`ent/schema/street.go` 添加索引的信息后，再次查看streets表的信息，其实这里我们就是通过添加约束，使得一个城市中每个街道的名称是唯一的



```go
func (Street) Indexes() []ent.Index {
   return []ent.Index{
      index.Fields("name").
         Edges("city").
         Unique(),
   }
}
```



```sql
CREATE TABLE `streets` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `city_streets` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `street_name_city_streets` (`name`,`city_streets`),
  KEY `streets_cities_streets` (`city_streets`),
  CONSTRAINT `streets_cities_streets` FOREIGN KEY (`city_streets`) REFERENCES `cities` (`id`) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
```



## Config

可以使用Table选项为类型提供自定义表名，如下所示:

```go
package schema

import (
    "github.com/facebook/ent"
    "github.com/facebook/ent/schema/field"
)

type User struct {
    ent.Schema
}

func (User) Config() ent.Config {
    return ent.Config{
        Table: "Users",
    }
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age"),
        field.String("name"),
    }
}
```



## Mixin

Mixin允许您创建可重用的`ent.Schema`代码。`ent.Mixin` 接口如下：

```go
Mixin interface {
    // Fields returns a slice of fields to add to the schema.
    Fields() []Field
    // Edges returns a slice of edges to add to the schema.
    Edges() []Edge
    // Indexes returns a slice of indexes to add to the schema.
    Indexes() []Index
    // Hooks returns a slice of hooks to add to the schema.
    // Note that mixin hooks are executed before schema hooks.
    Hooks() []Hook
}
```



Mixin的一个常见用例是将一些常用的通用字段进行内置，如下：

```go
package schema

import (
    "time"

    "github.com/facebook/ent"
    "github.com/facebook/ent/schema/field"
    "github.com/facebook/ent/schema/mixin"
)

// -------------------------------------------------
// Mixin definition

// TimeMixin implements the ent.Mixin for sharing
// time fields with package schemas.
type TimeMixin struct{
    // We embed the `mixin.Schema` to avoid
    // implementing the rest of the methods.
    mixin.Schema
}

func (TimeMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Immutable().
            Default(time.Now),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),
    }
}

// DetailsMixin implements the ent.Mixin for sharing
// entity details fields with package schemas.
type DetailsMixin struct{
    // We embed the `mixin.Schema` to avoid
    // implementing the rest of the methods.
    mixin.Schema
}

func (DetailsMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Int("age").
            Positive(),
        field.String("name").
            NotEmpty(),
    }
}

// -------------------------------------------------
// Schema definition

// User schema mixed-in the TimeMixin and DetailsMixin fields and therefore
// has 5 fields: `created_at`, `updated_at`, `age`, `name` and `nickname`.
type User struct {
    ent.Schema
}

func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        TimeMixin{},
        DetailsMixin{},
    }
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("nickname").
            Unique(),
    }
}

// Pet schema mixed-in the DetailsMixin fields and therefore
// has 3 fields: `age`, `name` and `weight`.
type Pet struct {
    ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
    return []ent.Mixin{
        DetailsMixin{},
    }
}

func (Pet) Fields() []ent.Field {
    return []ent.Field{
        field.Float("weight"),
    }
}
```



### Builtin Mixin

`Mixin`提供了一些内置的`Mixin`，可用于将create_time和update_time字段添加到schema中。

为了使用它们，将` mixin.Time mixin` 添加到schema，如下所示:

```go
package schema

import (
    "github.com/facebook/ent"
    "github.com/facebook/ent/schema/mixin"
)

type Pet struct {
    ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
    return []ent.Mixin{
        mixin.Time{},
        // Or, mixin.CreateTime only for create_time
        // and mixin.UpdateTime only for update_time.
    }
}
```



## 延伸阅读

- https://entgo.io/