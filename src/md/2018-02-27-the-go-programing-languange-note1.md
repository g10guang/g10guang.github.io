---
layout: post
title:  "《Go程序设计语言》笔记一"
date:   2018-03-2 12:00:05 +0800
categories: golang
---

# 标准输入输出

编程语言中的标准流如，**stdin** / **stdout** / **stderr** 是指向操作系统中的文件，以 Linux 为例，以下是 `os.Stdin` `os.Stdout` `os.Stderr` 的定义：

```go
var (
    Stdin  = NewFile(uintptr(syscall.Stdin), "/dev/stdin")
    Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")
    Stderr = NewFile(uintptr(syscall.Stderr), "/dev/stderr")
)
```

从控制台输入 `os.Stdin`，

```go
func readFromStdin() {
    reader := bufio.NewReader(os.Stdin)
    fmt.Println("enter text:")
    text, _ := reader.ReadString('\n')
    fmt.Println(text)
}
```

fmt 包中提供了强大的格式化功能，可以使用 `fmt.Sprintf` 方法格式化字符串，该方法类似 `fmt.Printf` 区别是前者是将格式化的字符串作为返回结果，后者直接将格式化后的字符串输出到控制台。如果需要将格式化的字符串输出到流（实现了 `io.Writer`接口）中，可以调用 `fmt.Fprintf`。

# 获取运行命令的参数

运行命令的参数保存在 `os.Args` []string 中

```go
func main(){
    args := os.Args
    fmt.Println(args)
}
```

假如我们使用 `go run main.go hello world` 命令来运行，得到不是 `["go run main.go", "hello", "world"]`，在我本机是：`["/tmp/go-build610217828/command-line-arguments/_obj/exe/readStdin", "hello", "world"]`。我们可以通过 `go run -x main.go hello world` 查看命令执行的过程。

# const

Go 中常量只能是 字符串、数值、布尔值，复杂类型如slice、map、数组、结构体、指针、接口、函数等都无法声明为常量。

# var

只有 chan、slice、map、函数、指针、接口变量 可以与 nil 做 `==` `!=` 比较，数组和结构体在声明时如果没有赋值，那么就会分配存储空间，然后将内容写为对应的默认值、

包级别的变量、常量在 **init** 函数开始前进行初始化，而 **init** 函数的执行在 **main** 函数之前。

只有变量才能够使用 `&` 操作符，也就是没有指针能够指向常量。

当两个指针变量指向同一个变量或者是两个指针都是零值 nil 时候，使用 `==` 比较得到 **true**

可以使用 **flag** 包来开发命令行工具，flag 为我们解析参数提供了很大的帮助

任何一个包、任何一个 .go 文件可以包含任意多个 `func init(){...}` 函数，在初始化包的时候将会按照 init 函数声明的顺序来执行 init 函数。

包初始化顺序：
1. 按照 **import** 顺序完成引入的包的初始化
2. 根据编译器导入 .go 文件的顺序进行包级别变量的初始化
3. 按照 **init** 函数的声明顺序，执行所有 init 函数

pointer / slice / map / function / channel 都是引用类型，共同特点是全部都间接地指向程序变量或者状态，于是操作所引用数据的效果就会遍历该数据的全部引用。

# 类型

在不同编程语言中，去模 mod 运算 **%** 有不同表现，在 Go 中，取模余数的正负号总是与*被除数*一致，`-5 % 3 == -2` `-5 % -3 == -2`。题外话，Python 中去模余数总是和*除数*的正负号一致，`-5 % 3 == 1` `-5 % -3 == -2`

无论是有符号数还是无符号数，若表示的运算结果所需要的位超过了该类型的范围，就会产生**溢出**。

对于 uint8

```go
var u uint = 255
fmt.Println(u, u+1, u*u)    // 255 0 1
```

其中 `255 * 255` 结果的二进制形式为 `1111111000000001`，所以采取截断，最后结果为 1； 
`u+1` 的结果的类型依然是 `uint8`

类似地，对于 int8

```go
var i int8 = 127
fmt.Println(u, u+1, u*u)    // 127 -128 1
```

## ^ &^

相对与 C 语言的不同，在 Go 中 `^` 运算符既可以作为一元运算符，也可以作为二元运算符：

```go
var u uint8 = 255
fmt.Println(^u)     // 0
fmt.Println(u^1)    //254
```

+ `^` 作为一元运算符时，是按位取反，相当于 C 中的 `~`
+ `^` 作为二元运算符时，是按位异或

Go 中还有一个 C 中没有的运算符 `&^` 按位清空。

```go
var u uint8 = 11
fmt.Println(u&^1)   // 254
```

`x&^y` 的位运算中，将 y 中对应位为 1 的位置，将 x 中对应位置置为 0，否则保持 x 中对应位置不变，对于 uint8 `11 &^ 3 == 8`

## fmt.Printf

```go
fmt.Printf("%d %[1]o %[1]x %[1]b\n", 100)
```

其中 `[1]` 表示使用第一个参数

## 字符串

```go
s := "hello, 世界"
b := []byte(s)
s = string(b)
```

上述字符串转化为 []byte，[]byte 转化为 string 时候都会发生重新分配内存空间，然后再进行内存的复制。因为 string 底层是不可以改变的，如果底层数组进行复用，则会造成改变 []byte 的值会间接改变 string 的内容，进行很多这样的操作会使得程序执行非常低效。但是**某些编译器**能够识别到如果后续 []byte 不会再改变的话，string 会复用 []byte 底层的数组。

若要进行很多关于 string 的操作，尽可能使用 `bytes.Buffer`，其底层使用 []byte 进行数据的存储，动态扩展，效率相对高。

## 标准输入

类似于 C，Go 提供 `fmt.Scanf` 函数帮助我们从命令行中输入信息

## 常量

```go
const (
    a = 1
    b
    c = 2
    d
)
// a == b == 1
// c == d == 2
```

常量只能指向基本类型或者是经过 `type Time int` 等经过重命名的基本类型，但是许多常量并不从属某一具体类型。编译器将这些从属类型待定的常量表示为某些值，可以认为它们的精度至少达到 256 位。

## 数组

声明数组时，使用 `...` 表示让编译器计算数组长度，不需要我们显式声明，如：

```go
r := [...]int{1, 2, 3, 4}
s := []int{1, 2, 3, 4}
fmt.Printf("%T\n", r)   // [4]int 数组
fmt.Printf("%T\n", s)   // []int slice
```

同类型的数组是可以通过 `==` 或 `!=` 进行比较的，这里需要注意的是 `[3]int [4]int` 不是同类型。

在函数的参数传递中，数组发生的是值传递，也就是每一次传递都在内存分配一样大小的空间，再进行数组内容的复制，在函数内改变数组元素不会影响到原数组。如果传递大数组将会十分低效。所以需要进行函数传递的建议使用 slice，slice 是一个结构体，不会发生底层数组的复制，只是将复制指针值；或者使用数组的指针进行数组的传递。

Go 内置函数很多都是直接对 slice 进行传递和操作，感觉 Go 还是建议人们尽可能使用 slice 而不是数组。

## slice

声明数组或 slice 的时候，可以指定特定位置元素的值，如果之前的值没有显示声明，就会被声明为默认值。

```go
a := [...]int{99:1} // 声明一个 [100]int 数组，0~98 为 int 默认值 0，99 位元素为 1
s := []int{99:1} // 声明一个 []int slice，0~98 为 int 默认值 0,99 位元素为 1
```

```go
days := []string{1:"Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"}
p := days[1:3]
t := p[0:5]
fmt.Println(p, len(p), cap(p))
fmt.Println(t, len(t), cap(t))
```

注意：`len(p) == 2` 为什么可以取 `t := p[0:5]` 长度为 5 的 slice 呢？

因为 `cap(p) == 7`，p 底层数组复用了 days，也就是说只要不超越 **cap** 限制，就可以取任意长度的切片。

slice 取切片都使用指针复用了底层的数组，所以 slice 的取切片操作是一个高效的操作，并不会占用很多资源，问题是 slice 底层复用数组，有可能一个改变会影响其他的运行结果，尤其是在 goroutine 众多的情况下。

如果库支持的函数中只支持对应类型的 slice，而不支持数组类型参数怎么办呢？
比如：`func test([]byte)` 函数只支持 []byte，而不支持 [N]byte

可以通过对数组取切片，得到的 slice 复用了底层的数组存储空间。
如：

```go
a := [3]byte{1, 2, 3}
test(a[:])
```

一个等于 nil 的 slice 与`len==cap==0` 的 slice 有的唯一区别就是，与 nil 进行 `==` `!=` 比较时的表现相反。

**下面将会变得非常绕**

[stackoverflow 上一个关于 copy make 等内置函数的讨论](https://stackoverflow.com/questions/18512781/built-in-source-code-location)

一个例子：

```go
s := []int{1, 2, 3, 4}
copy(s[1:], s[0:2])
fmt.Println(s)  // [1 1 2 4]
```

这里得出的结论是 copy 函数应该是从后往前复制的，如果是从前往后复制的话得到的结果是 `[1 1 1 4]`

另一个例子：

```go
s := []int{1, 2, 3, 4}
copy(s[0:], s[1:])
fmt.Println(s)  // [2 3 4 4]
```

这里得出的结论是 copy 函数应该是从前往后复制的，如果是从后往前复制的话得到的结果是 `[4 4 4 4]`

这让我产生疑问，`copy` 函数的复制算法到第是怎么样的呢？

**copy** 函数的实现

```go
func slicecopy(to, fm slice, width uintptr) int {
    if fm.len == 0 || to.len == 0 {
        return 0
    }

    n := fm.len
    if to.len < n {
        n = to.len
    }

    if width == 0 {
        return n
    }
    // 忽略中间某些关于 race 的细节
    size := uintptr(n) * width
    if size == 1 { // common case worth about 2x to do here
        // TODO: is this still worth it with new memmove impl?
        *(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
    } else {
        memmove(to.array, fm.array, size)
    }
    return n
}
```

`memmove` 的实现在 [https://github.com/golang/go/blob/master/src/runtime/memmove_amd64.s](https://github.com/golang/go/blob/master/src/runtime/memmove_amd64.s) 由于全是汇编没有看懂。

如果需要实现 slice 的循环移动的话，我们可以通过三次翻手实现：

比如需要把 `[1 2 3 4 5]` 循环移动为 `[3 4 5 1 2]`

1. 把 `[1 2]` 旋转为 `[2 1]`
2. 把 `[3 4 5]` 旋转为 `[5 4 3]`
3. 此时 slice 已经变成 `[2 1 5 4 3]`，再进行一次翻手得到 `[3 4 5 1 2]`

## map

键 K 必须能够通过操作符 `==` 进行比较，选 K 的时候尽可能不适用浮点型，虽然浮点型可以通过 `==` 进行比较，但是比较存在不精确，通常我们比较字符串是通过一个阀值做到的，比如 `1 - 0.001 <= f <= 1 + 0.001`。

`delete` 函数从 map 中删除指定的 K-V，即使 K 不存在于 map 中也不会发生异常，如果从 map 中取出一个 K，如果该 K 不存在 map 中会返回一个类型 K 的零值，所以通常取元素操作会附带一个 comma-ok如：`v, ok := m[k]`。

map 元素不是一个变量，无法获取它们的地址，如下操作是不能够通过编译的：

```go
fmt.Printf("%p", &m["hello"])
```

原因是 map 是有可能动态增长的，当发生动态增长的时候，K-V 所在的地址发生了迁移，通过获取 `&m["hello"]` 没有意义，地址无效。

可以通过 `len` 函数获知 map 中 K-V 的数量。

map 的零值是 nil，向 nil map 中查找元素、`len(m)`、`delete(m, k)`、for range 循环，都不会发生错误，其行为像对已经初始化但是依然是空的 map 操作一样，但是如果向 nil map 中设置 K-V 将会导致错误。

## struct

```go
package p
type T struct {a, b int}

package q
import "p"
var a = p.T{a:1, b: 2}  // 编译错误
var b = p.T{1, 2}       // 编译错误
```

因为 a b 是以小写开头的，都是不可导出的，在别的包下无法显示 `p.T{a:1, b: 2}` 或隐式 `p.T{1, 2}` 引用。

如果一个结构体中所有的成员都是可以比较的，那么这个结构体就是可以比较的；否则该结构体不能够通过 `==` `!=` 进行比较，可以比较的结构体可以作为 map 的 K。

```go
// 可比较的结构体
type Point struct {
    X, Y int
}

// 不可比较的结构体
type array struct {
    S []int
}
```

结构体可以组合是 Go 实现面相对象的重要部分。

```go
type Point struct {
    X, Y int
}

type Circle struct {
    Point
    Radius int
}
```

这样 Circle 就能够调用 Point 的方法，Circle 中 Point 的名字就是 Point，也就是 Cicle 中不能够再有名为 Point 的成员了。

# 函数

许多编程语言都为线程分配一个固定长度的函数调用栈，大小在 64KB 到 2MB 之间，递归的深度受限于固定长度大小栈，Go 语言实现了可变长的栈，栈的使用会随着使用的增大而增大，可达到 1GB 左右的上限，这样我们可以安全地使用递归而不用担心栈溢出。当然在栈编程的时候像是 slice、map 的扩展，会出现内存的复制，频繁增长会使效率降低。

Go 垃圾回收机制可回收未使用的内存，但不能指望它会释放未使用的操作系统资源，比如打开文件以及网络连接等，**必须显示关闭操作系统资源**。

可变参数传递的是一个对应类型的 slice，如：

```go
func sum(vals ...int)
```

vals 是一个 `[]int` 的 slice

## defer

使用 defer 语句正确的地方是在成功获得资源后。

值得一提是先生成 **return** 返回的值，然后再以先入后出的顺序执行 **defer** 语句。

```go
func main() {
    a := test()
    fmt.Println(a)
}

func test() int {
    a := 0
    defer func() {
        fmt.Println(a)
    }()
    defer func() {
        a++
    }()
    return a
}
```

输出：

```
1
0
```

但是如果返回值有命名，那么 defer 修改命名返回值可以修改外围调用者接收到的返回值，如：

```go
func main() {
    a := test2(1)
    fmt.Println(a)
}

func test2(x int) (result int) {
    defer func() {result += x}()
    return x + x
}
```

输出： `3`

## 闭包

Go 中闭包的实现是通过 **逃逸分析** 做到的，通过分析变量是否有逃逸的可能，如果有则把变量创建在堆中，否则变量分配在当前 goroutine 的栈中。

变量在同一个包下都是可见的，无论该属性是否是可导出的*exported*

Go 语法糖（由编译器实现的便利功能）之一是：对于方法的接收者是**值**或者**指针**，Go 编译器会帮助我们实现**取值`*`**和**取地址`&`**操作。如：

```go
type S struct {
    s string
}

func (s *S) hello() {
    fmt.Println("hello,", s)
}

func (s S) world() {
    fmt.Println("world wide web", s)
}

func main() {
    s := S{s: "world"}
    s.hello()  // OK
    p := &s
    p.world()  // OK
}
```

但是自动 `*` 和 `&` 操作只能针对变量有效，对于类型的字面量不起作用，如：

```go
S{s: "world"}.hello()   // 编译错误
```

## nil 是合法的方法接收者

因为很多类型中 nil 是有意义的，比如 slice、map、chan，只要将 nil 转化为其他可以为 nil 的类型，那么 nil 也可以作为方法的接收者。比如一个链表的例子：

```go
type IntList struct {
    Val int
    Next *IntList
}

func (node *IntList) Sum() int {
    if node == nil {
        return 0
    }
    return node.Val + node.Next.Sum()
}
```

如果不是环形链表的话，那么最后一个结点的 `node.Next == nil`，使用 `node.Next.Sum()` 可以正常运行。对于方法的接收者可以是 nil 的情况下要多加小心，比如修改 nil map 可能会触发宕机 panic。

## 方法函数变量

```go
type S struct {
    s string
}

func (s *S) hello() {
    fmt.Println("hello,", s)
}

func (s S) world() {
    fmt.Println("world wide web", s)
}

func test1() {
    s := S{s: "world"}
    f := s.hello
    s.s = "hi"
    fmt.Printf("%T\n", f)
    f()
}

func test2() {
    p := &S{s: "world"}
    f := p.world
    p.s = "ja"
    f()
}
```

test1 函数输出：

```
func()
hello, &{hi}
```

也就是修改了 `s.s` 的值后，`f()` 方法的调用收到了影响。

test2 函数输出：

```
world wide web {world}
```

也就是修改了 `p.s` 的值后，`f()` 方法的调用不受影响。

分析：因为 test1 中 f 方法保存的是 (s *S) 指针，他们始终引用同一个地址，所以修改了引用的地址的值，`f()` 也会受到影响。test2 中 f 方法保存的是 (s S) 值，与 p 指针指向的地址不一样，因为 s 已经经过了值复制，修改 p 并不会影响到 s，所以 `f()` 调用不受影响。

> 如果使用方法函数变量，请注意如果类型中对应有引用，可能会发生难以排查的错误

## 集和的实现

Go 并没有 build-in 集和 set（Python 有，Java 也有），而 map 与 set 非常相似，只是 map 存储的是 K-V，set 存储的是 K，可以通过把 V 插入一个 bool 值来达到把 map 改装为 set 的功能。

PS：Java 中的 HashSet 就是把 HashMap 中的 V 架空实现的，其他算法和 HashpMap 几乎一样。

```go
type set map[T]bool
```
