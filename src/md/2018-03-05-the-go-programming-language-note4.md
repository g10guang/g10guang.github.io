---
layout: post
title:  "《Go程序设计语言》笔记四"
date:   2018-03-05 12:00:05 +0800
categories: golang
---

# 包和 go 工具

## 包

Go 通过 import 语句导入的包一般在 **$GOROOT**、**$GOPATH** 下，**$GOROOT** 下导入的是 Go build-in 的包，比如 `math/rand`；而 **$GOPATH** 下导入的是通过 `go get` 命令从互联网上拉下的包，或者是自己之前写的程序。

1. 不管包的导入路径是什么，如果该包定义了一条命令（可执行 Go 程序），那么它总是使用名称 main。告诉 go build 必须调用连接器生成可执行文件
2. 目录中可能有一些名字以 _test.go 结尾，包名中会出现以 _test 结尾。这样的目录包含两个包一个普通的，一个外部测试包。`_test` 后缀告诉 go test 两个包都要构建，并且指明文件属于哪个包。外部测试包避免所依赖的导入图中的循环依赖
3. 有一些依赖管理工具会在包导入路径的尾部追加版本号后缀，如 `"gokpg.in/yaml.v2"`。包名不包含后缀，因此这种情况下包名为 **yaml**

### 重命名导入

```go
import (
    "crypto/rand"
    mrand "math/rand"
)
```

### 空导入

```go
import (
    _ "github.com/lib/pq"
)
```

空导入将包的名字重命名为 **_**，也就是无法在代码中引用该包中的变量和函数。这会产生什么影响呢？

虽然无法在代码只能够使用该包中的变量和函数，但该包下的包级别变量还是会进行声明和初始化，`init()` 函数还是会执行。如上空导入的作用就是加载 postgres 的数据库驱动，完成数据库驱动的注册。

## go 工具

### 工作空间组织

$GOPATH 下有三个文件夹，分别是：src、pkg、bin

1. src 存储源代码
2. pkg 存储构建工具编译后的包，go install 命令将会在 pkg 产生相应的 .a 文件
3. bin 存储可执行文件

go build 命令构建所有需要的包以及它们所有的依赖性，然后丢弃除了最终可执行程序之外的所有编译后的代码。显然如果当项目变得复杂，引用的包多了以后，重新编译依赖性的时间明显变慢。

go install 命令和 go build 命令相似，区别是它会*保存*每一个包的编译代码和命令，而不是把他们丢弃，结果保存在 $GOPATH/pkg 目录下。

go build 命令会特殊对待导入路径中包含路径片段 internal 的情况，这些包叫内部包。内部包只能够被位于以 internal 目录的父目录为根目录的树中。例如，给定下面的包，net/http/internal/chunked 可以从 net/http/httputil 或 net/http 导入，但是不能从 net/url 进行导入。

# 测试

在 *_test.go 文件中，是那种函数需要特殊对待，即功能测试函数、基准测试函数和示例函数。

1. *功能测试函数* 是以 Test 前缀命名的函数，用来检测一些程序逻辑的正确性，go test 运行测试函数，并且报告结果是 PASS 还是 FAIL。
2. *基准测试函数* 的名称以 Benchmark 开头，用来测试某些操作的性能，go test 汇报操作的平均执行时间。
3. *示例函数* 以 Example 开头，用来提供机器检查过的文档。

每一个测试文件必须导入 testing 包，这些函数签名如下：

```go
func TestXxx(t *testing.T) {

}
```

1. go test -v 输出包中每个测试用力的名称和执行的时间
2. go test -run="regexp" -run 参数是一个正则表达式，它可以使得 go test 只运行那些测试函数名称匹配给定模式的函数，而不用重新运行所有的测试用例

## 外部测试包

在编写测试用例的时候，为了避免包之间的循环引用。比如 net/http 依赖于 net/url，那么当我们在 net/url 保证写测试时候引用到了 net/http，这就出现了循环引用。解决方法是将测试写在 net/url_test 包中。

查看某个包内的 go 文件：

```
go list -f={{.GoFiles}} fmt
```

查看某个包内的测试文件：

```
go list -f={{.TestGoFiles}} fmt
```

查看某个包的外部测试包：

```
go list -f={{.XTestGoFiles}} fmt
```

如果一个包内的某些函数需要进行白盒测试，而这些函数又正巧是不可导出的呢？

这问题可以通过写像 fmt/export_test.go 类似的文件，使用一个可导出的别名指向该函数：

```go
package fmt

var IsSpace = isSpace
var Parsenum = parsenum
```

## cover 覆盖率

从本质上看，测试从来不会结束，因为可能的输入有无限多种情况。无论多少测试都无法证明一个包是没 bug 的，最好的情况下，这些包是可以在很多重要场景下使用的。

```bash
go tool cover
```

工具为我们查看测试代码的覆盖率起到了很好的帮助，它可以输出一个 html 文档，在浏览器中打开大大增强了可阅读性。

## benchmark 基准测试

基准测试就是在一定的工作负载之下检测程序性能的一种方法。在 Go 里面，基准测试函数看上去像一个测试函数，但是前缀是 Benchmark 并且拥有 *testing.B 参数用来提供大多数和 *testing.T 相同的方法，额外增加了一些与性能测试相关方法，其中提供了一个整形成员 N，用来指定被检测操作的执行次数。

```go
import "testing"

func BenchmarkXxx(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // test code
    }
}
```

默认情况下不会执行任何基准测试，标记 -bench 参数指定了要运行的基准测试，它是一个匹配 Benchmark 函数名称的**正则表达式**，它有默认值不匹配任何函数。模式 "." 使它匹配包 word 中所有的基准测试函数。

```go
go test -bench="."

go test -bench="Xxx"
```

## profile 性能剖析

性能剖析是通过自动化手段在程序执行过程中给予一些性能事件的采样来进行性能评价，然后再从这些采样中推断分析，得到统计报告。

Go 支持很多性能剖析方式，每一个都和一个不同方面的性能指标相关，但是它们都需要记录一些相关事件，每一个都有一个相关的栈信息————在事件发生时活跃的函数调用栈。工具 go test 内置支持一些类别的性能剖析。

**CPU 性能剖析**识别执行过程中需要 CPU 最多的函数。

```bash
go test -cpuprofile=cpu.out
```

**堆性能剖析**识别出负责分配最多内存的语句。

```bash
go test -blockprofile=block.out
```

**阻塞性能剖析**识别出那些阻塞协程最久的操作。

```bash
go test -memprofile=mem.out
```

性能剖析并非只能够通过命令行来启动，对于长时间运行的程序也可以开启性能剖析，Go 运行时的性能剖析特性可以让程序员通过 runtime API 来启用。

在获取性能分析结果后，需要使用 pprof 工具来分析它，基本的参数有两个，产生性能剖析结果的可执行文件和性能剖析日志。

```bash
go tool pprof
```

## Example 函数

函数以 Example 开头，既没有参数也没有返回结果，如：

```go
func ExampleXxx() {
    fmt.Println(Xxx())
}
```

示例函数有三个目的：
1. 作为文档，比起乏味的描述，举例子是描述函数功能最简洁直观的方法。和普通文档不一样，ExampleXxx 函数是 Go 代码必须要通过编译检查，它会随着代码的改变而改变。
2. 可以通过 go test 运行的可执行测试。
3. 提供手动实验代码。
