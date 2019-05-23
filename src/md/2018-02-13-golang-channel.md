---
layout: post
title:  "Golang happens before"
date:   2018-02-13 21:00:05 +0800
categories: golang
---

# happens before

**什么是 happens before？**

现代编译器会对指令进行重排，已达到更优的运行效果，如以下的程序：

```c
int a = 0
int b = 0

int main() {
    a = b + 1   // (1)
    b = 1       // (2)
}
```

(1) 操作一定是先于 (2) 操作完成的吗？

不一定。

可能编译器将上述代码翻译成：

```
加载 b 到寄存器X
进行 b = 1 赋值
将寄存器X中值加上1后赋值给 a
```

(1) 操作不会对 (2) 操作进行任何影响，即使 (2) 操作可能对 (1) 操作有影响，也不需要保证 (1) 操作一定在 (2) 操作之前完成，因为可以使用寄存器作为临时存储。也就是想说的，(1) dose not happen before (2)

happens before 指代事件 *e1* *e2* 存在某些影响，需要保证 *e1* *e2* 之间的开始和完成需要有一定的逻辑关系。

happens before 指的不是时序上的关系，描述的是一个写操作*w*对变量*v*的修改能够被读操作*r*感知到，那么*w* happens befoer *r*

怎么样的写操作*w*能够被读操作*r*检测到呢？

条件：

1. *w* happens before *r*
2. Any other write to the shared variable v either happens before w or after r

## Happens before 在 golang 中的体现

+ If a package *p* imports package *q*, the completion f *q*'s init functions happens before the start of any of *p*'s.

+ The start of the function main.main happens after all init function have finished.

+ The go statement that starts a new goroutine happens before the goroutines's execution begins.

+ The exit of a program is not guaranteed to happen before any event in the program.

+ A send on a channel happens before the corresponding receive from that channel completes.

+ The closing of a channel happens before a receive that returns a zero value because the channel is closed.

+ A receive from an unbuffered channel happens before the send on that channel completes.

+ The kth receive on a channel with capacity C happens before the k+Cth send from that channel completes.

+ For any `sync.Mutex` or `sync.RWMutex` variable l and n < m, call n of `l.Unlock()` hannpens before call m of `l.Lock()` returns

+ For any call to `l.RLock` on a `sync.RWMutex` variable l, there is an n such that the `l.RLock` happens afrer call n to `l.Unlock` and the matching `l.RUnlock` happens before call n+1 to `l.Lock`

# channel

Golang 中除了锁机制外，还可以通过 channel 来达到同步控制，使用通信进行同步。

1. 对未初始化 channel 操作会 block，甚至会导致 deadlock，因为 goroutine scheduler 检测到一直没有 goroutine ready-to-run over worker threads
2. 连续关闭 channel 导致 panic
3. 向已经关闭的 channel 发送数据触发 panic
4. 读取已经关闭的 channel 会返回 zero-value 不会阻塞

**如何检测 channel 是否关闭了？**

use comma-ok to check the channel is close or not.

```golang
v, ok := <- ch
```

or use for-range to read from channel till the channel is close.

```golang
for v := range ch {
    // do something
}
```

**如何实现非阻塞 channel 操作？**

使用 select-case 实现非阻塞 channel 通信

```golang
select {
    case c1 <- 1:
    // c1 可以写入
    case v <- c2:
    // c2 可以读取
    ...
    default:
    // 没有任何 case channel 可以读或写
}
```

**那如何控制 select 的最大等待时间呢？**

time 包可以帮助我们实现这一功能，time 包下有不少功能也是通过 channel 实现的，比如以下的一个炸弹：

```golang
func main() {
    tick := time.Tick(1 * time.Second)
    boom := time.After(5 * time.Second)
    for {
        select {
        case <-tick:
            fmt.Println("tick.")
        case <-boom:
        fmt.Println("BOOM!")
           return
        default:
            fmt.Println(".")
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

output:

```
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
BOOM!
```

`time.Tick` 可以控制该 channel 在指定时间后可读（永远不可写，因为函数返回的是一个只读 `<-chan Time`）

**如何通知goroutine退出**

通过从 channel 中读取数据实现

```golang
func main() {
    quitSignal := make(chan int)
    go work(quitSignal)
    time.Sleep(5 * time.Second)
    quitSignal <- 1
}

func work(quitSignal <-chan int) {
    for{
        select {
        case <-quitSignal:
            fmt.Println("收到结束信号")
            return
        default:
            fmt.Println("work")
            time.Sleep(time.Second)
        }
    }
}
```

**如何控制一个函数的最大并发数？**

通过 buffered channel 实现，每个需要执行该函数的 goroutine 向 buffered channel 发送一个数据，然后执行完成后从 channel 中取出一个数据

```golang
var concurrent = 3

var limit = make(chan uint8, concurrent)

// 控制并发包装器 exported
func Controller(args ...interface{}) {
    limit <- 1
    function(args)
    <-limit
}

// 需要被控制并发数量的函数，只有本包内可访问
func function(args ...interface{}) {

}
```

**如何实现定时操作？**

我们可以通过 `time.After` 自己实现一个，或使用现成的 `time.AfterFunc`

```golang
func main() {
    time.AfterFunc(2 * time.Second, func() {
        fmt.Println("After func")
    })
    fmt.Println("haha")
    time.Sleep(100 * time.Second)
}
```

`time.AfterFunc` 并不会阻塞当前 goroutine，而是使用了新的 goroutine

**如何保证一个函数只被执行一次？**

同样可以使用 channel buffer，只不过 buffer 大小为 1，执行前先向该 channel 发送数据

```golang
const limit = 1
var c = make(chan int, limit)

func main() {
    for i := 0; i < 100; i++ {
        go Controller()
    }
    time.Sleep(2 * time.Second)
}

// exported
func Controller()  {
    select {
    case c <- 1:
        once()
    default:
        fmt.Println("once 已经被执行过")
    }
}

// inner
func once() {
    fmt.Println("once")
}
```

同样的，需要修改执行的次数，只需要修改 limit

或者使用锁机制，控制控制只有一个变量进入，而且采用一个标识符记录该函数是否已经执行过，相应的实现在 `sync.Once`。`sync.Once` 提供一个安全机制，确保通过 `sync.Once.Do(f func())` 调用的 f() 最多只被执行一次。

以下是其内部代码，实现非常简单：

```golang
type Once struct {
    m    Mutex
    done uint32
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 1 {
    return
    }
    // Slow-path.
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
    defer atomic.StoreUint32(&o.done, 1)
    f()
    }
}
```

# select{} 与 for{} 区别

`for{}` 相信大家都不陌生，但 `select{}` 很让人疑惑。

+ `for{}` 会把 cpu 使用率到 100%
+ `select{}` 会阻塞当前 goroutine，cpu 使用率将近 0%


# 参考：

[Golang 内存模型](https://tiancaiamao.gitbooks.io/go-internals/content/zh/01.3.html)

[The Go Memory Model](https://golang.org/ref/mem)