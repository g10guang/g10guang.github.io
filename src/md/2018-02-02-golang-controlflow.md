---
layout: post
title:  "Golang 控制流"
date:   2018-02-06 09:00:05 +0800
categories: golang
---

# for - loop

以下代码省略 condition statement 将会引起无限循环

```golang
for ;; {
    fmt.Println("hello world")
}
```

# if

相对与其他语言，Go 中支持 if 像 for-loop 一样先执行 init，init 中声明的变量只能在 `if {} else {}` 中使用

```golang
if v:=method(); v == 0 {
    statment()
}
// cannot use v from here
```

# switch

switch 是一连串 if-else 的简写，但是 Go 中 switch 与 C java 不同，Go 中只有命中的 case 会被执行。所以 break 相当于出现在每个 case 之后

```golang
func test_switch() {
	fmt.Println("Go runs on")
	switch os := runtime.GOOS; os {
	case "darwin":
		fmt.Println("OS X")
	case "linux":
		fmt.Println("Linux")
	default:
		fmt.Printf("%s\n", os)
	}
}
```

swithc without the condition == 选择第一个 true case，空 switch 有利于书写很长的 if-else 语句

```golang
func switch3() {
	t := time.Now()
	// without condition equals to select true
	switch {
	case t.Hour() < 12:
		fmt.Println("Good morning")
	case t.Hour() < 17:
		fmt.Println("Good afternoon")
	default:
		fmt.Println("Good evening")

	}
}
```

不想 C java 中 case 紧随一定要是常量，Go 中 case 后可以跟随任何变量，甚至函数

# fallthrough

switch-case 中强制执行命中 case 的下一条 case

```golang
func main() {
	// 空 switch 则相当于寻找第一个为 true 的 case
	switch {
	case 1+1 == 2:
		fmt.Println("1+1=2")
		fallthrough
	case 1 == 2:
		fmt.Println("fallthrough")
		fallthrough
	default:
		fmt.Println("default")
	}
}
```

output:

```
1+1=2
fallthrough
default
```

# defer

defer 语句能够推迟某些语句的执行，直到 return 语句附近

```golang
func defer1() {
	defer fmt.Println("world")
	defer fmt.Println("AAAA")

	fmt.Println("Hello")
}
```

defer 将语句放到最后执行，然后将语句执行的顺序完全相反。其背后是通过栈（FILO）实现的。

output:
```
Hello
AAAA
world
```