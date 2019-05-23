---
layout: post
title:  "Golang 变量内存模型"
date:   2018-02-17 12:00:05 +0800
categories: golang
---

# Golang 变量在内存的形式

int uint 在不同系统不同编译器有不同表现，`gc` `gccgo` 的实现是在 64 位系统下，int uint 为 64 位，而 32 位系统为 32 位。

类似的，指针长度在 64 位系统为 8 字节，32 位系统为 4 字节。

数组、结构体中数据在内存中的紧密相连的。

# 字符串

```golang
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

字符串使用 16 字节长的数据结构表示，包含一个指向字符串存储数据的指针和一个长度数据。采用字符串切片生成新的字符串的时候不会涉及到内存的分配和复制操作，因为多个字符串重用了底层的存储数据，因为字符串是不可变的（改变字符串会生成新的字符串），不会有内存共享问题。

Go 使用 utf-8 编码字符串，（utf-8编码作者是 Go 作者之一），Go 的字符串每一个字符是 rune，rune 是 uint32 的别名，unicode 字符的长度可能是1,2,3,4个字节。如果统计字数算的是 rune。

```golang
s := "刘曦光"
len(s)  // 9，字节数
len([]rune(s))  // 3，rune 数
```

使用下标访问字符串，得到的不是第 n 个字符，而是底层存储的第 n 个 byte。

```golang
s := "刘曦光"
s[0]    // 229
```

一个一直没弄清楚的问题：**Unicode UTF-8 string 之间的关系**

上古时代的程序员可能会出现字符集、编码方式等问题，但是现在我们开发中编码方式等问题一般都有编辑器或者IDE提供好了完美的支持。

通过阅读以下两篇文章我弄清楚了二者的关系：

+ [字符编码笔记：ASCII，Unicode 和 UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
+ [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)

简而言之：

+ Unicode 是 charset 字符集，*for see* 对应的是码位/码点/code point
+ UTF-8 是 encoding 编码方式，*for store* 存储在存储设备上，内存外存

Unicode 字符集可以表示所有的字符，但是其有不同的实现方式，比如 UTF-18，UTF-16 等等。没错 UTF-8 只是 Unicode 的一种实现方式。UTF-8 采用特殊的编码方式，使用频率高的字符对应的存储字节数越短。

*错误例子*
之前学习 Java 的过程中阅读过某些**错误**的资料说 Java 使用 Unicode 编码，每一个字符采用两个字节存储，可以表达所有的字符。

事实上，两个字节 16 位一共可表达的字符数量为 `2 ** 32 = 65536`，根本不足以表达所有字符，且 Unicode 只是字符集而不是编码方式。Unicode 编码数量没有实际上限，事实上他们拥有远超 65536 个，所以不是所有 Unicode 编码都能够被压缩到 2 字节。

即使 Times New Roman 等使用了不同样式显示 A，但是 A 还是同一个字符。只是使用了不同字体样式显示，存储还是使用了相同的编码方式。

在不同的系统中可能会有大端存储 big-endian 或者小端存储 small-endian，更多关于该方面可以阅读阮一峰的一篇文章：[理解字节序](http://www.ruanyifeng.com/blog/2016/11/byte-order.html)

In Go a string is in effect a **read-only slice of bytes**.

```golang
unsafe.Sizeof(variable)
```

十六进制代表字符串

```golang
s := "\x68\x65\x6c\x6c\x6f" // "hello"
```

数字中使用 0xFF 代表十六进制，0111 代表八进制

```golang
fmt.Printf("%q", string(100))
```

输出的是 "d"，而不是 "100"

Go 源码 source code 只允许使用 UTF-8 编码

使用 raw string 里面不对 `\n` `\xff` 等进行转义

```golang
s := `hello
    world`
```

Go 中的 string 只能够包含 UTF-8 编码吗？

不是的，string 还能够通过 "\xff" 等形式控制每一个 byte。

**for range** 遍历字符串中的每一个字符，而不是字节

```golang
s := "hello world"
for _, v := range s {
    // v 的类型是 int32，也就是 rune    
    fmt.Println("%v ", v)
}
fmt.Println()
for _, v := range s {
    fmt.Printf("%v ", string(v))
}
```

output:

```
104	101	108	108	111	32	119	111	114	108	100
h	e	l	l	o	 	w	o	r	l	d
```

官方库 **unicode/utf8** 中有很多 UTF-8 方面的支持

字符串引用同一个源字符串的坏处：对于一个很大的源字符串，即使只有一小部分还被引用，源字符串就无法被回收。

字符串引用同一个源字符串的好处：字符串的切割、复制操作非常昂贵，需要分为分配-复制两步。

[Golang 官方对 strings 说明](https://blog.golang.org/strings)

# slice

```golang
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

数组、slice 并不会真正复制一份数据，而是复用了底层的数组存储

即使是 slice 的赋值，底层的数组都是使用同一个，其中一个的变化会引发另外一个的同步变化

```golang
func main() {
    x := []int{1, 2, 3, 4, 5}
    y := x
    y[0] = 10
    fmt.Println("x:", x)
    fmt.Println("y:", y)
}
```

output:

```
x: [10 2 3 4 5]
y: [10 2 3 4 5]
```

## 扩容

在对 slice 进行 append 等操作可能会触发 slice 的扩容

扩容规则：

+ 如果当前 cap < 1024，按每次 2 倍增长，否则每次按当前 cap 的 1/4 增长

## 创建 slice

可以通过 **new** 或 **make** 创建 slice，new 返回的是一个已经清零的指针，而 make 返回的是一个复杂的结构。

创建 slice 最好使用 make 创建

更多请参考：[深入解析 Go 中 Slice 底层实现](https://halfrost.com/go_slice/)

# map

```golang
type hmap struct {
    count     int // # live cells == size of map.  Must be first (used by len() builtin)
    flags     uint8
    B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

    extra *mapextra // optional fields
}
```

```golang
func main() {
    m1 := make(map[int]int)
    m2 := m1
    for i := 0; i < 1000; i++ {
        m1[i] = i
    }
    for k, _ := range m1 {
        _, ok := m2[k]
        fmt.Println(ok)
    }
}
```

总是输出 **true**。

map 进行的复制并不会重新分配空间，而是复用了底层的存储存储结构，即使是 m1 插入了很多数据，已经触发了扩展，buckets 的重新分配，m1 和 m2 还是会同步变化的。

map 使用链表解决哈希冲突问题，而不是开放地址，因为开放地址法在真实扩容的时候性能下降得很快。链表的位置不需要重新计算哈希值，因为扩容是成倍增长。

map 的扩容采用了两个 bucket 的方法，不是一次性完成扩容操作，而不一次次地把 oldbuckets 中的元素移到 buckets 中，虽然这样不能够消除总扩展时间，但是扩展时间分摊到每一次插入，这样防止程序发生长时间的阻塞。

更多请参考：
+ [如何设计并实现一个线程安全的 Map ？(上篇)](https://halfrost.com/go_map_chapter_one/)
+ [如何设计并实现一个线程安全的 Map ？(下篇)](https://halfrost.com/go_map_chapter_two/)

# nil

空值 zero value：

| type | value |
| ---- | ------ 
| bool | false |
| int  | 0     |
| string | ""  |
| pointer | nil |
| func  | nil |
| interface | nil |
| slice | nil |
| channel | nil |
| map | nil |

读、写一个 nil 的 channel 会阻塞

