---
layout: post
title:  "Golang 基础"
date:   2018-02-02 09:00:05 +0800
categories: golang
---

# Exported Name

在 package 中，有且只有以大写开头的变量、函数、类型会被外面所访问

# Basic types

Go 基本类型

```
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128
```

int uint uintptr 是 32 位如果当前系统为 32 位，64 位如果当前系统为 64 位


# 输出格式

[fmt说明](https://golang.org/pkg/fmt/)

+ `%T` 类型
+ `%v` 值

# 初始值

+ 0 for number
+ false for bool
+ "" for string

# Type Conversion 类型转化

在 Go 中没有默认类型转化机制，如果类型不匹配将会抛出错误

基本类型转化 `T(v)` T 是类型， v 是值，如:

```golang
i := 10
var f float = float(i)
// var f float = i  # error
```

不同类型之间不能进行运算，甚至不能够进行比较 `int + float` error

# const

常量与变量声明不一致

```golang
// 变量，使用 :=
var i := 1
// 常量，使用 =
const Pi = 3.14
```

常量还可以作为枚举


常量中比较特殊的关键字 `iota`

Numeric constants are high-precision values. Go 将保留所有精确值

```golang
package main

import "fmt"

const (
	// Create a huge number by shifting a 1 bit left 100 places.
	// In other words, the binary number that is 1 followed by 100 zeroes.
	Big = 1 << 100
	// Shift it right again 99 places, so we end up with 1<<1, or 2.
	Small = Big >> 99
)

func needInt(x int) int { return x*10 + 1 }
func needFloat(x float64) float64 {
	return x * 0.1
}

func main() {
	fmt.Println(needInt(Small))
	fmt.Println(needFloat(Small))
	fmt.Println(needFloat(Big))
}
```

output:
```
21
0.2
1.2676506002282295e+29
```


## new make

new(T) 创建某个地址空间，返回对应的地址，然后将属性初始化为 zero value

make() 创建 slice / map / chan，并且完成对该类型的初始化，返回的是引用

