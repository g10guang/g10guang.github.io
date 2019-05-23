---
layout: post
title:  "Golang 方法"
date:   2018-02-02 09:00:05 +0800
categories: golang
---

# method

method 不同于普通 fuction，method 有指定接受者的参数

```golang
// 定义结构体
type Vertex struct {
    X, Y int
}

// 定义结构体的方法
func (v Vertex) Abs() float64 {
    // 距离
    return math.Sqrt(v.X * v.X + v.Y * v.Y)
}

func main() {
    v := Vertex{3, 4}
    fmt.Println(v.Abs())
}
```

Go 中可以给每一个每一个类型声明方法，而不仅仅是 *struct* 类型

类型别名：`type MyFloat float64`，此后就可以使用 `MyFloat` 来代替 float64

在方法传递中，*struct* 发生的是值复制传递

```golang
type Vertex struct {
    X float64
    Y float64
}

// 不会改变外围调用者的X Y值
func (v Vertex) A(f float64) {
    v.X = v.X * f
    v.Y = v.Y * f
}

// 传递指针，外围和内层使用的是同一个地址，修改会影响外围调用者
func (v *Vertex) B(f float64) {
    v.X = v.X * f
    v.Y = v.Y * f
}

func main() {
    v := Vertex{3, 4}
    fmt.Println(&v)
    v.A(10)
    fmt.Println(v)
    v.B(10)
    fmt.Println(v)
}
```

output:

```
&{3 4}
{3 4}
{30 40}
```

Go 中函数传递都是值传递，进行了拷贝复制。Go 中采用引用传递的一个地方是闭包中。

```golang
for i := 0; i < 3 i++ {
    defer fmt.Println(i)
}
```

output:

```
2
1
0
```

```golang
for i := 0; i < 3; i++ {
    defer func() {
        fmt.Println(i)
    }()
}
```

output:

```
3
3
3
```

所有的匿名函数都是引用了外部 i，所以输出结果都是相同的

为了防止上述情况，我们应该把 i 当做函数参数传递

```golang
for i := 0; i < 3; i++ {
    defer func(i int){
        fmt.Println(i)
    }(i)
}
```

output:

```
2
1
0
```

函数使用*指针*传递参数的两个主要原因：
+ 需要修改调用者的值
+ 避免大数据结构的复制


# interface

Go 中使得 interface 声明接口和具体 struct 实现方法分离，只要某个 struct 实现了所有 interface 接口的方法，那么该 struct 就是 interface 类型

如果接口 `var i I` 被赋予了 nil，那么方法中依然可进行方法调用，并不会触发空指针异常等，但是方法中传递的值是 `nil`

```golang
interface {}
```

`var i interface{}` 任何类型的值都可以赋予给 `i`，空接口常用在处理类型未知的情况下，例如`fmt.Println()` 可以接收任意数量任意类型的参数 `func Print(a ...interface{})`

interface type assertion

```golang
var i I
i = &T{"hello world"}
t := i.(*T)
```

如果 i 中不是 T 类型，就发生 panic

```golang
var i I
i = &T{"hello world"}
t := i.(F)  // panic
```

```golang
var i I
i = &T{"hello world"}
t, ok := i.(F)  // no panic
fmt.Println(t, ok)
```

如果 type assertion 成立，那么 `ok == true`，t 为对应的 T 值；否则 `ok == false`，t 为 类型 F 的 zero value。

# Stringer 接口

Golang 中使用 Stringer 接口来打印格式化

```golang
type Stringer interface {
    String() string
}
```

只要实现了 `String() string` 方法，那么使用 `fmt.Sprintf("%v", v)` 就会自动调用 `String() string 方法`，输出格式化字符串。

# error 接口

```golang
type error interface {
    Error() string
}
```

自定义异常只需要实现方法 `Error() string`

# io.Reader

Go 中定义一个 Reader 接口

```golang
type Reader interface{
    func Read(b []byte) (n int, err error)
}
```

大多数标准库都实现了该接口，比如文件、网络、压缩、密码等等，我们可以通过该接口方便地访问数据

+ []byte slice 用于承装读取的数据
+ n 表示该次读取，成功读取了多少个字节
+ err 如果已经到了流末尾，`err = io.EOF`
