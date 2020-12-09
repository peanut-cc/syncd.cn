---
title: "Go进阶笔记-并发编程"
date: 2020-12-09T17:39:42+08:00
draft: false
author: "fan"
subtitle: "Go进阶笔记-关于error"
tags: ["go"]
categories: ["go"]
toc:
  enable: true
  auto: true
---

## goroutine

Go 语言层面支持的 go 关键字，可以快速的让一个函数创建为 goroutine，我们可以认为 main 函数就是作为 goroutine 执行的。操作系统调度线程在可用处理器上运行，Go运行时调度 goroutines 在绑定到单个操作系统线程的逻辑处理器中运行(P)。即使使用这个单一的逻辑处理器和操作系统线程，也可以调度数十万 goroutine 以惊人的效率和性能并发运行。



并发不是并行。并行是指两个或多个线程同时在不同的处理器执行代码。如果将运行时配置为使用多个逻辑处理器，则调度程序将在这些逻辑处理器之间分配 goroutine，这将导致 goroutine 在不同的操作系统线程上运行。但是，要获得真正的并行性，您需要在具有多个物理处理器的计算机上运行程序。否则，goroutines 将针对单个物理处理器并发运行，即使 Go 运行时使用多个逻辑处理器。


虽然go 开启一个goroutine很方便，但是这并意味着我们可以不过脑子的随便go，我们每次go开启一个goroutine都要思考如下问题：
- 它什么时候会退出？
- 如何能够让它结束？
- 把并发交给调用者！

初学者写go代码的时候经常可能是如下例子：

```
package main

import (
	"fmt"
	"net/http"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
		fmt.Println(rw, "Hello Golang")
	})
	go http.ListenAndServe("127.0.0.1:8080", http.DefaultServeMux)
	http.ListenAndServe("127.0.0.1:9090", mux)
}

```

这里很明显我们对go开启的goroutine 是不能能知道它什么时候会退出的，并且我们也没有一个好的办法让它退出，优雅的代码应该如下：



```
package main

import (
	"context"
	"fmt"
	"net/http"
)


func serverApp(stop <-chan struct{}) error {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
		fmt.Println(rw, "Hello Golang")
	})
	s := http.Server{
		Addr:    "0.0.0.0:8080",
		Handler: mux,
	}
	go func() {
		<-stop
		s.Shutdown(context.Background())
	}()
	return s.ListenAndServe()

}

func serverDebug(stop <-chan struct{}) error {
	s := http.Server{
		Addr:    "0.0.0.0:9090",
		Handler: http.DefaultServeMux,
	}
	go func() {
		<-stop
		s.Shutdown(context.Background())
	}()
	return s.ListenAndServe()
}

func main() {
	done := make(chan error, 2)
	stop := make(chan struct{})
	go func() {
		done <- serverApp(stop)
	}()
	go func() {
		done <- serverDebug(stop)
	}()

	var stoped bool
	for i := 0; i < cap(done); i++ {
		if err := <-done; err != nil {
			fmt.Printf("error:%v\n", err)
		}
		if !stoped {
			stoped = true
			close(stop)
		}
	}
}

```

我们再看一个例子：


```
type Tracker struct{}

func (t *Tracker) Event(data string) {
	time.Sleep(time.Microsecond)
	log.Println(data)
}

type App struct {
	track Tracker
}

func (a *App) Handle(w http.ResponseWriter, r *http.Request) {

	// do some work
	w.WriteHeader(http.StatusCreated)

	// 这个地方其实是有问题的
	go a.track.Event("test event")

}
```

还是同样的，重要的事情先思考如下问题：
- 它什么时候会退出？
- 如何能够让它结束？
- 把并发交给调用者！

显然上面的代码是不满足的，更改之后如下：


```
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	tr := NewTracker()
	go tr.Run()

	_ = tr.Event(context.Background(), "test1")
	_ = tr.Event(context.Background(), "test2")
	_ = tr.Event(context.Background(), "test3")
	_ = tr.Event(context.Background(), "test4")
	_ = tr.Event(context.Background(), "test5")
	_ = tr.Event(context.Background(), "test6")
	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(3*time.Second))
	defer cancel()
	tr.Shutdown(ctx)
}

type Tracker struct {
	ch   chan string
	stop chan struct{}
}

func NewTracker() *Tracker {
	return &Tracker{
		ch: make(chan string, 10),
	}
}

func (t *Tracker) Event(ctx context.Context, data string) error {
	select {
	case t.ch <- data:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (t *Tracker) Run() {
	for data := range t.ch {
		time.Sleep(1 * time.Second)
		fmt.Println(data)
	}
	t.stop <- struct{}{}
}

func (t *Tracker) Shutdown(ctx context.Context) {
	close(t.ch)
	select {
	case <-t.stop:
	case <-ctx.Done():
	}
}

```







## sync

Go 的并发原语 goroutines 和 channels 为构造并发软件提供了一种优雅而独特的方法。

在Go中如果我们写完代码想要对代码是否存在数据竞争进行检查，可以通过`go build -race` 对程序进行编译

```
package main

import (
	"fmt"
	"sync"
)

var Wait sync.WaitGroup
var Counter int = 0

func main() {
	for routine := 1; routine <= 2; routine++ {
		Wait.Add(1)
		go Routine()
	}
	Wait.Wait()
	fmt.Printf("Final Counter:%d\n", Counter)
}

func Routine() {
	Counter++
	Wait.Done()
}
```

`go build -race ` 编译后的程序，运行可以很方便看到代码中存在的问题


```
==================
WARNING: DATA RACE
Read at 0x000001277ce0 by goroutine 8:
  main.Routine()
      /Users/zhaofan/open_source_study/test_code/202012/race/main.go:21 +0x3e

Previous write at 0x000001277ce0 by goroutine 7:
  main.Routine()
      /Users/zhaofan/open_source_study/test_code/202012/race/main.go:21 +0x5a

Goroutine 8 (running) created at:
  main.main()
      /Users/zhaofan/open_source_study/test_code/202012/race/main.go:14 +0x6b

Goroutine 7 (finished) created at:
  main.main()
      /Users/zhaofan/open_source_study/test_code/202012/race/main.go:14 +0x6b
==================
Final Counter:2
Found 1 data race(s)
```

对于锁的使用： 最晚加锁，最早释放。


对于下面这段代码，这是模拟一个读多写少的情况，正常情况下，每次读到cfg中的数字都应该是依次递增加1的，但是如果运行代码，则会发现，会出现意外的情况。

```
package main

import (
	"fmt"
	"sync"
)

var wg sync.WaitGroup

type Config struct {
	a []int
}

func main() {
	cfg := &Config{}
	// 这里模拟数据的变化
	go func() {
		i := 0
		for {
			i++
			cfg.a = []int{i, i + 1, i + 2, i + 3, i + 4, i + 5}
		}
	}()

	// 这里模拟去获取数据
	var wg sync.WaitGroup
	for n := 0; n < 4; n++ {
		wg.Add(1)
		go func() {
			for n := 0; n < 20; n++ {
				fmt.Printf("%v\n", cfg)
			}
			wg.Done()
		}()
	}
	wg.Wait()
}

```

对于上面这个代码的解决办法有很多
- Mutex
- RWMutext
- Atomic

对于这种读多写少的情况，使用RWMutext或Atomic 都可以解决，这里只写写一个两者的对比，通过测试也很容易看到两者的性能差别：


```
package main

import (
	"sync"
	"sync/atomic"
	"testing"
)

type Config struct {
	a []int
}

func (c *Config) T() {

}

func BenchmarkAtomic(b *testing.B) {
	var v atomic.Value
	v.Store(&Config{})

	go func() {
		i := 0
		for {
			i++
			cfg := &Config{a: []int{i, i + 1, i + 2, i + 3, i + 4, i + 5}}
			v.Store(cfg)
		}
	}()

	var wg sync.WaitGroup
	for n := 0; n < 4; n++ {
		wg.Add(1)
		go func() {
			for n := 0; n < b.N; n++ {
				cfg := v.Load().(*Config)
				cfg.T()
				// fmt.Printf("%v\n", cfg)
			}
			wg.Done()
		}()
	}
	wg.Wait()
}

func BenchmarkMutex(b *testing.B) {
	var l sync.RWMutex
	var cfg *Config

	go func() {
		i := 0
		for {
			i++
			l.RLock()
			cfg = &Config{a: []int{i, i + 1, i + 2, i + 3, i + 4, i + 5}}
			cfg.T()
			l.RUnlock()
		}
	}()

	var wg sync.WaitGroup
	for n := 0; n < 4; n++ {
		wg.Add(1)
		go func() {
			for n := 0; n < b.N; n++ {
				l.RLock()
				cfg.T()
				l.RUnlock()
			}
			wg.Done()
		}()
	}
	wg.Wait()
}

```

从结果来看性能差别还是非常明显的：

```
 zhaofan@zhaofandeMBP  ~/open_source_study/test_code/202012/atomic_ex2  go test -bench=. config_test.go
goos: darwin
goarch: amd64
BenchmarkAtomic-4       310045898                3.91 ns/op
BenchmarkMutex-4        11382775               101 ns/op
PASS
ok      command-line-arguments  3.931s
 zhaofan@zhaofandeMBP  ~/open_source_study/test_code/202012/atomic_ex2  
```

Mutext锁的实现有一下几种模式：
- Barging, 这种模式是为了提高吞吐量，当锁释放时，它会唤醒第一个等待者，然后把锁给第一个等待者或者第一个请求锁的人。注意这个时候释放锁的那个goroutine 是不会保证下一个人一定能拿到锁，可以理解为只是告诉等待的那个人，我已经释放锁了，快去抢吧。
- Handsoff，当释放锁的时候，锁会一直持有直到第一个等待者准备好获取锁，它降低了吞吐量，因为锁被持有，即使另外一个goroutine准备获取它。相对Barging，这种在释放锁的时候回问下一个要获取锁的，你准备好了么，准备好了我就把锁给你了。
- Spinning，自旋在等待队列为空或者应用程序重度使用锁时效果不错，parking和unparking goroutines 有不低的性能成本开销，相比自旋来说要慢的多。


Go 1.8 使用了Bargin和Spinning的结合实现。当试图获取已经被持有的锁时，如果本地队列为空并且P的数量大于1，goroutine 将自旋几次（用一个P旋转会阻塞程序），自旋后，goroutine park 在程序高频使用锁的情况下，它充当了一个快速路径。

Go1.9 通过添加一个新的饥饿模式来解决出现锁饥饿的情况，该模式将会在释放的时候触发handsoff, 所有等待锁超过一毫秒的goroutine（也被称为有界等待）将被诊断为饥饿，当被标记为饥饿状态时，unlock方法会handsoff把锁直接扔给第一个等待者。

在饥饿模式下，自旋也会被停用，因为传入的goroutines将没有机会获取为下一个等待者保留的锁。

### errgroup

https://pkg.go.dev/golang.org/x/sync/errgroup

使用场景，如果我们有一个复杂的任务，需要拆分为三个任务goroutine 去执行，errgroup 是一个非常不错的选择。

下面是官网的一个例子：


```
package main

import (
	"fmt"
	"golang.org/x/sync/errgroup"
	"net/http"
)

func main() {
	g := new(errgroup.Group)
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somestupidname.com/",
	}
	for _, url := range urls {
		// Launch a goroutine to fetch the URL.
		url := url // https://golang.org/doc/faq#closures_and_goroutines
		g.Go(func() error {
			// Fetch the URL.
			resp, err := http.Get(url)
			if err == nil {
				resp.Body.Close()
			}
			return err
		})
	}
	// Wait for all HTTP fetches to complete.
	if err := g.Wait(); err == nil {
		fmt.Println("Successfully fetched all URLs.")
	}
}
```

### Sync.Poll

sync.poll的场景是用来保存和复用临时对象，减少内存分配，降低GC压力， Request-Drive 特别适合

Get 返回Pool中的任意一个对象，如果Pool 为空，则调用New返回一个新创建的对象

放进pool中的对象，不确定什么时候就会被回收掉，如果实现Put进去100个对象，下次Get的时候发现Pool是空的也是有可能的。所以sync.Pool中是不能放连接型的对象。所以sync.Pool中应该放的是任意时刻都可以被回收的对象。

sync.Pool中的这个清理过程是在每次垃圾回收之前做的，之前每次GC是都会清空pool, 而在1.13版本中引入了victim cache, 会将pool内数据拷贝一份，避免GC将其清空，即使没有引用的内容也可以保留最多两轮GC。

## Context

在Go 服务中，每个传入的请求都在自己的goroutine中处理，请求处理程序通常启动额外的goroutine 来访问其他后端，如数据库和RPC服务，处理请求的goroutine通常需要访问特定于请求（request-specific context）的值,例如最终用户的身份，授权令牌和请求的截止日期。**当一个请求被取消或者超时时，处理该请求的所有goroutine都应该快速推出，这样系统就可以回收他们正在使用的任何资源。*


如何将context 集成到API中？
- 首参数传递context对象
- 在第一个request对象中携带一个可选的context对象

**注意：尽量把context 放到函数的首选参数，而不要把context 放到一个结构体中。**

### context.WithValue

为了实现不断WithValue, 构建新的context，内部在查找key时候，使用递归方式不断寻找匹配的key，知道root context(Backgrond和TODO value的函数会返回nil)

context.WithValue 方法允许上下文携带请求范围的数据，这些数据必须是安全的，以便多个goroutine同时使用。这里的数据，更多是面向请求的元数据，而不应该作为函数的可选参数来使用(比如context里挂了一个sql.Tx对象，传递到Dao层使用)，因为元数据相对函数参数更多是隐含的，面向请求的。而参数更多是显示的。
同一个context对象可以传递给在不同的goroutine中运行的函数；上下文对于多个goroutine同时使用是安全的。对于值类型最容易犯错的地方，在于context value 应该是不可修改的，每次重新赋值应该是新的context，即： `context.WithValue(ctx, oldvalue)`，所以这里就是一个麻烦的地方，如果有多个key/value ，就需要多次调用`context.WithValue`， 为了解决这个问题，https://pkg.go.dev/google.golang.org/grpc/metadata  在grpc源码中使用了一个metadata.

`func FromIncomingContext(ctx context.Context) (md MD, ok bool)`  这里的md 就是一个map `type MD map[string][]string` 这样对于多个key/value的时候就可以用这个MD 一次把多个对象挂进去，不过这里需要注意：如果一个groutine从ctx中读出这个map对象是不能直接修改的。因为如果这个时候ctx被传递给了多个gouroutine， 如果直接修改就会导致data race, 因此需要使用copy-on-write的思路，解决跨多个goroutine使用数据，修改数据的场景。


比如如下场景：

新建一个context.Background() 的ctx1, 携带了一个map 的数据， map中包含了k1:v1 的键值对，ctx1 作为参数传递给了两个goroutine,其中一个goroutine从ctx1中获取map1，构建一个新的map对象map2,复制所有map1的数据，同时追加新的数据k2:v2 键值对，使用context.WithValue 创建新的ctx2,ctx2 会继续传递到其他groutine中。 这样各自读取的副本都是自己的数据，写行为追加的数据在ctx2中也能完整的读取到，同时不会污染ctx1中的数据，这种处理方式就是典型的COW(COPY ON Write)


### context cancel

当一个context被取消时， 从它派生的所有context也将被取消。WithCancel(ctx)参数认为是parent ctx， 在内部会进行一个传播关系链的关联。Done() 返回一个chan，当我们取消某个parent context， 实际上会递归层层cancel掉自己的chaild context 的done chan 从而让整个调用链中所有监听cancel的goroutine退出

下面是官网的例子,稍微调整了一下代码：

```
package main

import (
	"context"
	"fmt"
)

func main() {
	// gen generates integers in a separate goroutine and
	// sends them to the returned channel.
	// The callers of gen need to cancel the context once
	// they are done consuming generated integers not to leak
	// the internal goroutine started by gen.
	gen := func(ctx context.Context) <-chan int {
		dst := make(chan int)
		n := 1
		go func() {
			for {
				select {
				case <-ctx.Done():
					return // returning not to leak the goroutine
				case dst <- n:
					n++
				}
			}
		}()
		return dst
	}

	ctx, cancel := context.WithCancel(context.Background())

	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			cancel()
		}
	}
}
```

如果实现一个超时控制，通过上面的context的parent/child 机制， 其实只需要启动一个定时器，然后再超时的时候，直接将当前的context给cancel掉，就可以实现监听在当前和下层的context.Done()和goroutine的退出。


```
package main

import (
	"context"
	"fmt"
	"time"
)

const shortDuration = 1 * time.Millisecond

func main() {
	d := time.Now().Add(shortDuration)
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// Even though ctx will be expired, it is good practice to call its
	// cancellation function in any case. Failure to do so may keep the
	// context and its parent alive longer than necessary.
	defer cancel()

	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}

}
```

关于context 使用的规则总结：
- Incoming requests to a server should create a Context.
- Outgoing calls to servers should accept a Context.
- Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it.
- The chain of function calls between them must propagate the Context.
- Replace a Context using WithCancel, WithDeadline, WithTimeout, or WithValue.
- When a Context is canceled, all Contexts derived from it are also canceled.
- The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.
- Do not pass a nil Context, even if a function permits it. Pass a TODO context if you are unsure about which Context to use.
- Use context values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
- All blocking/long operations should be cancelable.
- Context.Value obscures your program’s flow.
- Context.Value should inform, not control.
- Try not to use context.Value.



## Channel

channels 是一种类型安全的消息队列，充当两个 goroutine 之间的管道，将通过它同步的进行任意资源的交换。channel 控制 goroutines 交互的能力从而创建了 Go 同步机制。当创建的 channel 没有容量时，称为无缓冲通道。反过来，使用容量创建的 channel 称为缓冲通道。

无缓冲 chan 没有容量，因此进行任何交换前需要两个 goroutine 同时准备好。当 goroutine 试图将一个资源发送到一个无缓冲的通道并且没有goroutine 等待接收该资源时，该通道将锁住发送 goroutine 并使其等待。当 goroutine 尝试从无缓冲通道接收，并且没有 goroutine 等待发送资源时，该通道将锁住接收 goroutine 并使其等待。
- Receive 先于Send发生
- 好处：100%保证能收到
- 代价：延迟时间未知



buffered channel 具有容量，因此其行为可能有点不同。当 goroutine 试图将资源发送到缓冲通道，而该通道已满时，该通道将锁住 goroutine并使其等待缓冲区可用。如果通道中有空间，发送可以立即进行，goroutine 可以继续。当goroutine 试图从缓冲通道接收数据，而缓冲通道为空时，该通道将锁住 goroutine 并使其等待资源被发送。
- Send先于Receive发生
- 好处：延迟更小
- 代价：不保证数据到达，越大的 buffer，越小的保障到达。buffer = 1 时，给你延迟一个消息的保障。

注意：
- channel的大小不代表性能和吞吐。吞吐是需要靠多线程，即多个消费的goroutine消费
- 注意：关于channel的close一定是发送者来操作。