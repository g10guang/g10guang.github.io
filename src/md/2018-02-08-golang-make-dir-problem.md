---
layout: post
title:  "Golang 创建文件权限问题"
date:   2018-02-08 09:00:05 +0800
categories: golang
---

# 问题描述

今天学习 Golang 文件操作实践时，当我创建一个文件（夹）出现文件权限与我代码设置不一致的问题

以下为我创建文件夹的代码：

```golang
func main() {
	err := os.MkdirAll("g10guang/t1/t2", 0666)
	if err != nil {
		fmt.Println(err)
	}
}
```

output:

```
mkdir g10guang/t1: permission denied
```

> g10guang 文件夹创建成功，可是下面的 t1 文件夹创建失败

通过 `ls -l` 命令查看 g10guang 文件夹信息

```
drw-rw-r-- 2 g10guang g10guang 4096 Feb  8 13:23 g10guang
```

# Linux 权限

比如：

```
drw-rw-r-- 2 g10guang g10guang 4096 Feb  8 13:23 g10guang
```

开头有 10 个字符，我们来一一分析：

第一个 `d` 代表这是一个文件夹，`-` 为文件，`l` 为链接

当前用户与文件关系分为三个等级：
+ 所有者
+ 同组用户
+ 其他用户

而后面的 9 个字符分别代表文件所有者、同组用户、其他用户对文件的读写权限

+ `r` 读权限
+ `w` 写权限
+ `x` 执行权限
+ `-` 不具有该权限

`rwx` ==> 111

`rw-` ==> 110

`r--` ==> 100

`---` ==> 000

如权限 `-rwxrw-r--` 表示这是一个文件，文件所有者拥有读写执行权限、同组用户有读写权限、其他用户只有读权限

可以通过 `chmod` 改变一个文件的权限，或者 `chown` 改变文件的所有者

我们创建文件夹时候指定的权限是 `0666`，这是一个八进制数字，拆分成二进制为 `110110110`，也就是 `rw-rw-rw-`，但是为什么我们创建的文件夹的权限是 `rw-rw-r--` 也就是 `0664` 呢？

其实我们创建文件夹、文件时候都是经过 Linux 系统调用完成的，所以这是 Linux 创建文件时候会把用户指定的权限 `-` 减去 `umask`，而我本地 umask 为 `002`。通过 `umask` 命令查看 umask


# 分析

通过 `g10guang` 用户运行代码，文件夹当然所属于 `g10guang`

尝试 `cd g10guang`，出现错误，错误信息：`cd: permission denied: g10guang`

文件夹是属于我的，我怎么不能够 `cd` 呢？

原因是：需要 `cd` 到一个文件夹，那么必须要有当前文件夹的执行权限 `x`，Linux 下设计一切皆文件，文件夹也是一个特殊的文件

如果使用 `os.MkdirAll` 方法创建文件夹时，必须基于文件夹所有者 `x` 执行权限以及 `w` 写权限，为什么需要写权限，可以通过  `vim` 去打开一个文件夹，可以看到文件夹里的信息（记得吗，文件夹也是文件），比如我一个文件夹通过 vim 打开的信息：

```vim
" ============================================================================
" Netrw Directory Listing                                        (netrw v155)
"   /home/g10guang/Templates/blog
"   Sorted by      name
"   Sort sequence: [\/]$,\<core\%(\.\d\+\)\=\>,\.h$,\.c$,\.cpp$,\~\=\*$,*,\.o$,\.obj$,\.info$,\.swp$,\.ba
"   Quick Help: <F1>:help  -:go up dir  D:delete  R:rename  s:sort-by  x:special
" ==============================================================================
../                                                                                                      
./
.git/
.sass-cache/
_posts/
_site/
g10guang.github.io/
.gitignore
.gitmodules
404.html
Gemfile
Gemfile.lock
_config.yml
about.md
index.md
```

文件夹中记录着里面有哪些文件以及文件夹，其中 `xxx/` 有 `/` 结尾的是文件夹，其他的是文件，所以在文件夹中创建一个文件夹需要改变文件夹信息，需要有写文件夹的权限

# 创建指定权限文件方法

两种方法：

1. 改变 `umask` 后再创建文件，其后再把 `umask` 改为原来的 umask
2. 先创建文件，然后再改变文件的权限

## 方法一

改变 `umask` 后再创建文件，其后再把 `umask` 改为原来的 umask

```golang
import (
	"os"
	"fmt"
	"syscall"
)

func main() {
	mask := syscall.Umask(0)    // 改为 0000 八进制
	defer syscall.Umask(mask)   // 改为原来的 umask
	err := os.MkdirAll("g10guang/t1/t2", 0766)
	if err != nil {
		fmt.Println(err)
	}
}
```

## 方法二

先创建文件，然后再改变文件的权限

```golang
import (
	"os"
	"fmt"
)

func main() {
	err := os.MkdirAll("g10guang/t1/t2", 0777)
	if err != nil {
        fmt.Println(err)
    }
    os.Chmod("g10guang", 0777)
    os.Chmod("g10guang/t1", 0777)
    os.Chmod("g10guang/t1/t2", 0777)
}
```

golang 还不支持递归更改多个文件夹的权限，所有需要一个一个调用。

# 总结

Linux 文件操作都是调用 Linux 的系统调用完成的，虽然 `Python` 、`java` 等创建一个文件不会让显示让编程人员指定所创建的文件的权限，但是 Golang 需要，所以我们编程还是需要了解一点内核知识，比如网络编程涉及到 `I/O`，而 Linux 下 I/O 有I/O 阻塞、I/O非阻塞、I/O复用、SIGIO 、异步I/O，而I/O复用中有 select / poll / epoll 等，上次面试被问到 epoll 时，闻所未闻，[更多I/O讲解](https://tech.youzan.com/yi-bu-wang-luo-mo-xing/)。

现在找实习、找工作，发现公司会看重面试者是否有阅读源码等经验，因为我们不仅仅需要懂得调用别人的 API 完成某个功能，而且需要某个 API 底层到底完成了哪些操作，出了问题如何定位问题，以及如何解决问题。
