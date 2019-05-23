---
layout: post
title:  "Golang 反射"
date:   2018-03-06 09:00:05 +0800
categories: golang
---

# 反射

Go 语言提供一种机制，在编译时不知道类型的情况下，可更新变量、在运行时查看值、调用方法以及直接对它们的布局进行操作，这种机制成为反射。通过反射我们甚至可以访问结构体中的非导出成员，但是不能够修改它们。

**为什么使用反射？**

原因：有时需要写一个函数有能力统一处理各种值类型的函数，而这些类型可能无法共享同一个接口，也可能内存布局未知，也有可能这个类型在我们设计函数时还不存在，甚至这个类型会同时存在上面的三种问题。

**注意事项**

1. 基于反射的代码非常脆弱，如果使用明确的类型进行操作，在编译期间编译器能够识别出其中的错误，而使用反射错误只有在运行时才会发现
2. 反射降低代码可读性
3. 使用反射比使用特定类型优化的函数慢一两个数量级

> 尽可能避免反射

曾阅读[今日头条中关于 Golang 优化](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2650996069&idx=1&sn=63e7f5d5f91f9d84f1c3278426f6edf6&chksm=bdbf05368ac88c20c273f325acc257811d6ee7534df30ace674f5c7eeb0c7986984dca209131&scene=21#wechat_redirect)的一篇文章，说尽可能不要用反射

go 中 interface 实现了 duck-typing，那么一个问题就是通过 `interface` 变量来重新寻找原始的值

+ 编译时：静态
+ 运行时：动态

[Data structure: interface](https://research.swtch.com/interfaces)

介绍了 interface 变量中既存储了相应的 `type` 和 `value`

反射三大法则：
+ Reflection goes from interface value to reflection object.
+ Reflection goes from reflection object to interface value.
+ To modify the object, the value must be settable.

```golang
var i Interface
i = obj
```

上述发生的是 copy 复制，也就是说 interface 变量底层真实存储的是一个跟 `obj` 完全不同的变量，所以 interface 中的 `data` 与 `obj` 是独立变化的

reflect 包下定义了两个重要的类型：Type 和 Value。Type 表示 Go 语言的一个类型，它是一个有很多方法的接口，这些方法可以用来识别类型以及透视类型的组成部分，比如一个结构的各个字段或者一个函数的各个参数。

应该尽可能避免在包的 API 里面暴露反射相关内容。

```golang
var x float64 = 3.4
fmt.Println(reflect.TypeOf(x))      // 获取 x 类型
fmt.Println(reflect.ValueOf(x))     // 获取 x 值
fmt.Println(reflect.ValueOf(x).Type())  // 我们可以从类型中获得值
fmt.Println(reflect.ValueOf(x).Kind())  // 获取存储对应的基本类型，如 reflect.Float64 reflect.Slice reflect.Struct
```

```golang
var x float64 = 3.4
v := reflect.ValueOf(x)
switch v.Kind() {
    case reflect.Int:
        fmt.Println(v.Int(()))
    case reflect.Float64:
        fmt.Println(v.Float())
}
```

对应 reflect 中对于类型的操作 `setter getter`，总是采用最大的类型，比如 uint64 int64 float64 等

```golang
var x uint8 = 'x'
v := reflct.ValueOf(x)
fmt.Printf("%T", v.Uint())  // uint64
x = uint8(v.Uint())         // 需要把类型转换到需要的类型
```

`Type()` 显示的是静态类型

`Kind()` 显示的是底层的类型，而不是静态类型

```golang
type MyInt int
x := MyInt(100)
v := reflect.ValueOf(x)
fmt.Println(x.Kind())   // int，而不是 MyInt
fmt.Println(x.Type())   // main.MyInt，而不是 int
```

`interface <==> reflection` 这两者之间是可以转化的

```golang
var x float64 = 1.0
v := reflect.ValueOf(x)
y := v.Interface()
fmt.Println(y)      // 1.0
// 提取处 float64
f := y.(float64)
```

不是所有 reflect.Value 都可以修改的

```golang
var x float64 = 1.0
v := reflect.ValueOf(x)
v.SetFloat(2.5)
```

会被抛出：

```
panic: reflect: reflect.Value.SetFloat using unaddressable value
```

原因是 v 不能够被寻址，从而不能够被修改。

**什么样的变量在反射后能够被寻址？**

简单地，可以使用 `Value.CanAddr()` 判断。

像是一个整数 123、浮点数 3.14，就是典型的不可寻址的，为什么？因为只有变量才能够被寻址。但为什么上例中 x 是变量但是错误显示 x 无法寻址呢？因为 `reflect.ValueOf(v interface{})` 参数接收的是一个 interface，而我们知道 interface 中的值存储动态类型以及动态值，在参数传递中，会进行参数的复制，也就是动态值只是一个 copy，改一个 copy 的值有意义吗？显然没有意义。

那到底什么才能够在反射后能够被寻址啊！！！

当传递给 `reflect.ValueOf(pointer)` 函数是一个指针的时候可以被寻址。因为此时 interface 里面存储的是指针，可以通过指针修改实参的值。

```go
x := 2
a := reflect.ValueOf(x)
b := reflect.ValueOf(2)
c := reflect.ValueOf(&x)
d := c.Elem()               // 取得 &x 对应的值，相当于 *(&x)==x
fmt.Println(a.CanAddr())    // false
fmt.Println(b.CanAddr())    // false
fmt.Println(c.CanAddr())    // false
fmt.Println(d.CanAddr())    // true
```

`c.Elem()` 就是取相同 `&x` 指向内存的变量，也就是 x 的值。

```go
d.Set(reflect.ValueOf(1))
fmt.Println(x)  // 1
```

**什么样的变量在反射后能够被修改？**

1. 首先该变量应该是能够被寻址的，连寻址都不能够做到，也就是连变量到底存储的位置都不知道，谈何修改。
2. 在结构体中，该成员是导出的。因为执行修改的函数不在该变量声明的包内，别的包是不能访问另一个包的非导出成员的。

```go
var x struct{
    a int
    A int
    }
a := reflect.ValueOf(&x)
b := a.Elem()
b.FieldByName("A").Set(reflect.ValueOf(10))
fmt.Println(x)  // {0 10}
```

如果以上代码改为

```go
b.FieldByName("a").Set(reflect.ValueOf(10))
```

会引发宕机：`panic: reflect: reflect.Value.Set using value obtained using unexported field`。最恐怖的是这是运行时的宕机，而不是编译失败，如果真实项目中运行出现 panic 宕机真实灾难。

通过 `CanSet` 方法查看某个 reflection object 是否可以修改

```golang
var x float64 = 1.0
v := reflect.ValueOf(x)
fmt.Println(v.CanSet())     // false
```

在函数传递参数时，go 是进行值传递，完全 coppy 了一个新的值，所以即使我们在方法中修改了值，外围调用函数也不会收到影响。
所以我们 `reflect.ValueOf()` 传递的是 copy，也就不能够 set。

所以应该传递指针

```golang
var x float64 = 1.0
v := reflect.ValueOf(&x)
fmt.Println(v.CanSet())     // false
p := v.Elem()
fmt.Println(p.CanSet())     // true
p.SetFloat(2.5)
fmt.Println(x)              // 2.5
fmt.Println(*(v.Interface().(*float64)))    // 2.5
```

通过反射修改 struct 的值

```golang
type Human struct {
    name string
    age  int
}


h := Human{"hello", 100}
v := reflect.ValueOf(&h).Elem()
typeOf := v.Type()
for i := 0; i < v.NumField(); i++ {
    f := v.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i, typeOf.Field(i).Name, f.Type(), f)
}
```

output:

```
0: name string = hello
1: age int = 100
```

struct 中只有大写开头的属性才能够被修改：

```golang
type Human struct {
    Name string
    age int
}

h := Human{"hello", 100}
v := reflect.ValueOf(&h).Elem()
v.Field(0).SetString("world")
typeOf := v.Type()
for i := 0; i < v.NumField(); i++ {
    f := v.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i, typeOf.Field(i).Name, f.Type(), f)
}
fmt.Println(h)
```

output:

```
0: Name string = world
1: age int = 100
{world 100}
```

如果 Human 中的名字是小写 name，那么就会引发错误：`panic: reflect: reflect.Value.SetString using value obtained using unexported field`

除了直接使用 == 进行比较两个值之外，Go 为我们提供了一个强大的函数：

```go
func reflect.DeepEqual(x, y interface{}) bool
```
