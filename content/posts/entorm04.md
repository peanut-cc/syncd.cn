---
title: "ent orm笔记2---Code Generation"
date: 2020-08-31T15:08:42+08:00
draft: true
author: "fan"
subtitle: "ent orm笔记2"
tags: ["orm","go"]
categories: ["go"]
toc:
  enable: true
  auto: true
---



在前面几篇文章中，我们经常使用的可能就是entc这个命令了，entc这个工具给带来了很多功能，这篇文章主要整理关于ent orm 中Code Generation



## 序言

### Initialize A New Schema

通过类似如下命令可以生成Schema 模板：

```shell
entc init User Pet
```

init 将在ent/schema 目录下创建两个schema user.go 和 pet.go ,如果ent目录不存在，则会创建



### Generate Assets

在添加了fields 和 edges 后，可以在项目的根目录运行entc generate 或者使用go generate 生成代码

```shell
go generate ./ent
```



Generate 命令生成一下内容：

- 用于与graph 交互的Client 和Tx对象
- 每种schema 的CRUD生成器
- 每个schema类型的Entity对象
- 用于与构建起交互的常量和断言
- SQL方言的migrate 包

### Version Compatibility Between `entc` And `ent`



这里主要是关于在项目中使用ent 的时候ent的版本要和entc的包的版本相同，并且项目中使用Go modules 进行包管理



### Code Generation Options



要了解更多关于 codegen 选项的信息，entc generate -h ：

```shell
generate go code for the schema directory

Usage:
  entc generate [flags] path

Examples:
  entc generate ./ent/schema
  entc generate github.com/a8m/x

Flags:
      --header string                           override codegen header
  -h, --help                                    help for generate
      --idtype [int int64 uint uint64 string]   type of the id field (default int)
      --storage string                          storage driver to support in codegen (default "sql")
      --target string                           target directory for codegen
      --template strings                        external templates to execute

```



### Storage选项

entc 可以为 SQL 和 Gremlin 方言生成资产。

### External Templates

接受要执行的外部 Go 模板。如果模板名称已经由 entc 定义，它将覆盖现有的名称。否则，它将把执行输出写入与模板同名的文件。Flag 格式支持如下文件、目录和 glob:

```shell
entc generate --template <dir-path> --template glob="path/to/*.tmpl" ./ent/schema
```

更多的信息和例子可以在外部模板文档中找到 

[external templates doc]: https://entgo.io/docs/templates/

### Use `entc` As A Package

运行 entc 的另一个选项是将其作为一个包使用，如下所示:

```go
package main

import (
    "log"

    "github.com/facebook/ent/entc"
    "github.com/facebook/ent/entc/gen"
    "github.com/facebook/ent/schema/field"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{
        Header: "// Your Custom Header",
        IDType: &field.TypeInfo{Type: field.TypeInt},
    })
    if err != nil {
        log.Fatal("running ent codegen:", err)
    }
}
```

### Schema Description



如果想要得到我们定义的schema的描述信息，可以通过如下命令：

```shell
entc describe ./ent/schema
```



以之前的例子中执行效果如下：

````shell
User:
        +-------+--------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
        | Field |  Type  | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
        +-------+--------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
        | id    | <nil>  | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
        | name  | string | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
        +-------+--------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
        +-----------+------+---------+-----------+----------+--------+----------+
        |   Edge    | Type | Inverse |  BackRef  | Relation | Unique | Optional |
        +-----------+------+---------+-----------+----------+--------+----------+
        | followers | User | true    | following | M2M      | false  | true     |
        | following | User | false   |           | M2M      | false  | true     |
        +-----------+------+---------+-----------+----------+--------+----------+

````



## CRUD API



### Create A New Client

**MySQL**

```go
package main

import (
    "log"

    "<project>/ent"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    client, err := ent.Open("mysql", "<user>:<pass>@tcp(<host>:<port>)/<database>?parseTime=True")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
}
```



**PostgreSQL**

```go
package main

import (
    "log"

    "<project>/ent"

    _ "github.com/lib/pq"
)

func main() {
    client, err := ent.Open("postgres","host=<host> port=<port> user=<user> dbname=<database> password=<pass>")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
}
```



**SQLite**

```go
package main

import (
    "log"

    "<project>/ent"

    _ "github.com/mattn/go-sqlite3"
)

func main() {
    client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
}
```



**Gremlin (AWS Neptune)**

```go
package main

import (
    "log"

    "<project>/ent"
)

func main() {
    client, err := ent.Open("gremlin", "http://localhost:8182")
    if err != nil {
        log.Fatal(err)
    }
}
```



### Create An Entity

**Save** a user.

```go
a8m, err := client.User.    // UserClient.
    Create().               // User create builder.
    SetName("a8m").         // Set field value.
    SetNillableAge(age).    // Avoid nil checks.
    AddGroups(g1, g2).      // Add many edges.
    SetSpouse(nati).        // Set unique edge.
    Save(ctx)               // Create and return.
```



**SaveX** a pet; Unlike **Save**, **SaveX** panics if an error occurs.

```go
pedro := client.Pet.    // PetClient.
    Create().           // Pet create builder.
    SetName("pedro").   // Set field value.
    SetOwner(a8m).      // Set owner (unique edge).
    SaveX(ctx)          // Create and return.
```



### Create Many

**Save** a bulk of pets

```go
names := []string{"pedro", "xabi", "layla"}
bulk := make([]*ent.PetCreate, len(names))
for i, name := range names {
    bulk[i] = client.Pet.Create().SetName(name).SetOwner(a8m)
}
pets, err := client.Pet.CreateBulk(bulk...).Save(ctx)
```



### Update One

更新一个从数据库返回的entity

```go
a8m, err = a8m.Update().    // User update builder.
    RemoveGroup(g2).        // Remove specific edge.
    ClearCard().            // Clear unique edge.
    SetAge(30).             // Set field value
    Save(ctx)               // Save and return.
```



### Update By ID

```go
pedro, err := client.Pet.   // PetClient.
    UpdateOneID(id).        // Pet update builder.
    SetName("pedro").       // Set field name.
    SetOwnerID(owner).      // Set unique edge, using id.
    Save(ctx)               // Save and return.
```



### Update Many

以断言进行过滤

```go
n, err := client.User.          // UserClient.
    Update().                   // Pet update builder.
    Where(                      //
        user.Or(                // (age >= 30 OR name = "bar") 
            user.AgeEQ(30),     //
            user.Name("bar"),   // AND
        ),                      //  
        user.HasFollowers(),    // UserHasFollowers()  
    ).                          //
    SetName("foo").             // Set field name.
    Save(ctx)                   // exec and return.
```



通过edge 断言进行查询

```go
n, err := client.User.          // UserClient.
    Update().                   // Pet update builder.
    Where(                      // 
        user.HasFriendsWith(    // UserHasFriendsWith (
            user.Or(            //   age = 20
                user.Age(20),   //      OR
                user.Age(30),   //   age = 30
            )                   // )
        ),                      //
    ).                          //
    SetName("a8m").             // Set field name.
    Save(ctx)                   // exec and return.
```



### Query The Graph

获取所有用户的关注者

```go
users, err := client.User.      // UserClient.
    Query().                    // User query builder.
    Where(user.HasFollowers()). // filter only users with followers.
    All(ctx)                    // query and return.
```



获取特定用户的所有跟随者; 从graph中的一个节点开始遍历

```go
users, err := a8m.
    QueryFollowers().
    All(ctx)
```



获取所有宠物的名字

```go
names, err := client.Pet.
    Query().
    Select(pet.FieldName).
    Strings(ctx)
```



获取所有宠物的名字和年龄

```go
var v []struct {
    Age  int    `json:"age"`
    Name string `json:"name"`
}
err := client.Pet.
    Query().
    Select(pet.FieldAge, pet.FieldName).
    Scan(ctx, &v)
if err != nil {
    log.Fatal(err)
}
```

### Delete One

这个用于如果我们已经通过client查询到了一个entity，然后想要删除这条记录：

```go
err := client.User.
    DeleteOne(a8m).
    Exec(ctx)
```



Delete by ID.

```go
err := client.User.
    DeleteOneID(id).
    Exec(ctx)
```



### Delete Many



使用断言进行删除

```go
err := client.File.
    Delete().
    Where(file.UpdatedAtLT(date))
    Exec(ctx)
```



### Mutation

 通过 entc init 生成的每个schema 都有自己的mutaion，例如我们通过 entc init User Pet, 在通过go generate ./ent 生成的代码中有 `ent/mutation.go` 

在该文件中定义了：

```go
.....
// UserMutation represents an operation that mutate the Users
// nodes in the graph.
type UserMutation struct {
	config
	op            Op
	typ           string
	id            *int
	name          *string
	age           *int
	addage        *int
	clearedFields map[string]struct{}
	done          bool
	oldValue      func(context.Context) (*User, error)
}

.....

// PetMutation represents an operation that mutate the Pets
// nodes in the graph.
type PetMutation struct {
	config
	op            Op
	typ           string
	id            *int
	name          *string
	age           *int
	addage        *int
	clearedFields map[string]struct{}
	done          bool
	oldValue      func(context.Context) (*Pet, error)
}
```



例如，所有的[`User` builders](https://entgo.io/docs/crud#create-an-entity)都共享相同的UserMutaion 对象，左右的builder 类型都集成通用的[`ent.Mutation`](https://pkg.go.dev/github.com/facebook/ent?tab=doc#Mutation)接口.

这里所说的 user builders，拿User schema来说指的是`UserCreate`、`UserDelete`、`UserQuery`、`UserUpdate` 对象，go generate 生成的代码中，我们可以到

`./ent/user_create.go、./ent/user_delete.go、./ent/user_query.go、./ent/user_update.go`文件中看到如下定义：

```go
// ./ent/user_create.go

// UserCreate is the builder for creating a User entity.
type UserCreate struct {
	config
	mutation *UserMutation
	hooks    []Hook
}


//./ent/user_delete.go
// UserDelete is the builder for deleting a User entity.
type UserDelete struct {
	config
	hooks      []Hook
	mutation   *UserMutation
	predicates []predicate.User
}

// ./ent/user_query.go
// UserQuery is the builder for querying User entities.
type UserQuery struct {
	config
	limit      *int
	offset     *int
	order      []OrderFunc
	unique     []string
	predicates []predicate.User
	// intermediate query (i.e. traversal path).
	sql  *sql.Selector
	path func(context.Context) (*sql.Selector, error)
}

// ./ent/user_update.go
// UserUpdate is the builder for updating User entities.
type UserUpdate struct {
	config
	hooks      []Hook
	mutation   *UserMutation
	predicates []predicate.User
}
```



在下面的例子中，ent.UserCreate 和 ent.UserUpdate 都使用一个通用的方法对age 和name 列进行操作：

```go
package main

import (
   "context"
   "log"

   _ "github.com/go-sql-driver/mysql"
   "github.com/peanut-cc/ent_orm_notes/aboutMutaion/ent"
)

func main() {
   client, err := ent.Open("mysql", "root:123456@tcp(10.211.55.3:3306)/aboutMutaion?parseTime=True")
   if err != nil {
      log.Fatal(err)
   }
   defer client.Close()
   ctx := context.Background()
   // run the auto migration tool
   if err := client.Schema.Create(ctx); err != nil {
      log.Fatalf("failed creating schema resources:%v", err)
   }
   Do(ctx, client)
}

func Do(ctx context.Context, client *ent.Client) {
   creator := client.User.Create()
   SetAgeName(creator.Mutation())
   creator.SaveX(ctx)
   updater := client.User.UpdateOneID(1)
   SetAgeName(updater.Mutation())
   updater.SaveX(ctx)
}

// SetAgeName sets the age and the name for any mutation.
func SetAgeName(m *ent.UserMutation) {
   m.SetAge(32)
   m.SetName("Ariel")
}
```



在某些情况下，你希望对多个不同的类型应用同一个方法，对于这种情况，要么使用通用的ent.Mutation 接口，或者自己实现一个接口，代码如下：

```go
func Do2(ctx context.Context, client *ent.Client) {
   creator1 := client.User.Create().SetAge(18)
   SetName(creator1.Mutation(), "a8m")
   creator1.SaveX(ctx)
   creator2 := client.Pet.Create().SetAge(16)
   SetName(creator2.Mutation(), "pedro")
   creator2.SaveX(ctx)
}

// SetNamer wraps the 2 methods for getting
// and setting the "name" field in mutations.
type SetNamer interface {
   SetName(string)
   Name() (string, bool)
}

func SetName(m SetNamer, name string) {
   if _, exist := m.Name(); !exist {
      m.SetName(name)
   }
}
```



## Graph Traversal

在这个部分的例子中会使用如下的Graph

![er-traversal-graph](https://entgo.io/assets/er_traversal_graph.png)

























