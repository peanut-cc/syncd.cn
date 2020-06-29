---
title: "Go单元测试"
date: 2020-05-09T01:03:02+08:00
draft: false
author: "fan"
subtitle: "go单元测试"
tags: ["Go"]
categories: ["Go"]
toc:
  enable: true
  auto: true
---

Go标准库：testing 提供了单元测试和压力测试

## Golang 单元测试要求

1. 用来测试的代码必须以`_test.go`结尾
2. 单元测试的函数名必须以Test开头(Test后面的第一个字母也必须是大写)，并且只有一个参数，类型是`*testing.T`
3. 基准测试或压力测试必须以Benchmark开头(Benchmark后面的第一个字母也必须是大写)，并且只有一个参数,类型是`*testing.B`

go test ： 执行当前包下的所有单元测试 go test -v 测试并打印详细的输出 go test 包名： 执行这个包下面的所有测试用例 go test 源文件名 ： 执行这个测试源文件里的所有测试用例 go test -run ： 执行指定的测试用例

如果要执行当前包下的压力测试需要：go -test bench .



## 来个栗子

### 基本使用

我当前目录下有一个calc.go 的文件，内容如下：

```go
package main

func add(a, b int) int {
	return a + b
}

func sub(a, b int) int {
	return a - b
}

```



我们为这这两个函数写一个简单的单元测试，在当前目录下创建calc_test.go文件，写如下内容：

```go
package main

import "testing"

func TestAdd(t *testing.T) {
	var a = 10
	var b = 20
	c := add(a, b)
	if c != 30 {
		// t.Fatal 之后，就会退出这个单元测试
		t.Fatalf("invalid a+b, c=%d", c)
	}
	t.Logf("a = %d, b = %d, a + b = %d", a, b, c)
}

func TestSub(t *testing.T) {
	var a = 20
	var b = 10
	c := sub(a, b)
	if c != 10 {
		// t.Fatal 之后，就会退出这个单元测试
		t.Fatalf("invalid a-b, c=%d", c)
	}
	t.Logf("a = %d, b = %d, a - b = %d", a, b, c)
}

```

在当前目录下执行`go test`命令即可看到如下内容：

```bash
➜  unit_test go test
PASS
ok      awesomeProject/ex07/unit_test   0.001s
➜  unit_test 

```

上述命令并没有看到我们打印的日志，如果想要看到详细信息，可以执行`go test -v`

```bash
➜  unit_test go test -v
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
    calc_test.go:13: a = 10, b = 20, a + b = 30
=== RUN   TestSub
--- PASS: TestSub (0.00s)
    calc_test.go:24: a = 20, b = 10, a - b = 10
PASS
ok      awesomeProject/ex07/unit_test   0.001s
➜  unit_test 

```



## 一个测试例子的一步步完善

### 最简单版本

```go
package main

import "strings"

// Split slices s into all substrings separated by sep and
// returns a slice of the substrings between those separators
func Split(s, sep string) []string {
	var result []string
	i := strings.Index(s, sep)
	for i > -1 {
		result = append(result, s[:i])
		s = s[i+len(sep):]
		i = strings.Index(s, sep)
	}
	return append(result, s)
}
```

通过基本使用的例子，我们很容易写一个简单的测试，如下：

```go
func TestSplit(t *testing.T) {
	got := Split("a/b/c", "/")
	want := []string{"a", "b", "c"}
	if !reflect.DeepEqual(want, got) {
		t.Fatalf("expected: %v, got: %v", want, got)
	}
}
```

但是其实我们上面这个单元测试是不全面的，很多情况测试不到。

### 改进版本1

```go
func TestSplit(t *testing.T) {
	type test struct {
		input string
		sep   string
		want  []string
	}
	tests := []test{
		{input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
		{input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
		{input: "abc", sep: "/", want: []string{"abc"}},
		{input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
	}

	for _, tc := range tests {
		got := Split(tc.input, tc.sep)
		if !reflect.DeepEqual(tc.want, got) {
			t.Fatalf("expected: %v, got: %v", tc.want, got)
		}
		t.Logf("test success")
	}
}
```

这个版本中我们增加测试的情况，但是这个如果存在测试失败的情况，看起来就不是非常直观，就如上我们的测试，执行结果如下：

```bash
=== RUN   TestSplit
--- FAIL: TestSplit (0.00s)
    split_test.go:36: test success
    split_test.go:36: test success
    split_test.go:36: test success
    split_test.go:34: expected: [a b c], got: [a b c ]
FAIL

Process finished with exit code 1
```

这样当测试出现问题的时候我们还需要取看测试代码，找到对应的行，看看测试数据是做了什么测试，那么继续改进

### 改进版本2

我们给test结构体增加一个字段name来描述测试说明，在测试日志进行输出，这样就非常直观的从结果中看到测试是哪些成功了，哪些是失败了

```go
func TestSplit(t *testing.T) {
	type test struct {
		name  string
		input string
		sep   string
		want  []string
	}
	tests := []test{
		{name: "simple", input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
		{name: "wrong sep", input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
		{name: "no sep", input: "abc", sep: "/", want: []string{"abc"}},
		{name: "trailing sep", input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
	}

	for _, tc := range tests {
		got := Split(tc.input, tc.sep)
		if !reflect.DeepEqual(tc.want, got) {
			t.Fatalf("%s: expected: %v, got: %v", tc.name, tc.want, got)
		}
		t.Logf("%s test success", tc.name)
	}
}
```

测试结果如下：

```bash
=== RUN   TestSplit
--- FAIL: TestSplit (0.00s)
    split_test.go:62: simple test success
    split_test.go:62: wrong sep test success
    split_test.go:62: no sep test success
    split_test.go:60: trailing sep: expected: [a b c], got: [a b c ]
FAIL

Process finished with exit code 1
```

这个时候思考一下，你平时开发中有没有碰到过一种bug，只有在某中操作中才会触发，或者在某种执行顺序的时候会出现bug,我们上面的测试是存在列表中，所以循环的时候是顺序执行，那么继续改造。

### 改进版本3

```go
func TestSplit(t *testing.T) {
   tests := map[string]struct {
      input string
      sep   string
      want  []string
   }{
      "simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
      "wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
      "no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
      "trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
   }
   for name, tc := range tests {
      got := Split(tc.input, tc.sep)
      if !reflect.DeepEqual(tc.want, got) {
         t.Fatalf("%s: expected: %v, got: %v", name, tc.want, got)
      }
      t.Logf("%s test success", name)
   }
}
```

因为map本身是无序的，所以我们可以很好的利用这个特性，但是这个时候因为我们在测试失败的时候用的是t.Fatalf，这个会导致测试失败就退出，而因为无序的，很可能正好是有问题的测试第一个执行，那么后面的测试都无法执行，这个时候可以把t.Fatalf改为t.Errorf

而从go1.7之后，还可以使用sub tests,更改代码如下：

```go
func TestSplit(t *testing.T) {
	tests := map[string]struct {
		input string
		sep   string
		want  []string
	}{
		"simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
		"wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
		"no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
		"trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
	}
	for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got := Split(tc.input, tc.sep)
            if !reflect.DeepEqual(tc.want, got) {
                t.Fatalf("expected:%v, got:%v", tc.want, got)
            }
        })
        t.Logf("%s test success", name)
	}
}
```

测试结果如下：

```bash
=== RUN   TestSplit
=== RUN   TestSplit/simple
=== RUN   TestSplit/wrong_sep
=== RUN   TestSplit/no_sep
=== RUN   TestSplit/trailing_sep
--- FAIL: TestSplit (0.00s)
    --- PASS: TestSplit/simple (0.00s)
    split_test.go:90: simple test success
    --- PASS: TestSplit/wrong_sep (0.00s)
    split_test.go:90: wrong sep test success
    --- PASS: TestSplit/no_sep (0.00s)
    split_test.go:90: no sep test success
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:87: expected:[a b c], got:[a b c ]
    split_test.go:90: trailing sep test success
FAIL

```

不过现在我们这个输出日志可以改一下

```go
t.Fatalf("expected:%#v, got:%#v", tc.want, got)
```

输出结果为：

```go
split_test.go:87: expected:[]string{"a", "b", "c"}, got:[]string{"a", "b", "c", ""}
```

### go-cmp使用

https://github.com/google/go-cmp  这个库提供了非常好的功能，可以让我们test打印的时候更加清楚明了，直接看例子：

```go
func TestSplit(t *testing.T) {
	tests := map[string]struct {
		input string
		sep   string
		want  []string
	}{
		"simple":       {input: "a/b/c", sep: "/", want: []string{"a", "b", "c"}},
		"wrong sep":    {input: "a/b/c", sep: ",", want: []string{"a/b/c"}},
		"no sep":       {input: "abc", sep: "/", want: []string{"abc"}},
		"trailing sep": {input: "a/b/c/", sep: "/", want: []string{"a", "b", "c"}},
	}
	for name, tc := range tests {
		t.Run(name, func(t *testing.T) {
			got := Split(tc.input, tc.sep)
			diff := cmp.Diff(tc.want, got)
			if diff != "" {
				t.Fatalf(diff)
			}
		})

	}
}

```



测试结果如下：

```bash
=== RUN   TestSplit
=== RUN   TestSplit/wrong_sep
=== RUN   TestSplit/no_sep
=== RUN   TestSplit/trailing_sep
=== RUN   TestSplit/simple
--- FAIL: TestSplit (0.00s)
    --- PASS: TestSplit/wrong_sep (0.00s)
    --- PASS: TestSplit/no_sep (0.00s)
    --- FAIL: TestSplit/trailing_sep (0.00s)
        split_test.go:85:   []string{
              	"a",
              	"b",
              	"c",
            + 	"",
              }
    --- PASS: TestSplit/simple (0.00s)
FAIL

Process finished with exit code 1

```



这样从测试结果中我们就能很清楚知道是哪里的问题了



## 延伸阅读

[go-cmp](https://pkg.go.dev/github.com/google/go-cmp/cmp?tab=doc)

[TheThreeRulesOfTdd](http://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd)

