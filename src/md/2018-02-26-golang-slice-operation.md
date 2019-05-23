---
layout: post
title:  "Golang slice操作"
date:   2018-02-26 12:00:05 +0800
categories: [golang, slice]
---

## 创建

```golang
// 1.只声明不赋值
var s []T

//2. 创建 nil slice
s := []T(nil)   // 也就是将 nil 转化为 slice，slice 和 nil 是可以做 == or != 比较的

//3.直接创建类型为 T 的 slice
s := []int{1, 2, 3} // [1 2 3]
```

**可以向 nil slice 进行 append 操作**

```go
var s []string
s = append(s, "hello", "world")     // 会触发 slice 的扩展
```

## 判断 slice 是否已经 make

```golang
var s []string
fmt.Println(s == nil)   // true
s = make([]string, 0)
fmt.Println(s == nil)   // false
```

## append

通过内置函数 `append` 向 slice 中追加元素

函数声明：

```golang
func append(slice []Type, elems ...Type) []Type
```

该函数可以接收多个参数，并且把参数依次添加到 slice。

在向 slice 中不断添加元素肯定会触发 slice 底层数组的扩容，那么 slice 的扩容

## copy

`copy` 将 src 中的内容复制到 dst 的头部中，复制元素长度规则：

```go
if src == nil || dst == nil {
    // do nothing
    return
}

if len(src) > len(dst) {
    // copy src[0:len(dst)] to dst
} else {
    // copy src to dst[:len(src)]
}
```

## cut

将 slice 的某部分删除

```go
s := make([]string, 10)
for i := 0; i < 10; i++ {
    s[i] = string(i + '0')
}
// 此时 slice  中的内容是 [0 1 2 3 4 5 6 7 8 9]
// 剪去下标 [5,7)
s = append(s[0:5], s[7:]...)
```

## delete

Go 中有提供 map 的 delete 函数，但是却没有提供对于 slice 中删除某个函数的操作，这样如果需要删除函数，所以我们需要自定义一个删除函数。

想想：slice 的 delete 和 cut 供能是不是很相像，delete 是删除一个元素，而 cut 是删除某个范围。

```go
s := make([]string, 10)
for i := 0; i < 10; i++ {
    s[i] = string(i + '0')
}
// 此时 slice  中的内容是 [0 1 2 3 4 5 6 7 8 9]
// 删除下标为 5 的元素
s = append(s[0:5], s[6:]...)
```

## pop

Python 的 list 提供了 `pop()` 方法移除末尾元素，那么 Go 中应该如何实现呢？

```go
s := make([]string, 10)
for i := 0; i < 10; i++ {
    s[i] = string(i + '0')
}
// 此时 slice  中的内容是 [0 1 2 3 4 5 6 7 8 9]
s = s[0:len(s) - 1]     // 如果需要 pop 多个元素可以是当扩大 1 的值
```

### 问题

如果 slice 中的内容是指针或者带有指针项的结构体，slice 中的底层数组始终引用着堆中的值，这样导致了结果就是需要被垃圾回收的数据没有被及时回收，造成内存空间资源的浪费。

值得一提的是 Go 会对在编译时变量做逃逸分析，如果该变量有可能逃逸则在堆中分配内存空间，否则在当前 goroutine 的栈中分配内存空间。

**应该在 for-loop 中声明变量吗？**

Go 处理机制是会做编译器优化复用一个内存地址还是在内存中创建多个变量呢？

```go
func main() {
    for i := 0; i < 3; i ++ {
        var a int
        a = i
        fmt.Println(&a)
    }
}
```

```
0xc4200160e8
0xc420016120
0xc420016128
```

显然通过打印变量地址知道：每次都创建不同的变量。所以写代码的时候应该注意避免在 for-loop 中创建变量

所以在 cut 和 delete 操作中应该把指针重定向到 nil，防止内存泄露。

## cut

```go
s := make([]*int, 10)
for i := 0; i < 10; i++ {
    var a int
    s[i] = &a
}
// cut [5,7)
i, j := 5, 7
copy(s[i:], s[j:])  // 将后面元素往前移动
// 消除指针引用
for k := len(s) -  1; k >= len(s) - j + i; k-- {
    s[k] = nil
}
s = s[:len(s) - j + i]
```

delete 类似

## expand

在特定位置扩容 slice，在下标 i 后插入 j 个元素

```go
s = append(a[:i], append(make([]T, j), a[i:]...)...)
```

但是根据 slice 的操作，该操作的内存利用率并不是很高，或者可以通过判断 `cap(s) > len(s) + j`，可以直接将 `a[i+1:]` 元素往后移动 j 位

## extend

扩展 slice 的容量，向末尾添加 j 个空位，扩展容量

```go
s = append(s, make([]T, j)...)
```

## insert

向特定下标插入特定值，比如向下标为 i 的位置插入元素 x

```go
s = append(s[:i], append([]T{x}, s[i:]...)...)
```

### 问题

上述对 slice 的 expand extend insert 等操作会创建一个新的 slice 然后再将新 slice 中的元素复制到原来的 slice 特定位置中，然后需要用额外的 GC 去回收新创建的临时 slice。这样的操作对 cpu 和内存来说都是低效的实现方法。

## 更高效的 insert

```go
s = append(s, zero_value)
copy(s[i+1:], s[i:])
s[i] = x
```

## 插入 vector

```go
a = append(a[:i], append(slice, a[i:]...)...)
```

## 向 slice 头部添加新元素

```go
s = append([]T{x}, s...)
```

## 倒转 slice

```go
func reverse() {
    s := make([]int, 10)
    for i := 0; i < 10; i++ {
        s[i] = i
    }
    fmt.Println(s)
    for i := 0; i < len(s) / 2; i++ {
        minor := len(s) - i - 1
        s[i], s[minor] = s[minor], s[i]
    }
    fmt.Println(s)
}
```

## 洗牌

```go
func shuffling() {
    s := make([]int, 10)
    for i := 0; i < 10; i++ {
        s[i] = i
    }
    fmt.Println(s)
    // 通过随机交换两个元素达到洗牌目的
    for i := len(s) - 1; i > 0; i-- {
        j := rand.Intn(len(s))
        s[j], s[i] = s[i], s[j]
    }
    fmt.Println(s)
}
```

### 部分翻译自

[slice Trick](https://github.com/golang/go/wiki/SliceTricks#delete)
