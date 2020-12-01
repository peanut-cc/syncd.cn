# Go进阶笔记-关于error


很多人对于Go的error比较吐槽，说代码中总是会有大量的如下代码：

```
if err != nil {
    ...
}
```

其实很多时候是使用的姿势不对，或者说，对于error的用法没有完全理解，这里整理一下关于Go中的error

## 关于源码中的error

先看一下go源码中`go/src/builtin/builtin.go`对于error的定义：


```
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
        Error() string
}
```

我们使用的时候经常会通过errors.New() 来返回一个error对象，这里可以看一下我们调用errors.New()的这段源码文件`go/src/errors/errors.go`,可以看到errorString实现了error解接口，而errors.New()其实返回的是一个 `&errorString{text}` 即errorString对象的指针。


```
package errors

// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

如果之前看过一些优秀源码或者go源码的，会发现代码中通常会定义很多自定义的error，并且都是包级别的变量，即变量名首字母大写:

```
// https://golang.org/pkg/bufio


var (
    ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
    ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
    ErrBufferFull        = errors.New("bufio: buffer full")
    ErrNegativeCount     = errors.New("bufio: negative count")
)
```

注意：自己之后在代码中关于这种自定义错误的定义，也要参照这种格式规范定义。
**"当前的包名：错误信息"**


```
package main

import (
	"errors"
	"fmt"
)

type errorString string

// 实现 error 接口
func (e errorString) Error() string {
	return string(e)
}

func New(text string) error {
	return errorString(text)
}

var errNamedType = New("EOF")
var ErrStructType = errors.New("EOF")

func main() {
	//  这里其实就是两个结构体值的比较
	if errNamedType == New("EOF") {
		fmt.Println("Named Type Error")   // 这行打印会输出
	}
	// 标准库中errors.New() 返回的是一个地址，每次调用都会返回一个新的内存地址
	// 标准库这样设计也是为了避免碰巧如果两个结构体值相同了，而引发一些不期望的问题
	if ErrStructType == errors.New("EOF") {
		fmt.Println("Struct Type Error")  // 这行打印不会输出
	}
}
```

关于结构体值的比较：

如果两个结构体值的类型均为可比较类型，则它们仅在它们的类型相同或者它们的底层类型相同（要考虑字段标签）并且其中至少有一个结构体值的类型为非定义类型时才可以互相比较。
如果两个结构体值可以相互比较，则它们的比较结果等同于逐个比较它们的相应字段


**注意：关于Go中函数支持多参数返回，如果函数有error的通常把返回值的最后一个参数作为error**

如果一个函数返回（value, error）这个时候必须先判定error
Go中的panic 意味着程序挂了不能继续运行了，不能假设调用者来解决panic。

对于刚学习go的时候经常用如下代码开启一个goroutine执行任务：

```
go func() {
    ...
}
```

这种情况也叫野生goroutine,并且这个时候recover是不能解决的。

可以定义一个包,通过调用该包中的Go() 方法来开goroutine，来避免野生goroutine。

```
package sync

func Go(x func()) {
    
    if err := recover(); err != nil {
        ....
    }
    go x()
}
```

关于代码的panic 通常在代码中是很少使用的，只有在极少情况下，我们需要panic，如我们项目的初始化地方连接数据库连接不上，并且这个时候，数据库是我们程序的强依赖，那么这个时候是可以panic。

下面通过一个例子来演示error的使用姿势：


```
package main

import (
	"errors"
	"fmt"
)

// 判断正负数
func Positivie(n int) (bool, error) {
	if n == 0 {
		return false, errors.New("undefined")
	}
	return true, nil
}

func Check(n int) {
	pos, err := Positivie(n)
	if err != nil {
		fmt.Println(n, err)
		return
	}
	if pos {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}

```

上面是一种非常正确的姿势，我们通过返回`(value, error)` 这种方式来解决，也是非常go 的一种写法，只有`err!=nil` 的时候我们的`value`才有意义

那么在实际中可能有很多各种姿势来解决上述的问题，如下：


```
package main

import "fmt"

func Positive(n int) *bool {
	if n == 0 {
		return nil
	}
	r := n > -1
	return &r
}

func Check(n int) {
	pos := Positive(n)
	if pos == nil {
		fmt.Println(n, "is neither")
		return
	}
	if *pos {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}
```

另外一种姿势：

```
package main

import "fmt"

func Positive(n int) bool {
	if n == 0 {
		panic("undefined")
	}
	return n > -1
}

func Check(n int) {
	defer func() {
		if recover() != nil {
			fmt.Println("is neither")
		}
	}()
	
	if Positive(n) {
		fmt.Println(n, "is positive")
	} else {
		fmt.Println(n, "is negative")
	}
}

func main() {
	Check(1)
	Check(0)
	Check(-1)
}

```

上面这两种姿势虽然也可以实现这个功能，但是非常的不好，也不推荐使用。在代码中尽可能还是使用`(value, error)` 这种返回值来解决error的情况。

对于真正意外的情况，那些不可恢复的程序错误，例如索引越界，不可恢复的环境问题，栈溢出等才会使用panic,对于其他的情况我们应该还是期望使用error来进行判定。 

## error 处理套路

### Sentinel Error 预定义error

通常我们把代码包中如下的这种error叫预定义error.

```
// https://golang.org/pkg/bufio


var (
    ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
    ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
    ErrBufferFull        = errors.New("bufio: buffer full")
    ErrNegativeCount     = errors.New("bufio: negative count")
)
```

这种姿势的缺点：

- 对于这种错误，在实际中的使用中我们通常会使用 `if err == ErrSomething {....}` 这种姿势来进行判断。但是也不得不说，这种姿势是最不灵活的错误处理策略，并且不能对于错误提供有用的上下文。
- Sentinel errors 成为API的公共部分。如果你的公共函数或方法返回一个特定值的错误，那么该错误就必须是公共的，当然要有文档记录，这最终会增加API的表面积。
- Sentinel errors 在两个包之间创建了依赖。对于使用者不得不导入这些错误，这样就在两个包之间建立了依赖关系，当项目中有许多类似的导出错误值时，存在耦合，项目中的其他包必须导入这些错误值才能检查特定的错误条件。


### Error types

Error type 是实现了error接口的自定义类型，例如MyError类型记录了文件和行号以展示发生了什么


```
type MyError struct {
    Msg string
    File string
    Line int
}

func (e *MyError) Error() string {
    return fmt.Sprintf("%s:%d:%s", e.File,e.Line, e.Msg)
}

func test() error {
    return &MyError("something happened", "server.go", 11)
}

func main() {
    err := test()
    switch err := err.(type){
    case nil:
        // ....
    case *MyError:
        fmt.Println("error occurred on line:", err.Line)
    default:
        // ....
    }
}

```

这种方式其实在标准库中也有使用如os.PathError


```
// https://golang.org/pkg/os/#PathError

type PathError struct {
    Op   string
    Path string
    Err  error
}
```

调用者要使用类型断言和类型switch，就要让自定义的error变成public，这种模型会导致和调用者产生强耦合，从而导致API变得脆弱。

### Opaque errors

这种方式也称为不透明处理，这也是相对来说比较优雅的处理方式,如下

```
func fn() error {
    
    x, err := bar.Foo()
    if err != nil {
        return err
    }
    // use x
}
```

这种不透明的实现方式，一种比较好的用法，这里以net库的代码来看：

```
// https://golang.org/pkg/net/#Error

type Error interface {
    error
    Timeout() bool   // Is the error a timeout?
    Temporary() bool // Is the error temporary?
}
```

这里是定义了一个Error接口，而让其他需要用到error的来实现这个接口，如net中的下面这个错误

```
// https://golang.org/pkg/net/#DNSConfigError

type DNSConfigError
    func (e *DNSConfigError) Error() string
    func (e *DNSConfigError) Temporary() bool
    func (e *DNSConfigError) Timeout() bool
    func (e *DNSConfigError) Unwrap() error
```

按照这个方式实现我们使用net时的异常处理可能就是如下情况：

```
if neerr, ok := err.(net.err); ok && nerr.Temporary() {
    time.Sleep(time.Second * 10)
    continue
}
if err != nil {
    log.Fatal(err)
}
```

其实这样还是不够优雅，好的方式是我们卡一定义temporary的接口，然后取实现这个接口，这样整体代码就看着非常简洁清楚，对外我们就只需要暴露IsTemporary方法即可，而不用外部再进行断言。

```
Type temporary interface {
    Temporary() bool
}

func IsTemporary(err error) bool {
    te, ok := err.(temporary)
    return ok && te.Temporary()
}

```

以上这几种姿势，其实各有各的用处，不同的场景，选择可能也不同，需要根据实际场景实际分析。


###  一个error 技巧使用例子

先看一段代码，相信这段代码如果很多人实现的时候也都是这个样子：


```
type Header struct {
    Key, Value string
}

type Status struct {
    Code    int
    Reason string
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
    
    _, err := fmt.Fprintf(w, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)
    if err != nil {
        return err
    }
    
    for _, h := range headers {
        _, err := fmt.Fprintf(w, "%s:%s\r\n", h.Key, h.Value)
        if err != nil {
            return err
        }
    }
    
    if _, err := fmt.Fprint(w, "\r\n"); err != nil {
        return err
    }
    
    _, err = io.Copy(w, body)
    return err
}
```

看这段代码时候估计很多就开始吐嘈go的error的处理，感觉代码中会存在很多err的判断处理，其实这里是可以写的更优雅一点的，上面的姿势不对，来换个姿势：

```
type errWriter struct {
    io.Writer
    err error
}

func(e *errWriter) Write(buf []byte) (int, error) {
    if e.err != nil {
        return 0, e.err
    }
    
    var n int
    n, e.err = e.Writer.Write(buf)
    return n,nil
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
    ew :=&errWriter{Writer:w}
    fmt.Fprintf(ew, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)
    
    for _, h := range headers {
        fmt.Fprintf(ew, "%s:%s\r\n", h.Key, h.Value)
    }
    
    fmt.Fprint(w, "\r\n")
    
    io.Copy(w, body)
    return ew.err
}
```

对比之下这种代码看起来是不是就非常简洁，所有很多时候可能是自己写代码的姿势不对，而不是go的error设计的不好。

## Wrap errors

就像下面这段代码一样，这样的使用方式，我自己在工程代码中也经常看到，这样就会导致生成的错误没有`file:line`信息,没有导致错误的调用堆栈信息，如果出现异常就非常不方便排查到底是哪里导致的问题，其次因为这里通过`fmt.Errorf`对错误进行了包装，也就破坏了原始错误。

```
func AuthenticateReuest(r *Request) error {
	err := authenticate(r.User)
	if err != nil {
		return fmt.Errorf("authenticate failed:%v", err)
	}
	return nil
}
```

关于error的处理中还有一个非常重要的地方就是是否是每次出现`err!=nil`的时候，我们都需要打印日志？ 如果这样做了，你会发现到处在打印日志，还有很多地方可能打印的是相同的日志。

```
func WriteAll(w io.Writer, buf[]byte) error {
	_, err := w.Write(buf)
	if err != nil {
		log.Println("unalbe to write:",err)  //这里记录了日志
		return err                           //将日志进行上抛给调用者
	}
	return nil
}

func WriteConfig(w io.Writer, conf *Config) error {
    buf, err := json.Marshal(conf)
    if err != nil {
        log.Printf("cound not marshal config:%v", err)
        return err
    }
    if err := WriteAll(w, buf); err != nil {
        log.Println("cound not write config:%v",err)
        return err
    }
    return nil
}
```

在上面这个例子中, 这个错误逐层返回给调用者，如果处理不好，可能就像上面这个例子，每次都打印日志，一直到程序的顶部
所以：error应该只被处理一次。
Go中错误的处理契约规定：在出现错误的情况下，不能对其他返回值的内容做任何假设,如下代码中，由于json序列化失败，buf的内容是未知的，这个时候把损坏的buf传给后续处理逻辑，这样就会导致一些未知的错误发生。


```
func WriteConfig(w io.Writer, conf *Config) error {
    buf, err := json.Marshal(conf)
    if err != nil {
        log.Printf("cound not marshal config:%v", err)
        // 忘记return
    }
    if err := WriteAll(w, buf); err != nil {
        log.Println("cound not write config:%v",err)
        return err
    }
    return nil
}
```

关于错误日志处理的规则：

- 错误要被日志记录
- 应用程序处理错误，保证100%的完整性
- 之后不再报告当前错误

`github.com/pkg/errors` 这个error处理包非常受欢迎，看一下这个包对错误的处理例子：


```
package main

import (
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"

	"github.com/pkg/errors"
)

func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, errors.Wrap(err, "open failed")
	}
	defer f.Close()
	buf, err := ioutil.ReadAll(f)
	if err != nil {
		return nil, errors.Wrap(err, "read failed")
	}
	return buf, nil
}

func ReadConfig() ([]byte, error) {
	home := os.Getenv("HOME")
	config, err := ReadFile(filepath.Join(home, ".settings.xml"))
	return config, errors.WithMessage(err, "cound not read config")
}

func main() {
	_, err := ReadConfig()
	if err != nil {
		fmt.Printf("original err:%T %v\n", errors.Cause(err), errors.Cause(err))
		fmt.Printf("stack trace:\n %+v\n",err)  // %+v 可以在打印的时候打印完整的堆栈信息
		os.Exit(1)
	}
}

```

执行结果如下：


```
original err:*os.PathError open /Users/zhaofan/.settings.xml: no such file or directory
stack trace:
 open /Users/zhaofan/.settings.xml: no such file or directory
open failed
main.ReadFile
        /Users/zhaofan/open_source_study/test_code/202012/wrap_errors/main.go:15
main.ReadConfig
        /Users/zhaofan/open_source_study/test_code/202012/wrap_errors/main.go:27
main.main
        /Users/zhaofan/open_source_study/test_code/202012/wrap_errors/main.go:32
runtime.main
        /Users/zhaofan/app/go/src/runtime/proc.go:204
runtime.goexit
        /Users/zhaofan/app/go/src/runtime/asm_amd64.s:1374
cound not read config
exit status 1
```

从代码上也非常简洁，处理的非常优雅，最终不管是错误信息还是堆栈信息，还可以添加自定义的上下文，同时也完全满足上面提出的关于错误日志处理的规则。
关于代码中的`Wrap`源码如下：

```
// Wrap returns an error annotating err with a stack trace
// at the point Wrap is called, and the supplied message.
// If err is nil, Wrap returns nil.
func Wrap(err error, message string) error {
	if err == nil {
		return nil
	}
	err = &withMessage{
		cause: err,
		msg:   message,
	}
	return &withStack{
		err,
		callers(),
	}
}
```

可以看到我们每次调用`errors.Wrap`方法的时候都是把我们的错误信息err存入到`withMessage`结构体的cause字段，同时又把包装的`withMessage` 作为err存到`withStack`结构体中，同时`withStack`包含了调用堆栈的信息

```
type withMessage struct {
	cause error
	msg   string
}
```

### 关于`github.com/pkg/errors` 使用姿势

- 在**你自己的应用程序**中，使用errors.New或者errors.Errorf返回错误
- 如果调用其他**包内的函数或者你当前项目里的其他函数**，通常简单的直接返回，即直接`return err`
- 如果你使用**第三方库如github库，公司的基础库，或者go的基础库**，这个时候应该使用`errors.Wrap`或者`errors.Wrap`f保存堆栈信息，同时添加自定义的上下文信息
- 直接返回错误，而不是每个错误产生的地方打日志
- 在程序的顶部或者工作的`goroutine`顶部(请求入口)使用`%+v`把堆栈详情记录
- 使用`errors.Cause` 获取root error即根因，在进行和`sentinel error`进行等值判定
- 一旦错误被处理,包括你打印日志，或者降级处理等，这个时候你就不应该再向上抛出err，而应该return nil.


## go1.13 中的errors

go 1.13 为errors和fmt标准库引入了新的特性，以简化处理包含其他错误的错误。其中最重要的就是：包含一个错误的error可以实现返回底层错误的Unwrap 方法。如果e1.Unwrap() 返回e2, 那么e1就包装了e2,就可以展开e1以获取e2

在Go的1.13 中`fmt.Errorf`支持新的`%w` ，这样就在错误信息中带入原始的信息,这样既保证了人阅读的方便，也方便了机器处理，如：


```
if err != nil {
    return fmt.Errorf("access denied %w", ErrrPermission)
}
```

把之前的例子进行调整如下：

```
package main

import (
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"

	"errors"
)


func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open failed: %w", err)
	}

	defer f.Close()
	buf, err := ioutil.ReadAll(f)
	if err != nil {
		return nil, fmt.Errorf("read failed: %w", err)
	}
	return buf, nil
}


func ReadConfig() ([]byte, error) {
	home := os.Getenv("HOME")
	config, err := ReadFile(filepath.Join(home, ".settings.xml"))
	return config, fmt.Errorf("cound not read config: %w", err)
}

func main() {
	_, err := ReadConfig()
	if err != nil {
		// errors.Is会一层一层的展开，找最内层的err
		fmt.Println(errors.Is(err, os.ErrNotExist))
		os.Exit(1)
	}
}

```

但是1.13的errors有个非常大的问题就是不支持携带堆栈信息，所以最好的办法就是把标准库中的`errors`和`github.com/pkg/errors`


```
package main

import (
	"errors"
	"fmt"

	xerrors "github.com/pkg/errors"
)

var errMy = errors.New("My Error")

func test0() error {
	return xerrors.Wrapf(errMy, "test0 failed")
}

func test1() error {
	return test0()
}

func test2() error {
	return test1()
}

func main() {
	err := test2()
	fmt.Printf("main: %+v\n", err)
	fmt.Println(errors.Is(err, errMy))
}
```

其实原则就是我们底层的错误还是通过 `github.com/pkg/errors` 中`Wrapf` 进行包装。并且这个时候也完全兼容标准库中的errors，可以使用`errors.Is` 和 `errors.As`方法做判断处理
