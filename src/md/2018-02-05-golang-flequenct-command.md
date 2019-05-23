---
layout: post
title:  "Golang 常用命令"
date:   2018-02-05 09:00:05 +0800
categories: golang
---

# go 基本命令给

## go install

先编译生成对应的 `.a` 文件，然后再移动到 `$GOPATH/pkg` 或者 `$GOPATH/bin` 目录下

```bash
go install
```

`-v` 参数显示底层执行信息

## go build

将该包打包为一个 `xxx.a` 文件，然后可以被其他包引用

```bash
go build
```

如果当前包为`package main`，则将此包打包为一个可运行程序；如果需要在 `$GOPATH/bin` 下生成可执行文件，那么需要执行 `go install`

`go build` 默认情况下会忽略，`_` 和 `.` 开头的go文件

**go build 参数说明**：

+ `-o-` 指定输出的文件名，可以带上路径
+ `-i` 安装相应的包，编译+`go install`
+ `-n` 把需要执行的命令打印出来，但是不执行，这样可以理解相应命令底层到底做了什么
+ `-p n` 指定并行可运行的编译数量，默认是 CPU 数量
+ `-a` 更新包，对标准包不起作用
+ `-race` 开启编译时自动显示数据竞争情况
+ `-v` 打印我们正在编译的包名
+ `-work` 打印编译时候临时文件夹的名称，如果已经存在就不删除
+ `-x` 打印出编译需要执行的命令，与`-n`不同的是 `-x` 执行打印输出的命令
+ `-ccflags 'args list'` 传递参数给 5c,6c,8c 调用
+ `-compile name` 指定编译器，`gccgo` or `gc`
+ `-gcflags 'args list'` 传递参数给 5g,6g,8g
+ `-installsuffix suffix` 为了和安装包区分开来
+ `-ldflags 'args list'` 传递参数给 5l,6l,8l
+ `-tags 'tags list'` 设置在编译时候可以适配的那些 tag

如果代码需要执行某些跨平台的操作，那么可以使用 `xxxx_linux.go` `xxxx_darwin.go` `xxxx_windows.go` 等命名，执行 go 命令只会采用以 `_platform` 当前系统名称结尾的文件

## go get

```bash
go get -u github.com/g10guang/xxxxx
```

-u 参数可以自动更新包，而且 go get 的时候会自动获取第三方依赖

目录结构：

```
$GOPATH
    src
        |-github.com
            |-g10guang
                |-xxxx
    pkg
        |-github.com
            |-g10guang
                |xxxx.a
```

go get 本质上可以理解为先把远程代码 `fetch` 下来，然后执行 `go install`

**参数说明：**

+ `-d` 只下载不安装
+ `-v` 显示执行的命令
+ `-u` 强制使用网络去更新包和它的依赖包
+ `-t` 下载测试所需要的包
+ `-fix` 在获取源码之后，先运行 fix，然后再去做其他事
+ `-f` 只有包含了 `-u` 参数时才生效，不去验证每一个 import 是否已经获取了，对于本地 fork 的包特别有效

## go clean

删除当前源码包和关联源码包里面编译生成的文件

**参数说明：**

+ `-i` 清除关联的安装的包和可运行文件，也就是通过 `go install` 生成的文件
+ `-n` 打印执行的相应命令，但是不执行
+ `-r` 循环清除 import 中引入的包
+ `-x` 输出执行的命令，并且执行

## go fmt

格式化 go 代码，也就是 vscode 失去焦点后自动执行的命令，这样可以保证一个团队中代码风格统一

**参数说明：**

+ `-l` 显示需要格式化的文件
+ `-w` 将格式化的结果输出到文件，而不是打印输出，如果不带该参数最终结果将不会输出到文件
+ `-r` 添加重写规则，方便做批量替换
+ `-s` 简化文件中的代码
+ `-d` 显示格式户前后的 diff，而不写入文件
+ `-e` 打印所有的语法错误到控制台
+ `-cpuprofile` 支持调试模式，写入相应的 `cpufile` 到指定文件

## go test

读取源码目录下的 `*_test.go` 文件，执行测试用例

**参数说明：**

+ `-bench regexp` 执行相应的 benchmark，例如 `-bench=.`
+ `-cover` 开启测试覆盖率
+ `-run regexp` 只运行匹配正则 regexp 的函数，例如 `-run=Array` 只运行所有以 Array 开头的函数
+ `-v` 显示测试的详细命令

## go tool

go tool 下聚集了很多命令，比如 `fix` `vet`

+ `go tool fix` 用来修复老版本的代码到新版本，自动修改变化的 API
+ `go tool vet directories | files` 分析代码的语法是否正确，比如检查 `fmt.Printf()` 中的参数是否正确，`return` 之后是否还有多余的代码

## go generate

用来在编译前生成某些代码，是通过分析源码中特殊的注释，判断需要生成某些特殊代码

`go generate` 是给当前包的开发人员使用的，而不是给使用该包的人使用的

比如我们经常使用 `yacc` 生成代码

```bash
go tool yacc -o gopher.go -p parser gopher.y
```

如果我们想让 `go generate` 替我们执行该命令，那么可以在代码中任意一个位置插入注释：

```golang
//go:generate go tool yacc -o gopher.go -p parser gopher.y
```

在编译时候执行：

```bash
go generate
go build
```

## godoc

查看文档的命令，安装 godoc：

```bash
go get golang.org/x/tools/cmd/godoc
```

比如查看 `net/http` 包的使用文档可以执行：

```bash
godoc net/http
```

查看某个函数文档：

```bash
godoc fmt Printf
```

查看函数源码：

```bash
godoc -src fmt Printf
```

本地运行 golang.org 站点中的文档，运行在特定端口上：

```bash
godoc -http=:8080
```

我们可以在浏览器查看 `127.0.0.1:8080` 查看文档

## go version

查看当前 go 版本

## go env

查看当前 go 的环境变量

## go list

列出当前包正在使用的包

## go run

编译并运行 go 程序

## go help cmd

查看某条命令的帮助文档