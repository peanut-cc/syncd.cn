---
title: "从别人的代码中学习golang系列--01"
date: 2020-06-24T23:48:11+08:00
draft: false
author: "fan"
subtitle: "Go"
tags: ["Go"]
categories: ["Go"]
toc:
  enable: true
  auto: true
---



​		自己最近在思考一个问题，如何让自己的代码质量逐渐提高，于是想到整理这个系列，通过阅读别人的代码，从别人的代码中学习，来逐渐提高自己的代码质量。本篇是这个系列的第一篇，我也不知道自己会写多少篇，但是希望自己能坚持下去。



第一个自己学习的源码是：https://github.com/LyricTian/gin-admin

自己整理的代码地址：https://github.com/peanut-pg/gin_admin 这篇文章整理的时候只是为了跑起来整体的代码，对作者的代码进行精简。

这篇博客主要是阅读gin-admin的第一篇，整理了从代码项目目录到日志库使用中学习到的内容：

1. 项目目录规范

2. 配置文件的加载

3. `github.com/sirupsen/logrus` 日志库在项目的使用

4. 项目的优雅退出

5. Golang的选项模式

   

## 项目目录规范

作者的项目目录还是非常规范的，应该也是按照https://github.com/golang-standards/project-layout 规范写的，这个规范虽然不是官方强制规范的，但是确实很多开源项目都在采用的，所以我们在生产中正式的项目都应该尽可能遵循这个目录规范标准进行代码的编写。关于这个目录的规范使用，自己会在后续实际使用中逐渐完善。



### /cmd

main函数文件（比如 `/cmd/myapp.go`）目录，这个目录下面，每个文件在编译之后都会生成一个可执行的文件。

不要把很多的代码放到这个目录下面，这里面的代码尽可能简单。



### `/internal`

应用程序的封装的代码。我们的应用程序代码应该放在 `/internal/app `目录中。而这些应用程序共享的代码可以放在 `/internal/pkg`目录中



### `/pkg`

一些通用的可以被其他项目所使用的代码，放到这个目录下面。



### `/vendor`

应用程序的依赖项，go mod vendor 命令可以创建vendor目录。



**Service Application Directories**



### `/api`

协议文件，`Swagger/thrift/protobuf` 等



**Web Application Directories**



### `/web`

web服务所需要的静态文件



**Common Application Directories**



### `/configs`

配置文件目录



### `/init`

系统的初始化



### `/scripts`

用于执行各种构建，安装，分析等操作的脚本。



### `/build`

打包和持续集成



### `/deployments`

部署相关的配置文件和模板



### `/test`

其他测试目录，功能测试，性能测试等



**Other Directories**

### `/docs`

设计和用户文档



### `/tools`

常用的工具和脚本，可以引用 `/internal` 或者 `/pkg` 里面的库



### `/examples`

应用程序或者公共库使用的一些例子



### `/assets`

其他一些依赖的静态资源



## 配置文件的加载

作者的gin-admin 项目中配置文件加载库使用的是：github.com/koding/multiconfig

在这之前，听到使用最多的就是大名鼎鼎的viper ，但是相对于viper相对来说就比较轻便了，并且功能还是非常强大的。感兴趣的可以看看这篇文章：https://sfxpt.wordpress.com/2015/06/19/beyond-toml-the-gos-de-facto-config-file/

### 栗子

程序目录结构为：

```bash
configLoad
├── configLoad
├── config.toml
└── main.go

```



通过一个简单的例子来看看multiconfig的使用

```go
package main

import (
	"fmt"

	"github.com/koding/multiconfig"
)

type (
	// Server holds supported types by the multiconfig package
	Server struct {
		Name     string
		Port     int `default:"6060"`
		Enabled  bool
		Users    []string
		Postgres Postgres
	}

	// Postgres is here for embedded struct feature
	Postgres struct {
		Enabled           bool
		Port              int
		Hosts             []string
		DBName            string
		AvailabilityRatio float64
	}
)

func main() {
	m := multiconfig.NewWithPath("config.toml") // supports TOML and JSON

	// Get an empty struct for your configuration
	serverConf := new(Server)

	// Populated the serverConf struct
	m.MustLoad(serverConf) // Check for error

	fmt.Println("After Loading: ")
	fmt.Printf("%+v\n", serverConf)

	if serverConf.Enabled {
		fmt.Println("Enabled field is set to true")
	} else {
		fmt.Println("Enabled field is set to false")
	}
}

```



配置文件config.toml内容为：

```toml
Name              = "koding"
Enabled           = false
Port              = 6066
Users             = ["ankara", "istanbul"]

[Postgres]
Enabled           = true
Port              = 5432
Hosts             = ["192.168.2.1", "192.168.2.2", "192.168.2.3"]
AvailabilityRatio = 8.23
```



编译之后执行，效果如下：

```bash
➜  configLoad ./configLoad 
After Loading: 
&{Name:koding Port:6066 Enabled:false Users:[ankara istanbul] Postgres:{Enabled:true Port:5432 Hosts:[192.168.2.1 192.168.2.2 192.168.2.3] DBName: AvailabilityRatio:8.23}}
Enabled field is set to false
➜  configLoad ./configLoad -h
Usage of ./configLoad:
  -enabled
        Change value of Enabled. (default false)
  -name
        Change value of Name. (default koding)
  -port
        Change value of Port. (default 6066)
  -postgres-availabilityratio
        Change value of Postgres-AvailabilityRatio. (default 8.23)
  -postgres-dbname
        Change value of Postgres-DBName.
  -postgres-enabled
        Change value of Postgres-Enabled. (default true)
  -postgres-hosts
        Change value of Postgres-Hosts. (default [192.168.2.1 192.168.2.2 192.168.2.3])
  -postgres-port
        Change value of Postgres-Port. (default 5432)
  -users
        Change value of Users. (default [ankara istanbul])

Generated environment variables:
   SERVER_ENABLED
   SERVER_NAME
   SERVER_PORT
   SERVER_POSTGRES_AVAILABILITYRATIO
   SERVER_POSTGRES_DBNAME
   SERVER_POSTGRES_ENABLED
   SERVER_POSTGRES_HOSTS
   SERVER_POSTGRES_PORT
   SERVER_USERS

flag: help requested
➜  configLoad ./configLoad -name=test
After Loading: 
&{Name:test Port:6066 Enabled:false Users:[ankara istanbul] Postgres:{Enabled:true Port:5432 Hosts:[192.168.2.1 192.168.2.2 192.168.2.3] DBName: AvailabilityRatio:8.23}}
Enabled field is set to false
➜  configLoad export SERVER_NAME="test_env"
➜  configLoad ./configLoad                 
After Loading: 
&{Name:test_env Port:6066 Enabled:false Users:[ankara istanbul] Postgres:{Enabled:true Port:5432 Hosts:[192.168.2.1 192.168.2.2 192.168.2.3] DBName: AvailabilityRatio:8.23}}
Enabled field is set to false
```



从上面的使用中，你能能够看到，虽然multiconfig 非常轻量，但是功能还是非常强大的，可以读配置文件，还可以通过环境变量，以及我们常用的命令行模式。



## 日志库在项目的使用

这个可能对很多初学者来说都是非常有用的，因为一个项目中，我们基础的就是要记录日志，golang有很多强大的日志库，如：作者的gin-admin 项目使用的`github.com/sirupsen/logrus`; 还有就是uber开源的`github.com/uber-go/zap`等等

这里主要学习一下作者是如何在项目中使用logrus，这篇文章对作者使用的进行了精简。当然只是去掉了关于gorm，以及mongo的hook的部分，如果你的项目中没有使用这些，其实也先不用关注这两个hook部分的代码，不影响使用，后续的系列文章也会对hook部分进行整理。

作者封装的logger库是在pkg/loggger目录中，我精简之后如下：

```go
package logger

import (
	"context"
	"fmt"
	"io"
	"os"
	"time"

	"github.com/sirupsen/logrus"
)

// 定义键名
const (
	TraceIDKey      = "trace_id"
	UserIDKey       = "user_id"
	SpanTitleKey    = "span_title"
	SpanFunctionKey = "span_function"
	VersionKey      = "version"
	StackKey        = "stack"
)

// TraceIDFunc 定义获取跟踪ID的函数
type TraceIDFunc func() string

var (
	version     string
	traceIDFunc TraceIDFunc
	pid         = os.Getpid()
)

func init() {
	traceIDFunc = func() string {
		return fmt.Sprintf("trace-id-%d-%s",
			os.Getpid(),
			time.Now().Format("2006.01.02.15.04.05.999999"))
	}
}

// Logger 定义日志别名
type Logger = logrus.Logger

// Hook 定义日志钩子别名
type Hook = logrus.Hook

// StandardLogger 获取标准日志
func StandardLogger() *Logger {
	return logrus.StandardLogger()
}

// SetLevel设定日志级别
func SetLevel(level int) {
	logrus.SetLevel(logrus.Level(level))
}

// SetFormatter 设定日志输出格式
func SetFormatter(format string) {
	switch format {
	case "json":
		logrus.SetFormatter(new(logrus.JSONFormatter))
	default:
		logrus.SetFormatter(new(logrus.TextFormatter))
	}
}

// SetOutput 设定日志输出
func SetOutput(out io.Writer) {
	logrus.SetOutput(out)
}

// SetVersion 设定版本
func SetVersion(v string) {
	version = v
}

// SetTraceIDFunc 设定追踪ID的处理函数
func SetTraceIDFunc(fn TraceIDFunc) {
	traceIDFunc = fn
}

// AddHook 增加日志钩子
func AddHook(hook Hook) {
	logrus.AddHook(hook)
}

type (
	traceIDKey struct{}
	userIDKey  struct{}
)

// NewTraceIDContext 创建跟踪ID上下文
func NewTraceIDContext(ctx context.Context, traceID string) context.Context {
	return context.WithValue(ctx, traceIDKey{}, traceID)
}

// FromTraceIDContext 从上下文中获取跟踪ID
func FromTraceIDContext(ctx context.Context) string {
	v := ctx.Value(traceIDKey{})
	if v != nil {
		if s, ok := v.(string); ok {
			return s
		}
	}
	return traceIDFunc()
}

// NewUserIDContext 创建用户ID上下文
func NewUserIDContext(ctx context.Context, userID string) context.Context {
	return context.WithValue(ctx, userIDKey{}, userID)
}

// FromUserIDContext 从上下文中获取用户ID
func FromUserIDContext(ctx context.Context) string {
	v := ctx.Value(userIDKey{})
	if v != nil {
		if s, ok := v.(string); ok {
			return s
		}
	}
	return ""
}

type spanOptions struct {
	Title    string
	FuncName string
}

// SpanOption 定义跟踪单元的数据项
type SpanOption func(*spanOptions)

// SetSpanTitle 设置跟踪单元的标题
func SetSpanTitle(title string) SpanOption {
	return func(o *spanOptions) {
		o.Title = title
	}
}

// SetSpanFuncName 设置跟踪单元的函数名
func SetSpanFuncName(funcName string) SpanOption {
	return func(o *spanOptions) {
		o.FuncName = funcName
	}
}

// StartSpan 开始一个追踪单元
func StartSpan(ctx context.Context, opts ...SpanOption) *Entry {
	if ctx == nil {
		ctx = context.Background()
	}
	var o spanOptions
	for _, opt := range opts {
		opt(&o)
	}
	fields := map[string]interface{}{
		VersionKey: version,
	}
	if v := FromTraceIDContext(ctx); v != "" {
		fields[TraceIDKey] = v
	}
	if v := FromUserIDContext(ctx); v != "" {
		fields[UserIDKey] = v
	}
	if v := o.Title; v != "" {
		fields[SpanTitleKey] = v
	}
	if v := o.FuncName; v != "" {
		fields[SpanFunctionKey] = v
	}

	return newEntry(logrus.WithFields(fields))

}

// Debugf 写入调试日志
func Debugf(ctx context.Context, format string, args ...interface{}) {
	StartSpan(ctx).Debugf(format, args...)
}

// Infof 写入消息日志
func Infof(ctx context.Context, format string, args ...interface{}) {
	StartSpan(ctx).Infof(format, args...)
}

// Printf 写入消息日志
func Printf(ctx context.Context, format string, args ...interface{}) {
	StartSpan(ctx).Printf(format, args...)
}

// Warnf 写入警告日志
func Warnf(ctx context.Context, format string, args ...interface{}) {
	StartSpan(ctx).Warnf(format, args...)
}

// Errorf 写入错误日志
func Errorf(ctx context.Context, format string, args ...interface{}) {
	StartSpan(ctx).Errorf(format, args...)
}

// Fatalf 写入重大错误日志
func Fatalf(ctx context.Context, format string, args ...interface{}) {
	StartSpan(ctx).Fatalf(format, args...)
}

// ErrorStack 输出错误栈
func ErrorStack(ctx context.Context, err error) {
	StartSpan(ctx).WithField(StackKey, fmt.Sprintf("%+v", err)).Errorf(err.Error())
}

// Entry 定义统一的日志写入方式
type Entry struct {
	entry *logrus.Entry
}

func newEntry(entry *logrus.Entry) *Entry {
	return &Entry{entry: entry}
}

func (e *Entry) checkAndDelete(fields map[string]interface{}, keys ...string) *Entry {
	for _, key := range keys {
		_, ok := fields[key]
		if ok {
			delete(fields, key)
		}
	}
	return newEntry(e.entry.WithFields(fields))
}

// WithFields 结构化字段写入
func (e *Entry) WithFields(fields map[string]interface{}) *Entry {
	e.checkAndDelete(fields,
		TraceIDKey,
		SpanTitleKey,
		SpanFunctionKey,
		VersionKey)
	return newEntry(e.entry.WithFields(fields))
}

// WithField 结构化字段写入
func (e *Entry) WithField(key string, value interface{}) *Entry {
	return e.WithFields(map[string]interface{}{key: value})
}

// Fatalf 重大错误日志
func (e *Entry) Fatalf(format string, args ...interface{}) {
	e.entry.Fatalf(format, args...)
}

// Errorf 错误日志
func (e *Entry) Errorf(format string, args ...interface{}) {
	e.entry.Errorf(format, args...)
}

// Warnf 警告日志
func (e *Entry) Warnf(format string, args ...interface{}) {
	e.entry.Warnf(format, args...)
}

// Infof 消息日志
func (e *Entry) Infof(format string, args ...interface{}) {
	e.entry.Infof(format, args...)
}

// Printf 消息日志
func (e *Entry) Printf(format string, args ...interface{}) {
	e.entry.Printf(format, args...)
}

// Debugf 写入调试日志
func (e *Entry) Debugf(format string, args ...interface{}) {
	e.entry.Debugf(format, args...)
}

```



通过`Logrus & Context` 实现了统一的 TraceID/UserID 等关键字段的输出，从这里也可以看出来，作者这样封装非常也符合符合pkg目录的要求，我们可以很容易的在internal/app 目录中使用封装好的logger。接着就看一下如何使用，作者在internal/app 目录下通过logger.go 中的InitLogger进行日志的初始化，设置了日志的级别，日志的格式，以及日志输出文件。这样我们在internal/app的其他包文件中只需要导入pkg下的logger即可以进行日志的记录。



## 项目的优雅退出

在这里不得不提的就是一个基础知识Linux信号signal,推荐看看https://blog.csdn.net/lixiaogang_theanswer/article/details/80301624

linux系统中signum.h中有对所有信号的宏定义，这里注意一下，我使用的是manjaro linux,我的这个文件路径是`/usr/include/bits/signum.h` 不同的linux系统可能略有差别，可以通过`find / -name signum.h` 查找确定

```c
#define SIGSTKFLT       16      /* Stack fault (obsolete).  */
#define SIGPWR          30      /* Power failure imminent.  */

#undef  SIGBUS
#define SIGBUS           7
#undef  SIGUSR1
#define SIGUSR1         10
#undef  SIGUSR2
#define SIGUSR2         12
#undef  SIGCHLD
#define SIGCHLD         17
#undef  SIGCONT
#define SIGCONT         18
#undef  SIGSTOP
#define SIGSTOP         19
#undef  SIGTSTP
#define SIGTSTP         20
#undef  SIGURG
#define SIGURG          23
#undef  SIGPOLL
#define SIGPOLL         29
#undef  SIGSYS
#define SIGSYS          31

```



在linux终端可以使用`kill -l` 查看所有的信号

```shell
➜  ~ kill -l
HUP INT QUIT ILL TRAP ABRT BUS FPE KILL USR1 SEGV USR2 PIPE ALRM TERM STKFLT CHLD CONT STOP TSTP TTIN TTOU URG XCPU XFSZ VTALRM PROF WINCH POLL PWR SYS
➜  ~ 

```



信号说明：

| **信号**  | **起源** |                **默认行为**                |                           **含义**                           |
| --------- | :------: | :----------------------------------------: | :----------------------------------------------------------: |
| SIGHUP    |  POSIX   |                    Term                    |                         控制终端挂起                         |
| SIGINT    |   ANSI   |                    Term                    |                键盘输入以终端进程（ctrl + C）                |
| SIGQUIT   |  POSIX   |                    Core                    |                键盘输入使进程退出（Ctrl + \）                |
| SIGILL    |   ANSI   |                    Core                    |                           非法指令                           |
| SIGTRAP   |  POSIX   |                    Core                    |                      断点陷阱，用于调试                      |
| SIGABRT   |   ANSI   |                    Core                    |                进程调用abort函数时生成该信号                 |
| SIGIOT    |  4.2BSD  |                    Core                    |                        和SIGABRT相同                         |
| SIGBUS    |  4.2BSD  |                    Core                    |                    总线错误，错误内存访问                    |
| SIGFPE    |   ANSI   |                    Core                    |                           浮点异常                           |
| SIGKILL   |  POSIX   |                    Term                    |            终止一个进程。该信号不可被捕获或被忽略            |
| SIGUSR1   |  POSIX   |                    Term                    |                      用户自定义信号之一                      |
| SIGSEGV   |   ANSI   |                    Core                    |                        非法内存段使用                        |
| SIGUSR2   |  POSIX   |                    Term                    |                       用户自定义信号二                       |
| SIGPIPE   |  POSIX   |                    Term                    |             往读端关闭的管道或socket链接中写数据             |
| SIGALRM   |  POSIX   |                    Term                    |           由alarm或settimer设置的实时闹钟超时引起            |
| SIGTERM   |   ANSI   |                    Term                    |         终止进程。kill命令默认发生的信号就是SIGTERM          |
| SIGSTKFLT |  Linux   |                    Term                    |        早期的Linux使用该信号来报告数学协处理器栈错误         |
| SIGCLD    | System V |                    Ign                     |                        和SIGCHLD相同                         |
| SIGCHLD   |  POSIX   |                    Ign                     |               子进程状态发生变化（退出或暂停）               |
| SIGCONT   |  POSIX   |                    Cont                    | 启动被暂停的进程（Ctrl+Q）。如果目标进程未处于暂停状态，则信号被忽略 |
| SIGSTOP   |  POSIX   |                    Stop                    |         暂停进程（Ctrl+S）。该信号不可被捕捉或被忽略         |
| SIGTSTP   |  POSIX   |                    Stop                    |                      挂起进程（Ctrl+Z）                      |
| SIGTTIN   |  POSIX   |                    Stop                    |                  后台进程试图从终端读取输入                  |
| SIGTTOU   |  POSIX   |                    Stop                    |                  后台进程试图往终端输出内容                  |
| SIGURG    | 4.3 BSD  |                    Ign                     |                  socket连接上接收到紧急数据                  |
| SIGXCPU   | 4.2 BSD  |                    Core                    |                进程的CPU使用时间超过其软限制                 |
| SIGXFSZ   | 4.2 BSD  |                    Core                    |                     文件尺寸超过其软限制                     |
| SIGVTALRM | 4.2 BSD  | Termhttps://github.com/LyricTian/gin-admin |   与SIGALRM类似，不过它只统计本进程用户空间代码的运行时间    |
| SIGPROF   | 4.2 BSD  |                    Term                    |      与SIGALRM 类似，它同时统计用户代码和内核的运行时间      |
| SIGWINCH  | 4.3 BSD  |                    Ign                     |                     终端窗口大小发生变化                     |
| SIGPOLL   | System V |                    Term                    |                         与SIGIO类似                          |
| SIGIO     | 4.2 BSD  |                    Term                    | IO就绪，比如socket上发生可读、可写事件。因为TCP服务器可触发SIGIO的条件很多，故而SIGIO无法在TCP服务器中用。SIGIO信号可用在UDP服务器中，但也很少见 |
| SIGPWR    | System V | Thttps://github.com/LyricTian/gin-adminerm |      对于UPS的系统，当电池电量过低时，SIGPWR信号被触发       |
| SIGSYS    |  POSIX   |                    Core                    |                         非法系统调用                         |
| SIGUNUSED |          |                    Core                    |                  保留，通常和SIGSYS效果相同                  |



作者中代码是这样写的：

```go
func Run(ctx context.Context, opts ...Option) error {
	var state int32 = 1
	sc := make(chan os.Signal, 1)
	signal.Notify(sc, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	cleanFunc, err := Init(ctx, opts...)
	if err != nil {
		return err
	}

EXIT:
	for {
		sig := <-sc
		logger.Printf(ctx, "接收到信号[%s]", sig.String())
		switch sig {
		case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
			atomic.CompareAndSwapInt32(&state, 1, 0)
			break EXIT
		case syscall.SIGHUP:
		default:
			break EXIT
		}
	}
    // 进行一些清理操作
	cleanFunc()
	logger.Printf(ctx, "服务退出")
	time.Sleep(time.Second)
	os.Exit(int(atomic.LoadInt32(&state)))
	return nil
}
```

这里监听了syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT，当收到这些信号的，则会进行系统的退出，同时在程序退出之前进行一些清理操作。

这里我们有一个知识点需要回顾一下：golang中的break label 和 goto label

- break label,break的跳转标签(label)必须放在循环语句for前面，并且在break label跳出循环不再执行for循环里的代码。break标签只能用于for循环
- goto label的label(标签)既可以定义在for循环前面,也可以定义在for循环后面，当跳转到标签地方时，继续执行标签下面的代码。



## Golang的选项模式

其实在很多的开源项目中都可以看到golang 选项模式的使用，推荐看看https://www.sohamkamani.com/golang/options-pattern/



假如我们现在想要建造一个房子，我们思考，建造房子需要木材，水泥......，假如我们现在就想到这两个，我们用代码实现：

```go
type House struct {
	wood   string // 木材
	cement string // 水泥
}

// 造一个房子
func NewHouse(wood, cement shttps://github.com/LyricTian/gin-admintring) *House {
	house := &House{
		wood:   wood,
		cement: cement,
	}
	return house
}
```

上面这种方式，应该是我们经常可以看到的实现方式，但是这样实现有一个很不好的地方，就是我们突然发现我们建造房子海需要钢筋，这个时候我们就必须更改NewHouse 函数的参数，并且NewHouse的参数还有顺序依赖。

对于扩展性来说，上面的这种实现放那格式其实不是非常好，而golang的选项模式很好的解决了这个问题。



### 栗子

```go
type HouseOption func(*House)

type House struct {
	Wood   string // 木材
	Cement string // 水泥
}

func WithWood(wood string) HouseOption {
	return func(house *House) {
		house.Wood = wood
	}
}

func WithCement(Cement string) HouseOption {
	return func(house *House) {
		house.Cement = Cement
	}
}

// 造一个新房子
func NewHouse(opts ...HouseOption) *House {
	h := &House{}
	for _, opt := range opts {
		opt(h)
	}
	return h
}
```



这样当我们这个时候发现，我建造房子还需要石头，只需要在House结构体中增加对应的字段，同时增加一个

`WithStone` 函数即可，同时我们我们调用NewHouse的地方也不会有顺序依赖，只需要增加一个参数即可，更改之后的代码如下：

```go
type HouseOption func(*House)

type House struct {
	Wood   string // 木材
	Cement string // 水泥
	Stone  string // 石头
}

func WithWood(wood string) HouseOption {
	return func(house *House) {
		house.Wood = wood
	}
}

func WithCement(Cement string) HouseOption {
	return func(house *House) {
		house.Cement = Cement
	}
}

func WithStone(stone string) HouseOption {
	return func(house *House) {
		house.Stone = stone
	}
}

// 造一个新房子
func NewHouse(opts ...HouseOption) *House {
	h := &House{}
	for _, opt := range opts {
		opt(h)
	}
	return h
}

func main() {
	house := NewHouse(
		WithCement("上好的水泥"),
		WithWood("上好的木材"),
		WithStone("上好的石头"),
	)
	fmt.Println(house)https://github.com/LyricTian/gin-admin
}
```



### 选项模式小结

- 生产中正式的并且较大的项目使用选项模式可以方便后续的扩展
- 增加了代码量，但让参数的传递更新清晰明了
- 在参数确实比较复杂的场景推荐使用选项模式





## 总结



从https://github.com/LyricTian/gin-admin 作者的这个项目中我们首先从大的方面学习了：

1. golang 项目的目录规范
2. github.com/koding/multiconfig 配置文件库
3. logrus 日志库的使用
4. 项目的优雅退出实现，信号相关知识
5. golang的选项模式

对于我自己来说，对于之后写golang项目，通过上面这些知识点，可以很快速的取构建一个项目的基础部分，也希望看到这篇文章能够帮到你，如果有写的不对的地方，也欢迎评论指出。



## 延伸阅读



- https://github.com/LyricTian/gin-admin
- https://github.com/golang-standards/project-layout
- https://sfxpt.wordpress.com/2015/06/19/beyond-toml-the-gos-de-facto-config-file/
- https://github.com/koding/multiconfig
- https://github.com/sirupsen/logrus
- https://blog.csdn.net/lixiaogang_theanswer/article/details/80301624
- https://www.sohamkamani.com/golang/options-pattern/