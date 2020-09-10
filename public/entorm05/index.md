# ent orm笔记5---结束




## Hooks

Hooks允许你在对表做一些操作的时候添加一些自定义的处理逻辑。如当我对某个表的某条数据进行更新的前和后，做一些自定义的操作，其实这个就和web 中 middleware 中间件的原理类似

具体我们可以一下的操作中增加hook:

- `Create` - Create node in the graph.
- `UpdateOne` - Update a node in the graph. For example, increment its field.
- `Update` - Update multiple nodes in the graph that match a predicate.
- `DeleteOne` - Delete a node from the graph.
- `Delete` - Delete all nodes that match a predicate.



mutation hooks 有两种类型：**schema hooks**  和 **runtime hooks**

schema hooks 主要是在schema 中添加一些自定义操作

runtime hooks 主要是用在像logging, metrics, tracing,等等



### Runtime hooks 



这里通过一个简单的例子理解：

```go
package main

import (
   "context"
   "log"
   "time"

   _ "github.com/go-sql-driver/mysql"
   "github.com/peanut-cc/ent_orm_notes/hook_runtime_example/ent"
)

func main() {
   client, err := ent.Open("mysql", "root:123456@tcp(10.211.55.3:3306)/hook_runtime?parseTime=True")
   if err != nil {
      log.Fatalf("ent open db error:%v", err)
   }
   defer client.Close()
   ctx := context.Background()
   // run the auto migration tool
   if err := client.Schema.Create(ctx); err != nil {
      log.Fatalf("failed creating schema resources:%v", err)
   }

   client.Use(func(next ent.Mutator) ent.Mutator {
      return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
         start := time.Now()
         defer func() {
            log.Printf("Op=%s\tType=%s\tTime=%s\tConreteType=%T", m.Op(), m.Type(), time.Since(start), m)
         }()
         return next.Mutate(ctx, m)
      })
   })
   Do(ctx, client)
}

func Do(ctx context.Context, client *ent.Client) {
   client.User.Create().SetName("peanut").SaveX(ctx)
}
```



日志打印如下：

`2020/09/02 11:44:55 Op=OpCreate Type=User       Time=4.535547ms ConreteType=*ent.UserMutation`



这里是添加了一个全局的hook, 记录了我们对数据库每条记录操作的耗时，这个在日常的工作中也是非常有用的。

但是在某些场景下，我们可能有更细粒度的使用，如下：

```go
func main() {
    // <client was defined in the previous block>

    // Add a hook only on user mutations.
    client.User.Use(func(next ent.Mutator) ent.Mutator {
        // Use the "<project>/ent/hook" to get the concrete type of the mutation.
        return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
            return next.Mutate(ctx, m)
        })
    })
    
    // Add a hook only on update operations.
    client.Use(hook.On(Logger(), ent.OpUpdate|ent.OpUpdateOne))
    
    // Reject delete operations.
    client.Use(hook.Reject(ent.OpDelete|ent.OpDeleteOne))
}
```



如果希望共享一个在多种类型(例如 Group 和 User)之间变化字段的钩子。有两种方法可以做到这一点:

```go
// Option 1: use type assertion.
client.Use(func(next ent.Mutator) ent.Mutator {
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        if ns, ok := m.(interface{ SetName(string) }); ok {
            ns.SetName("Ariel Mashraki")
        }
        return next.Mutate(ctx, m)
    })
})

// Option 2: use the generic ent.Mutation interface.
client.Use(func(next ent.Mutator) ent.Mutator {
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        if err := m.SetField("name", "Ariel Mashraki"); err != nil {
            // An error is returned, if the field is not defined in
            // the schema, or if the type mismatch the field type.
        }
        return next.Mutate(ctx, m)
    })
})
```



### Schema hooks

schema 在定义schema的时候使用，并且仅应用于schema type的更改，这个初衷是把schem 中关于节点类型相关的所有逻辑集中在一个地方，代码例子：

```go
package schema

import (
    "context"
    "fmt"

    gen "<project>/ent"
    "<project>/ent/hook"

    "github.com/facebook/ent"
)

// Card holds the schema definition for the CreditCard entity.
type Card struct {
    ent.Schema
}

// Hooks of the Card.
func (Card) Hooks() []ent.Hook {
    return []ent.Hook{
        // First hook.
        hook.On(
            func(next ent.Mutator) ent.Mutator {
                return hook.CardFunc(func(ctx context.Context, m *gen.CardMutation) (ent.Value, error) {
                    if num, ok := m.Number(); ok && len(num) < 10 {
                        return nil, fmt.Errorf("card number is too short")
                    }
                    return next.Mutate(ctx, m)
                })
            },
            // Limit the hook only for these operations.
            ent.OpCreate|ent.OpUpdate|ent.OpUpdateOne,
        ),
        // Second hook.
        func(next ent.Mutator) ent.Mutator {
            return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
                if s, ok := m.(interface{ SetName(string) }); ok {
                    s.SetName("Boring")
                }
                return next.Mutate(ctx, m)
            })
        },
    }
}
```



请注意，如果使用schema hook，则必须在main包中添加以下导入：

`import _ "<project>/ent/runtime"`



### Evaluation order

Hook按照它们注册到client的顺序被调用。因此client.Use(f，g，h)在mutations执行 f (g (h (...))。

还要注意，**runtime hooks**在**schema hooks**.之前调用。也就是说，如果 g 和 h 是在schema中定义的，而 f 是使用 client.Use(...)注册的。 它们将被执行如下: f (g (h (...)))。



## Transactions



开始一个事务，例子：

```go
// GenTx generates group of entities in a transaction.
func GenTx(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return fmt.Errorf("starting a transaction: %v", err)
    }
    hub, err := tx.Group.
        Create().
        SetName("Github").
        Save(ctx)
    if err != nil {
        return rollback(tx, fmt.Errorf("failed creating the group: %v", err))
    }
    // Create the admin of the group.
    dan, err := tx.User.
        Create().
        SetAge(29).
        SetName("Dan").
        AddManage(hub).
        Save(ctx)
    if err != nil {
        return rollback(tx, err)
    }
    // Create user "Ariel".
    a8m, err := tx.User.
        Create().
        SetAge(30).
        SetName("Ariel").
        AddGroups(hub).
        AddFriends(dan).
        Save(ctx)
    if err != nil {
        return rollback(tx, err)
    }
    fmt.Println(a8m)
    // Output:
    // User(id=2, age=30, name=Ariel)
    
    // Commit the transaction.
    return tx.Commit()
}

// rollback calls to tx.Rollback and wraps the given error
// with the rollback error if occurred.
func rollback(tx *ent.Tx, err error) error {
    if rerr := tx.Rollback(); rerr != nil {
        err = fmt.Errorf("%v: %v", err, rerr)
    }
    return err
}
```





有些情况，如果你现有的代码已经可以用`*ent.Client` 正常执行，这个时候，你希望这个执行过程添加事务的功能，这种情况下，可以通过如下方式实现：



```go
// WrapGen wraps the existing "Gen" function in a transaction.
func WrapGen(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    txClient := tx.Client()
    // Use the "Gen" below, but give it the transactional client; no code changes to "Gen".
    if err := Gen(ctx, txClient); err != nil {
        return rollback(tx, err)
    }
    return tx.Commit()
}

// Gen generates a group of entities.
func Gen(ctx context.Context, client *ent.Client) error {
    // ...
    return nil
}
```





### 最佳实践



```go
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    defer func() {
        if v := recover(); v != nil {
            tx.Rollback()
            panic(v)
        }
    }()
    if err := fn(tx); err != nil {
        if rerr := tx.Rollback(); rerr != nil {
            err = errors.Wrapf(err, "rolling back transaction: %v", rerr)
        }
        return err
    }
    if err := tx.Commit(); err != nil {
        return errors.Wrapf(err, "committing transaction: %v", err)
    }
    return nil
}

func Do(ctx context.Context, client *ent.Client) {
    // WithTx helper.
    if err := WithTx(ctx, client, func(tx *ent.Tx) error {
        return Gen(ctx, tx.Client())
    }); err != nil {
        log.Fatal(err)
    }
}
```



像schema hooks 和runtime hooks 一样，hooks可以注册在事务上，并且将会被执行在Tx.Commit` or `Tx.Rollback

```go
func Do(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    // Add a hook on Tx.Commit.
    tx.OnCommit(func(next ent.Committer) ent.Committer {
        return ent.CommitFunc(func(ctx context.Context, tx *ent.Tx) error {
            // Code before the actual commit.
            err := next.Commit(ctx, tx)
            // Code after the transaction was committed.
            return err
        })
    })
    // Add a hook on Tx.Rollback.
    tx.OnRollback(func(next ent.Rollbacker) ent.Rollbacker {
        return ent.RollbackFunc(func(ctx context.Context, tx *ent.Tx) error {
            // Code before the actual rollback.
            err := next.Rollback(ctx, tx)
            // Code after the transaction was rolled back.
            return err
        })
    })
    //
    // <Code goes here>
    //
    return err
}
```





## Migration



ent 提供了数据库迁移支持，可以使用数据库表结构与`ent/migrate/schema ` 中定义的对象保持一致。



### Auto Migration



在程序的初始化过程中，运行auto-migration

```go
if err := client.Schema.Create(ctx); err != nil {
    log.Fatalf("failed creating schema resources: %v", err)
}
```



Create 将创建 ent 项目所需的所有数据库资源。默认情况下，Create 在“ append-only”模式下工作; 这意味着，它只创建新的表和索引，将列追加到表或扩展列类型。例如，将 int 改为 bigint。

删除列或索引怎么样？



### Drop Resources

WithDropIndex 和 WithDropColumn 两个选项用于删除表列和索引。

```go
package main

import (
    "context"
    "log"
    
    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(
        ctx, 
        migrate.WithDropIndex(true),
        migrate.WithDropColumn(true), 
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```



为了在调试模式下运行迁移(打印所有 SQL 查询) ，请运行:

```go
err := client.Debug().Schema.Create(
    ctx, 
    migrate.WithDropIndex(true),
    migrate.WithDropColumn(true),
)
if err != nil {
    log.Fatalf("failed creating schema resources: %v", err)
}
```



### Universal IDs



默认情况下，每个表的 SQL 主键从1开始; 这意味着不同类型的多个实体可以共享相同的 ID。不像 AWS Neptune， 是 uuid。

如果使用 GraphQL，这将不能很好地工作，因为它要求对象 ID 是唯一的。



要启用项目的 Universal-IDs 支持，请将 WithGlobalUniqueID 选项传递给迁移。

```go
package main

import (
    "context"
    "log"
    
    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    if err := client.Schema.Create(ctx, migrate.WithGlobalUniqueID(true)); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```





它是如何工作的？Ent 迁移为每个实体(表)的 id 分配1 << 32范围，并将此信息存储在名为 ent _ types 的表中。例如，a 类型的 id 的范围是[1,4294967296] ，b 类型的 id 范围是[4294967296,8589934592]等。

请注意，如果启用此选项，则可能的表的最大数量为65535。



### Offline Mode

脱机模式允许您将模式更改写入 io。然后在数据库中执行它们。这对于在数据库上执行 SQL 命令之前验证它们或手动运行 SQL 脚本非常有用。

**Print changes**

```go
package main

import (
    "context"
    "log"
    "os"
    
    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Dump migration changes to stdout.
    if err := client.Schema.WriteTo(ctx, os.Stdout); err != nil {
        log.Fatalf("failed printing schema changes: %v", err)
    }
}
```

**Write changes to a file**

```go
package main

import (
    "context"
    "log"
    "os"
    
    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Dump migration changes to an SQL script.
    f, err := os.Create("migrate.sql")
    if err != nil {
        log.Fatalf("create migrate file: %v", err)
    }
    defer f.Close()
    if err := client.Schema.WriteTo(ctx, f); err != nil {
        log.Fatalf("failed printing schema changes: %v", err)
    }
}
```



## sql.DB Integration

下面的示例演示如何将自定义 sql.db 对象传递给 ent.Client。



```go
package main

import (
    "time"

    "<your_project>/ent"
    "github.com/facebook/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    drv, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
        return nil, err
    }
    // Get the underlying sql.DB object of the driver.
    db := drv.DB()
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    return ent.NewClient(ent.Driver(drv)), nil
}
```





```go
package main

import (
    "database/sql"
    "time"

    "<your_project>/ent"
    entsql "github.com/facebook/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    db, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
        return nil, err
    }
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    // Create an ent.Driver from `db`.
    drv := entsql.OpenDB("mysql", db)
    return ent.NewClient(ent.Driver(drv)), nil
}
```



### Use Opencensus With MySQL 

```go
package main

import (
    "context"
    "database/sql"
    "database/sql/driver"

    "<project>/ent"
    
    "contrib.go.opencensus.io/integrations/ocsql"
    "github.com/go-sql-driver/mysql"
    entsql "github.com/facebook/ent/dialect/sql"
)

type connector struct {
    dsn string
}

func (c connector) Connect(context.Context) (driver.Conn, error) {
    return c.Driver().Open(c.dsn)
}

func (connector) Driver() driver.Driver {
    return ocsql.Wrap(
        mysql.MySQLDriver{},
        ocsql.WithAllTraceOptions(),
        ocsql.WithRowsClose(false),
        ocsql.WithRowsNext(false),
        ocsql.WithDisableErrSkip(true),
    )
}

// Open new connection and start stats recorder.
func Open(dsn string) *ent.Client {
    db := sql.OpenDB(connector{dsn})
    // Create an ent.Driver from `db`.
    drv := entsql.OpenDB("mysql", db)
    return ent.NewClient(ent.Driver(drv))
}
```

### Use pgx with PostgreSQL

```go
package main

import (
    "context"
    "database/sql"
    "log"

    "<project>/ent"

    "github.com/facebook/ent/dialect"
    entsql "github.com/facebook/ent/dialect/sql"
    _ "github.com/jackc/pgx/v4/stdlib"
)

// Open new connection
func Open(databaseUrl string) *ent.Client {
    db, err := sql.Open("pgx", databaseUrl)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Create an ent.Driver from `db`.
    drv := entsql.OpenDB(dialect.Postgres, db)
    return ent.NewClient(ent.Driver(drv))
}

func main() {
    client := Open("postgresql://user:password@127.0.0.1/database")

    // Your code. For example:
    ctx := context.Background()
    if err := client.Schema.Create(ctx); err != nil {
        log.Fatal(err)
    }
    users, err := client.User.Query().All(ctx)
    if err != nil {
        log.Fatal(err)
    }
    log.Println(users)
}
```





## 总结

对en orm 整个文档进行整理学习之后，计划后续通过ent orm ,gin web框架实现一个blog，这样也算是对于ent orm 有一个简单的应用







## 延伸阅读

- https://entgo.io/
