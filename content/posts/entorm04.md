---
mmtitle: "ent orm笔记4---Code Generation"
date: 2020-08-31T15:08:42+08:00
draft: false
author: "fan"
subtitle: "ent orm笔记2"
tags: ["orm","go"]
categories: ["go"]
toc:
  enable: true
  auto: true
---



在前面几篇文章中，我们经常使用的可能就是entc这个命令了，entc这个工具给带来了很多功能，这篇文章主要整理关于ent orm 中Code Generation

之前的例子中有个知识点少整理了，就是关于如果我们想要看orm在执行过程中详细原生sql语句是可以开启Debug看到的，代码如下：

```go
client, err := ent.Open("mysql", "root:123456@tcp(10.211.55.3:3306)/graph_traversal?parseTime=True",ent.Debug())
```



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



Generate 命令生成以下内容：

- 用于与graph 交互的Client 和Tx对象
- schema 的CRUD生成器
- 每个schema类型的Entity对象
- 用于与构建交互的常量和断言
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



### Storage

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



例如，所有的[`User` builders](https://entgo.io/docs/crud#create-an-entity)都共享相同的UserMutaion 对象，左右的builder 类型都继承通用的[`ent.Mutation`](https://pkg.go.dev/github.com/facebook/ent?tab=doc#Mutation)接口.

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

![er-traversal-graph](/home/fan/桌面/er_traversal_graph.png)







![er-traversal-graph-gopher](/home/fan/桌面/er_traversal_graph_gopher.png)





上面的遍历从一个 Group 实体开始，继续到它的 admin (edge) ，继续到它的朋友(edge) ，获取他们的宠物(edge) ，获取每个宠物的朋友(edge) ，并请求它们的主人

```go
func Traverse(ctx context.Context, client *ent.Client) error {
    owner, err := client.Group.         // GroupClient.
        Query().                        // Query builder.
        Where(group.Name("Github")).    // Filter only Github group (only 1).
        QueryAdmin().                   // Getting Dan.
        QueryFriends().                 // Getting Dan's friends: [Ariel].
        QueryPets().                    // Their pets: [Pedro, Xabi].
        QueryFriends().                 // Pedro's friends: [Coco], Xabi's friends: [].
        QueryOwner().                   // Coco's owner: Alex.
        Only(ctx)                       // Expect only one entity to return in the query.
    if err != nil {
        return fmt.Errorf("failed querying the owner: %v", err)
    }
    fmt.Println(owner)
    // Output:
    // User(id=3, age=37, name=Alex)
    return nil
}
```



下面的遍历如何？

![er-traversal-graph-gopher-query](/home/fan/桌面/er_traversal_graph_gopher_query.png)



我们希望得到所有宠物(entities)的所有者(edge)是朋友(edge)的一些群管理员(edge)。

```
func Traverse2(ctx context.Context, client *ent.Client) error {
	pets, err := client.Pet.
		Query().
		Where(
			pet.HasOwnerWith(
				user.HasFriendsWith(
					user.HasManage(),
				),
			),
		).
		All(ctx)
	if err != nil {
		return fmt.Errorf("failed querying the pets: %v", err)
	}
	fmt.Println(pets)
	// Output:
	// [Pet(id=1, name=Pedro) Pet(id=2, name=Xabi)]
	return nil
}
```



上面的查询中，查询所有的宠物，条件是: 宠物要有主人，同时宠物的主人是要有朋友，同时该主人还要属于管理员



## Eager Loading

ent 支持通过它们的edges 查询，并将关联的entities 添加到返回的对象中

通过下面的例子理解：

![er-group-users](/home/fan/桌面/er_user_pets_groups2.png)



查询上面关系中所有用户和它们的宠物，代码如下：

```go
func edgerLoading(ctx context.Context, client *ent.Client) {
   users, err := client.User.Query().WithPets().All(ctx)
   if err != nil {
      log.Fatalf("user query failed:%v", err)
   }
   log.Println(users)
   for _, u := range users {
      for _, p := range u.Edges.Pets {
         log.Printf("user (%v) -- > Pet (%v)\n", u.Name, p.Name)
      }
   }

}
```

完整的代码在：https://github.com/peanut-cc/ent_orm_notes/graph_traversal

查询的结果如下：

```shell
2020/09/01 20:09:07 [User(id=1, age=29, name=Dan) User(id=2, age=30, name=Ariel) User(id=3, age=37, name=Alex) User(id=4, age=18, name=peanut)]
2020/09/01 20:09:07 user (Ariel) -- > Pet (Pedro)
2020/09/01 20:09:07 user (Ariel) -- > Pet (Xabi)
2020/09/01 20:09:07 user (Alex) -- > Pet (Coco)

```



预加载允许查询多个关联，包括嵌套关联，还可以过滤，排序或限制查询结果，例如：



```go
func edgerLoading2(ctx context.Context, client *ent.Client) {
   users, err := client.User.
      Query().
      Where(
         user.AgeGT(18),
      ).
      WithPets().
      WithGroups(func(q *ent.GroupQuery) {
         q.Limit(5)
         q.WithUsers().Limit(5)
      }).All(ctx)
   if err != nil {
      log.Fatalf("user query failed:%v", err)
   }
   log.Println(users)
   for _, u := range users {
      for _, p := range u.Edges.Pets {
         log.Printf("user (%v) --> Pet (%v)\n", u.Name, p.Name)
      }
      for _, g := range u.Edges.Groups {
         log.Printf("user (%v) -- Group (%v)\n", u.Name, g.Name)
      }
   }

}
```



每个query-builder都有一个方法列表，其形式为 `With<E>(...func(<N>Query))`

`<E>`代表边缘名称(像`WithGroups`) ，`< N>` 代表边缘类型(像`GroupQuery`)。

注意，只有 SQL 方言支持这个特性



## Aggregation



### Group By

按所有用户的姓名和年龄字段分组，并计算其总年龄。

```go
package main

import (
	"context"
	"log"

	"github.com/peanut-cc/ent_orm_notes/groupBy/ent/user"

	_ "github.com/go-sql-driver/mysql"

	"github.com/peanut-cc/ent_orm_notes/groupBy/ent"
)

func main() {
	client, err := ent.Open("mysql", "root:123456@tcp(10.211.55.3:3306)/groupBy?parseTime=True",
		ent.Debug())
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()
	ctx := context.Background()
	// run the auto migration tool
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources:%v", err)
	}
	GenData(ctx, client)
	Do(ctx, client)

}

func GenData(ctx context.Context, client *ent.Client) {
	client.User.Create().SetName("peanut").SetAge(18).SaveX(ctx)
	client.User.Create().SetName("jack").SetAge(20).SaveX(ctx)
	client.User.Create().SetName("steve").SetAge(22).SaveX(ctx)
	client.User.Create().SetName("peanut-cc").SetAge(18).SaveX(ctx)
	client.User.Create().SetName("jack-dd").SetAge(18).SaveX(ctx)
}

func Do(ctx context.Context, client *ent.Client) {
	var v []struct {
		Name  string `json:"name"`
		Age   int    `json:"age"`
		Sum   int    `json:"sum"`
		Count int    `json:"count"`
	}
	client.User.
		Query().
		GroupBy(
			user.FieldName, user.FieldAge,
		).
		Aggregate(
			ent.Count(),
			ent.Sum(user.FieldAge),
		).
		ScanX(ctx, &v)
	log.Println(v)

}
```



按一个字段分组，例子如下：

```go
func Do2(ctx context.Context, client *ent.Client) {
	names := client.User.Query().GroupBy(user.FieldName).StringsX(ctx)
	log.Println(names)
}
```



## Predicates



### Field Predicates

- Bool:
  -  =, !=
- Numberic:
  - =, !=, >, <, >=, <=,
  - IN, NOT IN
- Time:
  - =, !=, >, <, >=, <=
  - IN, NOT IN
- String:
  - =, !=, >, <, >=, <=
  - IN, NOT IN
  - Contains, HasPrefix, HasSuffix
  - ContainsFold, EqualFold (**SQL** specific)
- Optional fields:
  - IsNil, NotNil



### Edge Predicates



**HasEdge** 例如，查询所有宠物的所有者，使用:

```go
 client.Pet.
      Query().
      Where(pet.HasOwner()).
      All(ctx)
```

**HasEdgeWith**

```go
 client.Pet.
      Query().
      Where(pet.HasOwnerWith(user.Name("a8m"))).
      All(ctx)
```



### Negation (NOT)

```go
client.Pet.
    Query().
    Where(pet.Not(pet.NameHasPrefix("Ari"))).
    All(ctx)
```



### Disjunction (OR)

```go
client.Pet.
    Query().
    Where(
        pet.Or(
            pet.HasOwner(),
            pet.Not(pet.HasFriends()),
        )
    ).
    All(ctx)
```

### Conjunction (AND)

```go
client.Pet.
    Query().
    Where(
        pet.And(
            pet.HasOwner(),
            pet.Not(pet.HasFriends()),
        )
    ).
    All(ctx)
```



### Custom Predicates

如果想编写自己的特定于方言的逻辑，Custom predicates可能很有用。

```go
pets := client.Pet.
    Query().
    Where(predicate.Pet(func(s *sql.Selector) {
        s.Where(sql.InInts(pet.OwnerColumn, 1, 2, 3))
    })).
    AllX(ctx)
```



## Paging And Ordering

### Limit

将查询结果限制为 n 个实体。

```go
users, err := client.User.
    Query().
    Limit(n).
    All(ctx)
```

### Offset

设置从查询返回的第一个最大数量。

```go
users, err := client.User.
    Query().
    Offset(10).
    All(ctx)
```

### Ordering

Order 返回按一个或多个字段的值排序的实体。

```go
users, err := client.User.Query().
    Order(ent.Asc(user.FieldName)).
    All(ctx)
```



## 延伸阅读

- https://entgo.io/