---
layout: post
title:  "Golang 更多类型"
date:   2018-02-02 09:00:05 +0800
categories: golang
---

# Pointer

与 C 不一样，Go 中不支持关于指针的运算，比如 ptr ++ 等操作。

# struct

```golang
type Vertex struct {
    X int
    Y int
}

// create instance
v := Vertex{1, 2}
```

匿名结构体 struct

```golang
s := struct {
	i int
	b bool
}{1, false}
```

# array

创建数组会给予默认值

```golang
var a [2]string // a[0] a[1] is both ""

a := [...]int {1, 2, 3} // ... 要求编译器根据 {} 中数据计算数组长度，如果没有 ... a就变成了 slice
```

数组是基本类型，值类型，函数/赋值都会发生整个数组的复制，在真实应用中很容易产生灾难，传递大数组我们可以使用数组指针

```golang
var a = [2]string{"John", "Susan"}
p := &a
fmt.Println(p[0], p[1])
```

Go 中设计哲学，让使用指针像使用引用一样简单

# slice

> A slice does not store any data, it just describes a section of an underlying array.

> Changing the elements of a slice modifies the corresponding elements of its underlying array.

> Other slices that share the same underlying array will see those changes.

```golang
func slice1() {
	names := [4]string{
		"John",
		"Paul",
		"George",
		"Ringo",
	}
	fmt.Printf("%T %v\n", names, names)
	// 即使 names 是 [4]string array，但是 names 也能像 slice 一样使用
	// 创建 slice 底层的 array 复用原来的 names array，极度容易产生 bug
	a := names[0:2]
	b := names[1:3]
	fmt.Printf("%T %v\n", a, a)
	fmt.Printf("%T %v\n", b, b)

	b[0] = "Guang"
	fmt.Printf("%T %v\n", names, names)
	fmt.Printf("%T %v\n", a, a)
	fmt.Printf("%T %v\n", b, b)
}
```

output:
```
[4]string [John Paul George Ringo]
[]string [John Paul]
[]string [Paul George]
[4]string [John Guang George Ringo]
[]string [John Guang]
[]string [Guang George]
```

slice 可以通过以下方式生成:

```golang
var s1 []string

var arr [2]string{"John", "Susan"}

s2 := arr[0:]
```

`[3]bool{true, false, true}` 创建一个长度为 3 的数组

`[]bool{true, false, true}` 底层会创建一个数组，然后创建一个 slice 引用底层的数组

slice 是引用类型，赋值/函数传递发生的是引用地址的复制，对于大数组的传递比较有帮助

对于 `a := [10]int`

以下操作等级：

```golang
a[:]
a[0:10]
a[:10]
a[0:]
```

*length* *capacity*

+ length: slice 中元素的个数  `len(s)`
+ capacity: 底层数组的长度     `cap(s)`

slice 能在原有数组基础上扩张 extend，也就是直接把数组中后面的值内容读取进入 slice，length 因此变长

```golang
func test(){
    s := []int {2, 3, 5, 7, 11, 13}
    printSlice(s)

    s = s[:0]   // 清空 length，但是底层的数组和capacity不会改变
    printSlice(s)

    s = s[:4]   // 在原有数组上扩张 extend，length = 4，capacity和底层数组没哟改变
    printSlice(s)

    s = s[2:]   // 使数组可用长度较短两位，length-=2 capacity-=2，但是数组还是原来的数组
    printSlice(s)
}

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
```

slice 操作就是修改 length capacity 数组起始指针，在没有涉及到插入元素过多，或者删除了大部分元素后，底层数组的扩张和收缩前，一切都不会涉及到底层数组操作

`var s []int` len(s)=0 cap(s)=0 还没有底层数组，此时 `s == nil` == true

slice 只能够和 nil 进行比较

使用 make 函数创建指定 length capacity 的 slice，其中创建的默认数组长度 == capacity

```golang
b := make([]int, 0, 5)

c := b[:2]

d := c[2: 5]    // 这里即使 len(c) == 2，但是由于它指向的底层数组长度为5，所以这里 d 切片取得是底层数组 arr[2:5]
```

> Slicing does not copy the slice's data. It creates a new slice value that points to the original array. 

Slice grow，在 cap(slice) < 1024 会成倍增加 slice capacity，但是 cap(slice) > 1024 每次增长 1.25 倍    

```golang
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
    t[i] = s[i]
}
s = t
```

将一个 slice 拼接到另一个 slice 后面

```golang
a := []int{1, 2, 3}
b := []int{4, 5, 6}
a = append(a, b...) // ... 以此取出元素，相当于 python *list **dict
```

# range

range 用于遍历 array / slice / map

在 Go 中 `_` 是一个特殊变量，不会被真正赋值，所以使用 `_` 可以去掉 range 遍历中不感兴趣的变量

# Maps

The zero value of Map is nil. A nil map has no keys, nor can keys be added. 需要先 make 创建 map

```golang
var m map[string]string
m = make(map[string]string)
m["Hello"] = "World"
```

or 在声明时初始化

```golang
m := map[string]string {
    "Hello": "world",
    "Family": "Father and mother I love you"
}
```

Insert or update: `m[key] = value`

Delete: `delete(m, key)` if key not in map, no error

Retieve: `elem = m[key]`

Test a key is in map or not: `elem, ok = m[key]` if key is in map, ok = true; else key = false and elem is the zero value.

WordCounter:

```golang
func WordCount(s string) map[string]int {
	m := make(map[string]int)
	t := strings.Fields(s)
	for _, v := range t {
		if counter, ok := m[v]; ok {
			m[v] = counter + 1
		} else {
			m[v] = 1
		}
	}
	return m
}
```

# function 

函数也是 value，函数可以当做值在函数或者变量之间传递

函数闭包:

```golang

package main

import "fmt"

func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

内层函数引用外层函数的变量，这样会促使外层变量被保存在内存中。[闭包维基百科](https://zh.wikipedia.org/wiki/%E9%97%AD%E5%8C%85_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6))
