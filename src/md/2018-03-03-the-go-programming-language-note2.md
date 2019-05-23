---
layout: post
title:  "《Go程序设计语言》笔记二"
date:   2018-03-03 12:00:05 +0800
categories: golang
---

# 接口 interface

从概念上讲，一个接口类型的值（简称接口值）有两部分构成：具体类型和该类型的值。二者成为接口的动态类型和动态值。接口的零值就是动态类型和动态值都是 nil。

将一个变量赋予接口值，会发生变量值的复制。

接口值可以用 `==` `!=` 操作符来比较，如果两个接口值都是 nil 或者二者的动态类型完全一致且动态值相等（使用动态类型的 `==` 操作符来做比较），那么两个接口值相等。因为接口值是可以比较的，所以他们可以作为 map 的键，也可以作为 switch 语句的操作数。

> 注意：如果接口的动态类型是不可以比较的，而使用了 `==` `!=` 比较操作符，则会可能触发宕机

```go
var x interface{} = []int{}
fmt.Println(x == x) // 宕机
```

**含有空指针的非空接口**

```go
func handle(v interface{}) {
    if v == nil {
        fmt.Println("v is nil")
    } else {
        fmt.Println("v is not nil")
    }
}

func main() {
    var x *int = nil
    handle(x)
}
```

虽然 `x == nil`，但是调用 handle 方法时，实际上 v 接收到的是一个类型为 `*int`，值为 nil 的值。所以 `v == nil` 并不成立，因为它确实装载了一个变量。

## sort

sort 包提供了针对任意序列根据任意排序函数原地排序的功能。sort.Sort 函数（使用快速排序算法）对序列和其中的元素布局无任何要求，它使用 sort.Interface 接口来指定通用排序算法和每个具体的序列类型之间的协议。

对于排序我们需要知道的是需要排序的元素的长度、如何比较两个元素的大小以及如何交换两个元素。而 sort.Interface 接口就定义了三个对应的方法。

```go
package sort

type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

```go
type stringSlice []string

func (s stringSlice) Len() int {
    return len(s)
}

func (s stringSlice) Less(i, j int) bool {
    return s[i] < s[j]
}

func (s stringSlice) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

func main() {
    s := stringSlice{"hello", "world", "哈哈", "abc"}
    sort.Sort(s)
    fmt.Println(s)  // [abc hello world 哈哈]
}
```

+ `r.URL` 带有 query 参数
+ `r.URL.Path` 不带 query 参数

## 接口类型断言

判断该接口变量中存储的是否是某类型

假设有：

```go
var w io.Writer
w = os.Stdout
```

有三种用法：
1. `f := w.(*os.File)` 只有一个结果，如果 w 中的变量类型不是 `*os.File` 就会触发 panic
2. `f, ok := w.(*os.File)` 有两个结果，如果 w 中的变量类型不是 `*os.File` 不会触发 panic，但是 `ok == false`，f 中存储的是类型 `*os.File` 的零值，也就是 nil，常配合 if 使用

```go
if f, ok := w.(*os.File); ok {
    //...
}
```

3. 配合 switch 使用，tyoe-switch 语法

```go
switch w.(type) {
    case T1:
        //...
    case T2:
        //...
    case T3:
        //...
    default:
        //...
}
```

> type-switch 不能配合 fallthrough 使用

## 关于接口的建议

**以下是来自《Go程序设计语言》中关于接口的建议：**

+ 仅在两个或多个具体类型需要按统一的方式处理时才需要接口
+ 设计新类型时越小的接口越容易满足，一个不错的接口设计经验是*仅要你所需要的*

通过接口可以实现灵活的动态性，给予接口的方法可以提供良好的抽象，提高代码复用。
