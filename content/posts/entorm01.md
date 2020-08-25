---
title: "ent orm笔记1---快速尝鲜"
date: 2020-08-24T23:48:37+08:00
draft: false
author: "fan"
subtitle: "ent orm笔记1"
tags: ["orm","go"]
categories: ["go"]
toc:
  enable: true
  auto: true
---



前几天看到消息Facebook孵化的ORM ent转为正式项目，出去好奇，简单体验了一下，使用上自己感觉比GORM好用，于是打算把官方的文档进行整理，也算是学习一下如何使用。



## 安装

ent orm 需要使用entc命令进行自动代码生成，所以需要先安装entc:

`go get github.com/facebook/ent/cmd/entc`

关于这个系列的所有代码笔记都会放到

`github.com/peanut-pg/ent_orm_notes`



## 快速使用



### 创建schema

正常情况下应该是在自己的项目根目录执行下面的命令，我这里是因为后续会有多个例子，为了让每个例子的内容独立，所以这里会在子目录下执行该命令

`entc init User`

这个命令执行后生成如下目录结构：

```shell
└── quick_user_example
    └── ent
        ├── generate.go
        └── schema
            └── user.go
```

而schema目录下的user.go的内容也非常简单：

```go
package schema

import "github.com/facebook/ent"

// User holds the schema definition for the User entity.
type User struct {
   ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
   return nil
}

// Edges of the User.
func (User) Edges() []ent.Edge {
   return nil
}
```

### 给schema添加字段

给schema添加字段非常简单，只需要在生成`ent/schema/user.go`的`Fields`方法中添加即可，修改之后的代码如下：

```go
package schema

import (
   "github.com/facebook/ent"
   "github.com/facebook/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
   ent.Schema
}

// Fields of the User.
// 用于给 user 表定义字段
func (User) Fields() []ent.Field {
   return []ent.Field{
      field.Int("age").
         Positive(),
      field.String("name").Default("unknown"),
   }
}

// Edges of the User.
func (User) Edges() []ent.Edge {
   return nil
}
```

执行`go generate ./ent` 自动生成代码，执行命令后的目录结构为：

```shell
└── quick_user_example
    └── ent
        ├── client.go
        ├── config.go
        ├── context.go
        ├── ent.go
        ├── enttest
        │   └── enttest.go
        ├── generate.go
        ├── hook
        │   └── hook.go
        ├── migrate
        │   ├── migrate.go
        │   └── schema.go
        ├── mutation.go
        ├── predicate
        │   └── predicate.go
        ├── privacy
        │   └── privacy.go
        ├── runtime
        │   └── runtime.go
        ├── runtime.go
        ├── schema
        │   └── user.go
        ├── tx.go
        ├── user
        │   ├── user.go
        │   └── where.go
        ├── user_create.go
        ├── user_delete.go
        ├── user.go
        ├── user_query.go
        └── user_update.go

```



### 创建表到数据库



创建表并进行简单的添加数据，和查询数据：

```go
package main

import (
   "context"
   "fmt"
   "log"

   _ "github.com/go-sql-driver/mysql"
   "github.com/peanut-pg/ent_orm_notes/quick_user_example/ent"
   "github.com/peanut-pg/ent_orm_notes/quick_user_example/ent/user"
)

func main() {
   client, err := ent.Open("mysql", "root:123456@tcp(192.168.1.104:3306)/ent_orm?parseTime=True")
   if err != nil {
      log.Fatal(err)
   }
   defer client.Close()
   ctx := context.Background()
   // run the auto migration tool
   if err := client.Schema.Create(ctx); err != nil {
      log.Fatalf("failed creating schema resources:%v", err)
   }
   CreateUser(ctx, client)
   peanut, err := QueryUser(ctx, client)
   if err != nil {
      log.Fatalln(err)
   }
   log.Fatalf("query user name is:%v, aget is %v", peanut.Name, peanut.Age)
}

// CreateUser 创建用户 name=peanut, age=18
func CreateUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
   u, err := client.User.
      Create().
      SetAge(18).
      SetName("peanut").
      Save(ctx)
   if err != nil {
      return nil, fmt.Errorf("failed creating user: %v", err)
   }
   log.Println("user was created: ", u)
   return u, nil
}

// QueryUser 查询用户 where name=peanut
func QueryUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
   u, err := client.User.
      Query().
      Where(user.NameEQ("peanut")).
      // `Only` fails if no user found,
      // or more than 1 user returned.
      Only(ctx)
   if err != nil {
      return nil, fmt.Errorf("failed querying user: %v", err)
   }
   log.Println("user returned: ", u)
   return u, nil
}
```



### 创建表关系

还是用同样的方法创建Car和Group 的schema

`entc init Car Group`

分别给`ent/schema`目录下的`car.go`和`group.go`添加对应的字段信息

`car.go`文件：

```go
package schema

import (
	"github.com/facebook/ent"
	"github.com/facebook/ent/schema/field"
)

// Car holds the schema definition for the Car entity.
type Car struct {
	ent.Schema
}

// Fields of the Car.
func (Car) Fields() []ent.Field {
	return []ent.Field{
		field.String("model"),
		field.Time("registered_at"),
	}
}

// Edges of the Car.
func (Car) Edges() []ent.Edge {
	return nil
}

```

`group.go`文件：

```go
package schema

import (
   "regexp"

   "github.com/facebook/ent"
   "github.com/facebook/ent/schema/field"
)

// Group holds the schema definition for the Group entity.
type Group struct {
   ent.Schema
}

// Fields of the Group.
func (Group) Fields() []ent.Field {
   return []ent.Field{
      field.String("name").
         // regexp validation for group name.
         Match(regexp.MustCompile("[a-zA-Z_]+$")),
   }
}

// Edges of the Group.
func (Group) Edges() []ent.Edge {
   return nil
}
```



在ent orm 中给表之间建立关系是通过Edges方法实现的，我们更改`ent/schema/user.go`中的Edges方法：

```go
package schema

import (
   "github.com/facebook/ent"
   "github.com/facebook/ent/schema/edge"
   "github.com/facebook/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
   ent.Schema
}

// Fields of the User.
// 用于给 user 表定义字段
func (User) Fields() []ent.Field {
   return []ent.Field{
      field.Int("age").
         Positive(),
      field.String("name").Default("unknown"),
   }
}

// Edges of the User.
// 和Cars表建立关系
func (User) Edges() []ent.Edge {
   return []ent.Edge{
      edge.To("cars", Car.Type),
   }
}
```

执行`go generate ./ent` 自动生成代码,然后重新生成一下表结构

然后在数据中执行`show create table ent_orm.cars` 查看表的详细结构语句：

```sql
CREATE TABLE `cars` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `model` varchar(255) COLLATE utf8mb4_bin NOT NULL,
  `registered_at` timestamp NULL DEFAULT NULL,
  `user_cars` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `cars_users_cars` (`user_cars`),
  CONSTRAINT `cars_users_cars` FOREIGN KEY (`user_cars`) REFERENCES `users` (`id`) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
```

可以看出，通过在印记功能建立的外键关系

#### 外键的数据添加

```go
// CreateCars 创建 Tesla 和Ford 汽车，加该汽车属于user: peanut_pg
func CreateCars(ctx context.Context, client *ent.Client) (*ent.User, error) {
   // creating new car with model "Tesla".
   tesla, err := client.Car.
      Create().
      SetModel("Tesla").
      SetRegisteredAt(time.Now()).
      Save(ctx)
   if err != nil {
      return nil, fmt.Errorf("failed creating car: %v", err)
   }

   // creating new car with model "Ford".
   ford, err := client.Car.
      Create().
      SetModel("Ford").
      SetRegisteredAt(time.Now()).
      Save(ctx)
   if err != nil {
      return nil, fmt.Errorf("failed creating car: %v", err)
   }
   log.Println("car was created: ", ford)

   // create a new user, and add it the 2 cars.
   peanut_pg, err := client.User.
      Create().
      SetAge(18).
      SetName("peanut_pg").
      // AddCars 将车属于user peanut_pg
      AddCars(tesla, ford).
      Save(ctx)
   if err != nil {
      return nil, fmt.Errorf("failed creating user: %v", err)
   }
   log.Println("user was created: ", peanut_pg)
   return peanut_pg, nil
}
```

#### 外键的查询

代码内容如下：

```go
package main

import (
   "context"
   "fmt"
   "log"
   "time"

   _ "github.com/go-sql-driver/mysql"
   "github.com/peanut-pg/ent_orm_notes/quick_user_example/ent"
   "github.com/peanut-pg/ent_orm_notes/quick_user_example/ent/car"
   "github.com/peanut-pg/ent_orm_notes/quick_user_example/ent/user"
)

func main() {
   client, err := ent.Open("mysql", "root:123456@tcp(192.168.1.104:3306)/ent_orm?parseTime=True")
   if err != nil {
      log.Fatal(err)
   }
   defer client.Close()
   ctx := context.Background()
   // run the auto migration tool
   if err := client.Schema.Create(ctx); err != nil {
      log.Fatalf("failed creating schema resources:%v", err)
   }
   peanut_pg, err := QueryUserByName(ctx, client, "peanut_pg")
   if err != nil {
      log.Fatalln(err)
   }
   QueryCars(ctx, peanut_pg)
}

// QueryUserByName 通过name 查询
func QueryUserByName(ctx context.Context, client *ent.Client, name string) (*ent.User, error) {
   u, err := client.User.
      Query().
      Where(user.NameEQ(name)).
      // `Only` fails if no user found,
      // or more than 1 user returned.
      Only(ctx)
   if err != nil {
      return nil, fmt.Errorf("failed querying user: %v", err)
   }
   log.Println("user returned: ", u)
   return u, nil
}

// QueryCars 查询用户peanut_pg是否有Ford 这个车
func QueryCars(ctx context.Context, peanut_pg *ent.User) error {
   cars, err := peanut_pg.QueryCars().All(ctx)
   if err != nil {
      return fmt.Errorf("failed querying user cars: %v", err)
   }
   log.Println("returned cars:", cars)

   // what about filtering specific cars.
   ford, err := peanut_pg.QueryCars().
      Where(car.ModelEQ("Ford")).
      Only(ctx)
   if err != nil {
      return fmt.Errorf("failed querying user cars: %v", err)
   }
   log.Println(ford)
   return nil
}
```



#### 反向查询

在平常的查询中我们还会经常用到一些反向查询，如我们想要查询这个车所属的用户是谁，这个时候需要修改

`ent/schema/car.go` 中的Edges方法：

```go
package schema

import (
	"github.com/facebook/ent"
	"github.com/facebook/ent/schema/edge"
	"github.com/facebook/ent/schema/field"
)

// Car holds the schema definition for the Car entity.
type Car struct {
	ent.Schema
}

// Fields of the Car.
func (Car) Fields() []ent.Field {
	return []ent.Field{
		field.String("model"),
		field.Time("registered_at"),
	}
}

// Edges of the Car.
func (Car) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			// create an inverse-edge called "owner" of type `User`
			// and reference it to the "cars" edge (in User schema)
			// explicitly using the `Ref` method.
			Ref("cars").
			// setting the edge to unique, ensure
			// that a car can have only one owner.
			Unique(),
	}
}

```



先通过`QueryCarByModel` 查询一个Model=Tesla的汽车，然后通过`QueryCarUser `查看这个汽车的所属者是谁

```
// QueryCarByModel 查询car.model=Tesla
func QueryCarByModel(ctx context.Context, client *ent.Client) (*ent.Car, error) {
	car, err := client.Car.Query().
		Where(car.ModelEQ("Tesla")).
		Only(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed query car")
	}
	return car, nil
}

// QueryCarUser 查询car.model=Tesla的所属者是谁
func QueryCarUser(ctx context.Context, car *ent.Car) error {
	owner, err := car.QueryOwner().Only(ctx)
	if err != nil {
		return fmt.Errorf("failed querying car %q owner:%v", car.Model, err)
	}
	log.Printf("car %q owner: %q\n", car.Model, owner.Name)
	return nil
}
```



#### 复杂查询

在上面的关系上再添加一个用户和组的关系，分别修改`ent/schema/user.go`和`ent/schema/car.go` 的Edges方法

```go
// Edges of the User.
// 和Cars表建立关系
func (User) Edges() []ent.Edge {
   return []ent.Edge{
      edge.To("cars", Car.Type),
      // create an inverse-edge called "groups" of type `Group`
      // and reference it to the "users" edge (in Group schema)
      // explicitly using the `Ref` method.
      edge.From("groups", Group.Type).
         Ref("users"),
   }
}
```



```go
// Edges of the Car.
func (Car) Edges() []ent.Edge {
   return []ent.Edge{
      edge.From("owner", User.Type).
         // create an inverse-edge called "owner" of type `User`
         // and reference it to the "cars" edge (in User schema)
         // explicitly using the `Ref` method.
         Ref("cars").
         // setting the edge to unique, ensure
         // that a car can have only one owner.
         Unique(),
   }
}
```



执行`go generate ./ent` 自动生成代码

通过如下方法生成基础数据：

```go
// CreateGraph 创建基础数据
func CreateGraph(ctx context.Context, client *ent.Client) error {
   // first, create the users.
   a8m, err := client.User.
      Create().
      SetAge(30).
      SetName("Ariel").
      Save(ctx)
   if err != nil {
      return err
   }
   neta, err := client.User.
      Create().
      SetAge(28).
      SetName("Neta").
      Save(ctx)
   if err != nil {
      return err
   }
   // then, create the cars, and attach them to the users in the creation.
   _, err = client.Car.
      Create().
      SetModel("TeslaY").
      SetRegisteredAt(time.Now()). // ignore the time in the graph.
      SetOwner(a8m).               // attach this graph to Ariel.
      Save(ctx)
   if err != nil {
      return err
   }
   _, err = client.Car.
      Create().
      SetModel("TeslaX").
      SetRegisteredAt(time.Now()). // ignore the time in the graph.
      SetOwner(a8m).               // attach this graph to Ariel.
      Save(ctx)
   if err != nil {
      return err
   }
   _, err = client.Car.
      Create().
      SetModel("TeslaS").
      SetRegisteredAt(time.Now()). // ignore the time in the graph.
      SetOwner(neta).              // attach this graph to Neta.
      Save(ctx)
   if err != nil {
      return err
   }
   // create the groups, and add their users in the creation.
   _, err = client.Group.
      Create().
      SetName("GitLab").
      AddUsers(neta, a8m).
      Save(ctx)
   if err != nil {
      return err
   }
   _, err = client.Group.
      Create().
      SetName("GitHub").
      AddUsers(a8m).
      Save(ctx)
   if err != nil {
      return err
   }
   log.Println("The graph was created successfully")
   return nil
}
```

