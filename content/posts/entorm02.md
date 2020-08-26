---
title: "Entorm02"
date: 2020-08-26T10:25:49+08:00
draft: true
---


在上一篇关于快速使用ent orm的笔记中，我们再最开始使用`entc init User` 创建schema，在ent orm 中的schema 其实就是数据库模型，在schema中我们可以通过Fields 定义数据库中表的字段信息；通过Edges 定义表之间的关系信息；通过Index 定义字段的索引信息等等，这篇文章会整理一下关于ent orm 中如何使用这些。


## Fileds

当我们执行 `entc init User` 之后，会在当前目录下生成一个ent目录，在该目录下有一个schema目录，默认情况下schema/user.go文件如下：

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

如果要对user 这表添加字段，需要在Fileds方法中添加如下所示：

```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("username").
			Unique(),
		field.Time("created_at").
			Default(time.Now),
		field.Float32("salary").
			Optional(),
	}
}
```
**注意： 默认情况下，所有字段都是必填字段，可以使用Optional方法将其设置为optional。**


### 数据类型

下面的数据类型都是支持的：
* All Go numeric types. Like int, uint8, float64, etc.
* bool
* string
* time.Time
* []byte (only supported by SQL dialects).
* JSON (only supported by SQL dialects).
* Enum (only supported by SQL dialects).

```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("username").
			Unique(),
		field.Time("created_at").
			Default(time.Now),
		field.Float32("salary").
			Optional(),
		field.Bool("active").
			Default(false),
		field.JSON("strings", []string{}).
			Optional(),
		field.Enum("state").
			Values("on", "off").
			Optional(),
	}
}
```

### ID字段

数据库表的id字段，默认是内置的，不需要单独添加，其类型默认为int, 并在数据库中自动递增，

为了将id配置为在所有表中唯一，需要在schema migration的时候使用WithGlobalUniqueID

如果需要对id字段进行其他配置，或者想要使用UUID格式存id，则需要覆盖id的配置。


```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("id").
			StructTag(`json:"oid,omitempty"`),
	}
}
```


```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.UUID("id", uuid.UUID{}),
	}
}
```

```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").
			MaxLen(25).
			NotEmpty().
			Unique().
			Immutable(),
	}
}
```

### 数据库类型

每个数据库都有自己的从go的数据类型到数据库类型的映射，例如，Mysql 在数据库中将float64字段创建为双精度的。ent orm 有一个选项参数可以使用SchemaType 方法覆盖默认行为

```go
// Fields of the Card.
func (Card) Fields() []ent.Field {
	return []ent.Field{
		field.Float("amount").
			SchemaType(map[string]string{
				dialect.MySQL:    "decimal(6,2)", // Override MySQL.
				dialect.Postgres: "numeric",      // Override Postgres.
			}),
	}
}
```

### GO Type

字段的默认类型是基本的Go数据类型，例如，对于字符串字段，类型为string, 对于时间字段，类型为time.Time

GoType 方法提供了一个选项，可以使用自定义类型替换默认的ent类型。但自定义类型必须是可以转换为Go的基本类型的类型，或者实现了[ValueScanner](https://pkg.go.dev/github.com/facebook/ent/schema/field?tab=doc#ValueScanner)接口的类型

```go
// Fields of the Card.
func (Card) Fields() []ent.Field {
	return []ent.Field{
		field.Float("amount").
			GoType(Amount(0)),
		field.String("name").
			Optional().
            // A ValueScanner type.
			GoType(&sql.NullString{}),
	}
}
```

### Default Values 默认值

Non-unique 的字段可以通过Default 和 UpdateDefault方法设置默认值

```go
// Fields of the Group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Default(time.Now),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
	}
}
```

### Validators

关于字段的validator是通过 func(T) error 函数，该函数使用Validate方法在schema中定义，并在创建或更新schema的时候应用于字段的校验

字段的validator支持的类型有string 和所有的数字类型


```go
// Fields of the Group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Match(regexp.MustCompile("[a-zA-Z_]+$")).
			Validate(func(s string) error {
				if strings.ToLower(s) == s {
					return errors.New("group name must begin with uppercase")
				}
				return nil
			}),
	}
}
```

### 内置的 Validators

ent orm 提供了一些内置的validators, 如下：

* Numeric types:
    Positive() - Positive adds a minimum value validator with the value of 1
    Negative() - Negative adds a maximum value validator with the value of -1
    NonNegative() - NonNegative adds a minimum value validator with the value of 0
    Min(i) - Validate that the given value is > i.
    Max(i) - Validate that the given value is < i.
    Range(i, j) - Validate that the given value is within the range [i, j].
    
* string:

    MinLen(i)
    MaxLen(i)
    Match(regexp.Regexp)
    
    
### Optional 可选字段

可选字段是在创建的时候不是必传的字段，并将在数据库设置为可为空的字段
默认情况下，字段都是必填字段


### Nillable

有时候你可能希望区分字段的零值和nil，如数据库的某列包含0 或者NULL,Nillable选项正式为此而存在的.

如果有一个类型为T的字段设置为Nillable ，在通过go generate 生成代码的时候的时候生成的struct 中改字段的类型是*T, 如果数据库中该字段是NULL， 那么在ent orm的查询结果中就是nil, 否则对于没有设置Nillable的字段，如果数据库中字段的值是NULl，返回的则是改字段的零值


```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("required_name"),
		field.String("optional_name").Optional(),
		field.String("nilable_name").Optional().Nillable(),
		field.String("nilable_name2").Optional().Nillable(),
		field.Int("age").Optional(),
		field.Int("age2").Optional().Nillable(),
	}
}
```

我们通过如下代码进行数据的创建和查询，这里分别创建了两条数据，第一条数据的设置SetOptionalName，SetNilableName 的字段都没有设置内容，第二次的时候都设置了内容

```go
package main

import (
	"context"
	"log"

	"github.com/peanut-cc/ent_orm_notes/schema_notes/ent/user"

	_ "github.com/go-sql-driver/mysql"
	"github.com/peanut-cc/ent_orm_notes/schema_notes/ent"
)

func main() {
	client, err := ent.Open("mysql", "root:123456@tcp(10.211.55.3:3306)/schema_notes?parseTime=True")
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()
	ctx := context.Background()
	// run the auto migration tool
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources:%v", err)
	}
	client.User.Create().SetRequiredName("peanut").Save(ctx)
	client.User.Create().SetRequiredName("syncd").
		SetOptionalName("option_name").
		SetNilableName("nil_name").
		SetNilableName2("nil_name2").
		SetAge(18).
		SetAge2(20).
		SaveX(ctx)
	u := client.User.Query().Where(user.RequiredNameEQ("peanut")).OnlyX(ctx)
	log.Printf("required_name is:%v option_name is:%v nil_name is:%v nil_name2 is:%v age is :%v age2 is:%v\n", u.RequiredName,
		u.OptionalName, u.NilableName, u.NilableName2, u.Age, u.Age2)
	u2 := client.User.Query().Where(user.RequiredNameEQ("syncd")).OnlyX(ctx)
	log.Printf("required_name is:%v option_name is:%v nil_name is:%v nil_name2 is:%v age is :%v age2 is:%v\n", u2.RequiredName,
		u2.OptionalName, u2.NilableName, u2.NilableName2, u2.Age, u2.Age2)
}

```

下面是数据的打印结果：

```shell
2020/08/26 20:39:47 required_name is:peanut option_name is: nil_name is:<nil> nil_name2 is:<nil> age is :0 age2 is:<nil>
2020/08/26 20:39:47 required_name is:syncd option_name is:option_name nil_name is:0xc000200580 nil_name2 is:0xc000200590 age is :18 age2 is:0xc00020c4d8

```

### Immutable 不可变字段

不可变字段，是只能在创建的时候设置值

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.Time("created_at").
            Default(time.Now).
            Immutable(),
    }
}

```

### Uniqueness 唯一索引

可以使用Unique方法给字段设置唯一索引。 注意：唯一所以字段不能具有默认值

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.String("nickname").
            Unique(),
    }
}
```

### Storage Key

可以使用StorageKey方法配置自定义存储名称。在SQL中映射为列名

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            StorageKey(`old_name"`),
    }
}
```

### Indexes 索引

可以在多个字段和一些关系表中创建索引

### Struct Tags

可以使用StructTag方法将自定义struct tag添加到生成的实体中。
请注意，如果未提供此选项，或者提供的该选项不包含json标记，则默认json标记将与字段名称一起添加。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            StructTag(`gqlgen:"gql_name"`),
    }
}
```

### Sensitive Fields

可以使用Sensitive方法将字符串字段定义为Sensitive Fields。
Sensitive Fields不会被打印，并且在编码时将被忽略。
请注意，Sensitive Fields不能具有struct标记。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("password").
            Sensitive(),
    }
}
```

### Annotations

在代码生成中，Annotations用于将任意元数据附加到字段对象。模板扩展可以检索这个元数据并在它们的模板中使用它。注意，元数据对象必须可序列化为 JSON 原始值(例如，struct、 map 或 slice)。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.Time("creation_date").
            Annotations(entgql.Annotation{
                OrderField: "CREATED_AT",
            }),
    }
}
```