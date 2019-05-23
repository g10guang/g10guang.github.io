---
layout: post
title:  "Golang 错误异常处理 defer panic recover"
date:   2018-02-08 21:00:05 +0800
categories: golang
---

# Golang错误、异常

别的语言都有异常处理语法，以 python 的异常处理语法为例：

```python
try:
    # do something
    raise Exception
except KeyError as e:
    pass
except Exception as e:
    pass
else:
    pass
finally:
    pass
```

Golang 中并没有如此语法

**Golang 中错误和异常并不是一码事**

```golang
type error interface {
    Error() string
}
```

错误是实现了 `error` 接口的对象。

异常是由内置函数 `func panic(v interface{})` 发起的，可以被 `func recover() interface{}` 函数捕获

第三方或内置库函数最后一个函数返回 `error` 对象，调用者可以查看 `err != nil` 来判断本次调用是否成功执行，这会导致大量判断是否发生错误的冗余代码，开发中应该把处理错误逻辑相同的部分抽取出来。如果对是否发生错误不关心可以使用 `_` 来接收返回错误。（PS：`_` 在编译时会处理掉）

而第三方或内置库函数不会 `panic` 一个异常，让调用者 `recover`。

如果 `panic` 没有被 `recover`，那么有可能会挂起整个程序。
`panic` 应该在包内 `recover`，`panic` 只能由当前 goroutine `recover`。`panic` 目的就是为了解决：

> 遇到了在本包中无法处理的异常，一个跳出多个调用栈，再被外围的 recover 捕获，该包再返回调用者 error

# 语法

`recover` 只能够在 `defer` 中使用，`func recover() interface{}` 返回的是 `func panic(v interface{})` 的参数 **v**。如果没有 `panic` 但是调用了 `recover`，`recover` 返回的是 **nil**，多次调用 `recover` 不会有副作用。

`defer` 是一个先出后出的调用栈，参数的赋值在 `defer` 语句处已经完成，无论该函数是正常的执行，或返回 `error`，或 `panic`，`defer` 中的函数都能够保证得到执行。常用于 `recover` 捕获 `panic`、关闭文件资源、释放锁等等。

```golang
func main() {

}

func f(i int) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered in f", r)
		}
    }()
    fmt.Println("Calling g.")
	g(i)
	fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
	    fmt.Println("Panicking")
	    panic(fmt.Sprintf("%v", i))
	}
	defer fmt.Println("Defer in g", i)
	fmt.Println("Printing in g", i)
	g(i + 1)
}

```

output:

```
main start
Calling g.
Printing in g 2
Printing in g 3
Panicking
Defer in g 3
Defer in g 2
Recovered in f 4
main end
```

# panic 只能在本 goroutine 处理

尝试一下在别的 goroutine 中处理 panic，改造一下上面的 demo，`defer recover` 放到 main，然后使用 goroutine 执行 `f(i)` 函数。

```golang
func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered in f", r)
		}
	}()
	fmt.Println("main start")
    go f(2)
    // 当前 goroutine 睡眠，等待 panic
	time.Sleep(time.Second)
	fmt.Println("main end")
}

func f(i int) {
	fmt.Println("Calling g.")
	g(i)
	fmt.Println("Returned normally from g.")
}

func g(i int) {
	if i > 3 {
		fmt.Println("Panicking")
		panic(fmt.Sprintf("%v", i))
	}
	defer fmt.Println("Defer in g", i)
	fmt.Println("Printing in g", i)
	g(i + 1)
}
```

output:

```
main start
Calling g.
Printing in g 2
Printing in g 3
Panicking
Defer in g 3
Defer in g 2
panic: 4

goroutine 5 [running]:
exit status 2
```

panic 并没有被解决，一直抛到最上层，使整个程序崩溃，`fmt.Println("main end")` 并没有被执行，panic 只能由**当前** goroutine 解决。

`func g(i int)` 函数使用了递归调用本身，也就是 `panic` 发生在 `defer` 之前，但是 `defer fmt.Println("Defer in g", i)` 还是成功执行了。

# recover 只能在 defer 中有效

尝试把 `recover` 移除 `defer`，看看发生什么

```golang
func main() {
	fmt.Println("main start")
	f(2)
	fmt.Println("main end")
}

func f(i int) {
	fmt.Println("Calling g.")
	g(i)
	fmt.Println("Returned normally from g.")
	r := recover()
	fmt.Println("Recovered in f", r)
}

func g(i int) {
	if i > 3 {
		fmt.Println("Panicking")
		panic(fmt.Sprintf("%v", i))
	}
	defer fmt.Println("Defer in g", i)
	fmt.Println("Printing in g", i)
	g(i + 1)
}
```

output:

```
main start
Calling g.
Printing in g 2
Printing in g 3
Panicking
Defer in g 3
Defer in g 2
panic: 4

goroutine 1 [running]:
exit status 2
```

**Ooops**

# (err != nil) Always?

函数返回 error 中有一个坑，看以下代码，我们自定义了一个 `MyError` 实现了错误 `error` 接口：

```golang
func main() {
	if err := try(); err != nil {
		fmt.Println("err != nil")
	}
}

type MyError struct {}

func (e *MyError) Error() string {
	return "Oooops"
}

func try() error {
	var err *MyError = nil
	if false {
		err = new(MyError)
	}
	return err
}
```

无论运行多少次，运行结果都是 `err != nil`。

原因：

`error` 是一个接口类型，其中接口类型的存储有：`data` 和 `type`，函数返回会执行：

```golang
var err *MyError = nil
var returnError error = err
```

经过这个赋值后，`returnError` 中存储的 `data==nil`，但是 `type==*MyError`，`returnError` 已不再是 **nil**。

那该如何办？

1. 如果没有特殊要求，就直接使用内置函数 `errors.New()` 创建一个错误，其在 `errors/errors.go` 中的剪短源码如下：

```golang
package errors

// New returns an error that formats as the given text.
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

2. 采用自定义错误，再对函数做一层封装，如下：

```golang
type MyError struct {}

func (e *MyError) Error() string {
	return "Ooops"
}

func main() {
	err := CallMe()
	if err != nil {
		fmt.Println(err.Error())
	}
}

// 大写 C 开头，被外部调用
func CallMe() error {
	err := deal()
	if err != nil {
		errors.New("Some msg")
	}
	return nil
}

// 小写 d 开头，只能被内部使用
func deal() (e *MyError) {
	if false {
		e = new(MyError)
		return
	}
	return
}
```

# 总结

+ panic 不应跨域 package
+ recover 只能在 defer 中起作用
+ panic 只能由当前 goroutine recover
+ 自定义异常不应该作为返回值返回给外围调用者
