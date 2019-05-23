---
layout: post
title:  "utf8 in Golang"
date:   2018-04-01 20:00:05 +0800
categories: golang
---

# 为什么记录下这文章？

刷算法题的时候很多时候遇到字符串操作问题，我比较喜欢使用 Python 或者 Golang 进行算法的实现，但是比较坑的一点就是 Golang 的字符串操作比较麻烦。

怎么麻烦？我举几个例子：

```go
for _, x := range "hello" {
    // x 的类型是 rune，其实就是对应字符的 utf8 编码
}
```

```go
s = "刘曦光"
s[0] == uint8(229)
```

上面我标明是 uint8 的原因是 Go 是强类型语言，甚至没有像 C 一样的隐式转化。不同类型之间是不能够直接进行互操作的。比如 int 与 uint8 就无法比较大小。

```go
len("刘曦光") == 9
```

返回的是底层字符串字节所占用的长度，而不是字符串中对应 utf8 字符的个数。

如果我想取字符串中的第 k 个字符该咋搞呢？

如果字符串只包含 ASCII 符号（ASCII 所有字符编码长度为 1Byte），那么我们完全可以使用下标解决：

```go
s = "hello"
s[0] == byte('h')
s[1] == byte('e')
s[2] == byte('l')
s[3] == byte('l')
s[5] == byte('o')
```

但是如果遇到中文等 utf8 编码长度不止 1Byte 的呢？怎么办？

比较简单的解决方案：

```go
func find(s string, index int) (rune, error) {
    for i, v := range s {
        if i == index {
            return v, nil
        }
    }
    return rune(0), errors.New("Out of range.")
}
```

# unicode/utf8

求助 unicode/utf8 官方库的帮助，遗憾的是该库中也没有特别丰富的函数支持，但是总比我们自己写轮子好。

以下例子隐含：

```go
import (
    "unicode/utf8"
    "fmt"
)
```

## 遍历 utf8 字符串

```go
func main() {
    s := "今天愚人节"
    for i := 0; i < len(s); {
        r, size := utf8.DecodeRuneInString(s[i:])
        fmt.Print(string(r))
        i += size
    }
}
```

每次取字符串的一个切片，对字符串取切片的一个开销很小的操作，新的切片会引用原字符串。

## 逆序遍历 utf8

```go
func main() {
    s := "今天愚人节"
    for i := len(s); i > 0; {
        r, size := utf8.DecodeLastRuneInString(s[:i])
        fmt.Print(string(r))
        i -= size
    }
}
```

## 取 utf8 字符串长度

```go
func main() {
    s := "今天愚人节"
    fmt.Println(utf8.RuneCountInString(s))
}
```

## 取特定下标的 utf8 字符

```go
int main() {
    s := "今天愚人节"
    fmt.Printf("%q", string([]rune(s)[3]))   // "人"
}
```

这里我们先把字符串 s 转化为 []rune 类型，也就是相当于把字符串中的每一个 utf8 字符解码出来。如果字符串长度不是非常大的话，建议先将 string 转化为 []rune，但如果字符串很大的场景下，这个方法并不适用

**改良版**

```go
func stringIndex(s string, index int) (rune, error) {
    c := 0
    for i := 0; i < len(s); c++ {
        r, size := utf8.DecodeLastRuneInString(s[i:])
        if c == index {
            return r, nil
        }
        i += size
    }
    return 0, errors.New(fmt.Sprintf("index=%d len(s)=%d index out of range", index, c))
}
```

unicode/utf8 下的方法并不是很丰富，在刷算法题中可能用到的就是上面几个，上面对应的 string 方法都有一个面相 []byte 的版本。

# strings

每个语言都会有非常丰富的对应字符串的操作方法集和，Go 的字符串操作在 strings 包。

以下简单介绍以下 strings 的基本操作

## 比较两个字符串

最简单的方法是使用 `<  >  ==  >=  <=` 等操作符实现，非常灵活。

也可以使用 `strings.Compare(a, b string)` 方法实现，直接使用操作符返回的是 bool 值，而 `strings.Compare` 返回 -1 0 1 三者之一。想到什么没有？可以使用 switch 语句，减少大量的 if-else 语句。

```go
func main() {
    a, b := "hello", "world"
    switch strings.Compare(a, b) {
    case 0:
        fmt.Printf("%q==%q", a, b)
    case -1:
        fmt.Printf("%q<%q", a, b)
    case 1:
        fmt.Printf("%q>%q", a, b)
    }
}
```

## 判断是否字串

这个问题是不是与搜索字串及其下标很相似？

```go
strings.Contains(s, substr string) bool
```

判断字符串是否包含某个字符集和

```go
strings.ContainsAny(s, chars string) bool
```

没错，判断是否包含子串就是搜索子串下标，然后判断搜索下标是否为 -1。

## 判断两个字符串是否相等 case-insensitive

```go
fmt.Println(strings.EqualFold("hello你好", "HELLO你好"))    // true
fmt.Println(strings.EqualFold("hello", "world"))    // false
```

## 字符串切割

通过 unicode.IsSpace 判断是否是空白字符，然后通过空白字符进行切割

```go
fmt.Println(strings.Fields("hello\tworld 你好\n世界"))
// [hello world 你好 世界]
```

如果需要进行切割的不是空白字符，需要自定义方法怎么办？

```go
fmt.Println(strings.FieldsFunc("hello world", func(r rune) bool {
    return r == ' '
}))
// [hello world]
```

## 前后缀

判断是否包含前缀

```go
fmt.Println(strings.HasPrefix("hello world", "hello")) // true
```

判断是否包含后缀

```go
fmt.Println(strings.HasSuffix("hello world", "world"))  // true
```

## 子串所在位置

```go
fmt.Println(strings.Index("hello world", "world"))  // 6
```

上面例子是第一个出现的位置，下面这个例子我们求最后一次出现的位置

```go
fmt.Println(strings.LastIndex("hello world", "l"))  // 9
```

如果我们要求符合某个特别条件的字符的位置呢？

```go
fmt.Println(strings.IndexFunc("hello world", func(r rune) bool {
    return r + 1 == 'e'
}))
```

对应的也有 `strings.LastIndexFunc(s string, f func(r rune) bool)`

## 拼接字符串

如果我们有一个字符串slice []string，将他们拼装起来，并且指定分隔符

```go
fmt.Println(strings.Join([]string{"hello", "world"}, "/"))  // hello/world
```

`fmt.Sprintf()` 为我们提供了非常强大的拼装字符串的能力，而且能够指定输出的格式，与 `fmt.Printf` 类似，不同的是前者返回格式化后的字符串，后者将该字符串写入到标准输出流。比如将十进制数转化为对应的二进制字符串：

```go
s := fmt.Sprintf("%b\n", 100)
fmt.Println(s)  // 1100100
```

## Map 操作

如果使用过 Python 的 map filter reduce 函数的会觉得这样用起来写代码非常舒服，不用写很多循环，数据像是管道一样流动。

Go 对字符串也有对应的支持，下面演示一个对简单的[凯撒密码](https://en.wikipedia.org/wiki/Caesar_cipher)实现：

```go
func Caesar(s string) string {
    return strings.Map(func(r rune) rune {
        if 'a' <= r && r <= 'z' {
            r -= 3
            if r < 'a' {
                r += 26
            }
        }
        return r
    }, s)
}
```

## 字符串切割

```go
fmt.Println(strings.Split("hello world", "l"))  // [he  o wor d]
```

上面例子会把所有含有子串的都切割开，但是如果我们只是想切割指定数量的字符串呢？

```go
fmt.Println(strings.SplitN("hello world", "l", 2))  // [he lo world]
```

要求高一点想要保留子串怎么办？

```go
fmt.Println(strings.SplitAfter("hello world", "l")) // [hel l o worl d]
```

类似的，指定相应切割数量对应的方法：

```go
fmt.Println(strings.SplitAfterN("hello world", "l", 2)) // [hel lo world]
```

## 字符串复制增长 N 倍

增长字符串除了 `+` 运算符，`strings.Join` 方法还可以使用以下方法：

```go
fmt.Println(strings.Repeat("i", 10))    // iiiiiiiiii
```

## 大小写转换

这个功能手写也很简单，但是有轮子就别重复造了

```go
func ToLower(s string) string
func ToUpper(s string) string
```

Go 官方文档给出了一个例子，还可以自定义大小写转化的映射，有特殊需求的同学可以实现 unicode.SpecialCase。

```go
func ToLowerSpecial(c unicode.SpecialCase, s string) string
func ToUpperSpecial(c unicode.SpecialCase, s string) string
```

对应的还有 `strings.ToTitle`

## 统计子串出现的次数

```go
fmt.Println(strings.Count("hello world", "l")) // 3
```

## 去除特定前后缀

```go
func Trim(s string, cutset string) string
```

Trim 还有很多变种，比如 TrimPrefix、TrimSuffix 等。

## 子串替换

```go
fmt.Println(strings.Replace("hello world", "l", "L", 2))    // heLLo world
```

# strconv

strconv 下是一些字符串与其他基本类型的转换，如整数、浮点数与字符串之间的转换。

## 字符串与整形之间转换

Atoi / Itoa 与 C 语言中的字符串与整形之间的转换非常像：

```go
s := strconv.Itoa(-100) // "-100"
i, _ := strconv.Atoi("-100")    // -100
```

因为字符串转化为整形有可能输入值不合法，所以 Atoi 方法会返回一个错误值。

## strconv.ParseXxx

strconv.ParseXxx 将字符串转化为相应类型的数据：

```go
b, _ := strconv.ParseBool("true")
f, _ := strconv.ParseFloat("3.1415926", 64) // 最后一个参数指明 float 的 bitSize
```

```go
i64, _ := strconv.ParseInt("-100", 10, 32)
```

在字符串转化为整数可以指定进制，比如十进制、二进制、十六进制等等....虽然可以 bitSize 参数，但是返回的值还是 int64 类型。

二进制：

```go
bin, _ := strconv.ParseInt("1001", 2, 64)   // 9
```

十六进制：

```go
hex, _ := strconv.ParseInt("1a", 16, 64)    // 26
```

注意这里不能改写为 `0x1a`，如果加上 `0x` 或者 `0X` 前缀则无法解析。但可以加 `0` 前缀，比如改写为 `01a`。

八进制：

```go
oct, _ := strconv.ParseInt("-017", 8, 64)    // -15
```

类似十六进制、二进制，八进制也可以加 `0` 前缀，负数只需要再追前面加上 `-` 即可。

字符串转化为整数不能够出现小数点，也就是不能够输入一个浮点数格式的字符串。

无符号数的转换：

```go
u64, _ := strconv.ParseUint("100", 10, 64)  // 100
u64, err := strconv.ParseUint("-100", 10, 64)  // 0
```

如果无符号数转化中，字符串中形式是有符号的，则返回错误值，并且返回无符号数的零值也就是 0。

## strconv.FormatXxx

`strconv.FormatXxx` 与 `strconv.ParseXxx` 相反。其功能也可以通过 `fmt.Sprintf` 实现，`strconv.FormatXxx` 的相对优点是指定参数更加灵活，不需要写死到字符串中。

```go
b := strconv.FormatBool(true)   // "true"
```

```go
f := strconv.FormatFloat(3.1415926, 'f', 10, 32)    // "3.1415925026"
```

字符串的转化函数的参数比较多，具体可以参考[文档](https://golang.org/pkg/strconv/#FormatFloat)

```go
i := strconv.FormatInt(100, 2) // "1100100"
i = strconv.FormatInt(-100, 16)  // "-64"
```

```go
u := strconv.FormatUint(10, 2)  // "1010"
```

# 总结

**工欲善其事，必先利其器**

学习中、工作中需要掌握某些有利于提升我们效率的工具，比如对 IDE 快捷键、语言的标准库、第三方库等，要好好利用前人载的树来乘凉。Python 为什么在数据科学等领域如此火热呢？我想除了其自身特性外，更中要的是丰富的第三方库支持。很多不是 Python 开发的工具都有 Python 的接口实现，比如著名了 Tensorflow。

学习不仅需要花大量时间，更重要的是提升效率。

*使用 Golang 做算法题让我不禁怀念 Python 的语法糖*
