---
layout: post
title:  "《Go程序设计语言》笔记三"
date:   2018-03-03 12:00:05 +0800
categories: golang
---

# goroutine 和通道

执行 main 函数的 goroutine 结束后，所有的 goroutine 都会被强行终结，然后程序退出。

## Unix Socket  vs TCP/IP Socket

在使用 nginx 时有如下一段配置文件：

```
location / {
        include proxy_params;
        proxy_pass http://unix:/home/xxx/xxx/xxx.sock;
    }
```

而使用 Go 网络编程时候有 `net.Listen("tcp", "localhost:8080")` 创建 Listener，tcp 还能够被 unix 替代，这就让我感到好奇 Unix、Tcp/IP 两者到底有什么区别？

在 serverfault 中搜索到该答案：[https://serverfault.com/a/124518/458952](https://serverfault.com/a/124518/458952)

总结一下：
+ Unix 面向的是在同一操作系统内的进程间通信，TCP/IP 主要面向可以通过网络连接的两个操作系统之间的进程通信
+ Unix socket 比 TCP/IP socket 更快
+ Unix socket 连接到一个文件，TCP/IP 连接到 domainOrIP:port

## 结束命令行输入

```go
func main() {
    scaner := bufio.NewScanner(os.Stdin)
    for scaner.Scan() {
        s := scaner.Text()
        fmt.Println("You input:", s)
    }
}
```

可以发现输出永远无法结束。。。当初我就面对过这样的情况，Windows 下通过 `ctrl+z` 结束输入，Unix 下通过 `ctrl+d` 结束输入。

## channel

像 map 一样，通道是一个使用 make 创建的数据结构的引用。当复制或作为参数传递到一个函数时，复制的是引用，这样调用者和被调用者都引用同一份数据结构。通道的默认值是 nil。

同类型的 chan 可以通过 `==` 进行比较，如果两者都是同一个通道数据的引用时，比较值为 true，通道也可以和 nil 进行比较。

接收者使用 comma-ok 判断一个通道是否已经关闭。

```go
x, ok := <- ch
```

1. chan 已经关闭，`ok == false`
2. chan 还没关闭，`ok == true`

使用 for-range 循环从通道中读取元素，知道通道被关闭。

`close(channel)` 的调用通常是发送者。
`close(channel)` 操作并不是必须的，通道的关闭操作主要目的是告诉接收者没有更多元素需要通过通道传递。chan 可以被垃圾回收器自动回收，判断一个 chan 是否可以回收的标准是否还有变量可以访问到 chan，而不是 chan 是否已经关闭了。chan 并不是系统资源，而是 Go 运行时库提供的功能。

### 单向 chan

在函数参数声明时，
+ `in <-chan int` 只读通道
+ `out chan<- int` 只写通道

通道的只读和只写是通过编译时检查出来的，也就是编译器在编译代码时，如果发现有向只读 chan 中写数据，或向只写 chan 中读数据时，编译器会抛出编译失败错误。现在很多成熟的 IDE 和插件都可以做到静态语法检查功能，可以让我们写代码时避免此类错误。

### buffered chan

+ `cap(channel)` 判断带缓冲区通道的缓冲区长度
+ `len(channel)` 判断带缓冲区通道中元素的个数

### 多个 goroutine 同时写 chan，那么如何控制 chan 的关闭呢？

在多个 goroutine 同时写 chan 时，如果任意一个 goroutine 关闭了 chan 肯定会触发其他 goroutine 的 chan 的写操作宕机，最后引发整个程序的异常退出。

这个问题可以使用 `sync.WaitGroup` 来解决：

```go
func main() {
    var wg sync.WaitGroup
    tasks := []string{"hello", "world", "!!!"}
    ch := make(chan string)
    for _, work := range tasks {
        wg.Add(1)   // 确保 Add 在开始 goroutine 之前执行
        go func(work string) {
            defer wg.Done()	// defer 确保该语句能够被执行
            ch <- work
        }(work)
    }
    // 等待其他 goroutine 结束，然后关闭 channel，通知读者没有更多元素可以被读取
    go func() {
        wg.Wait()
        close(ch)
    }()
    for v := range ch {
        fmt.Println(v)
    }
    fmt.Println("finish...")
}
```

如果 `wg.Add(1)` 放在 worker goroutine 内执行的话，有一种特殊情况是，worker goroutine 全部还没有得到执行，closer goroutine 先执行，`wg.Wait()` 判断 **counter == 0**，然后就关闭了 chan，导致读者 for-range 没有读到内容且 worker goroutine 向 chan 中写数据时宕机。

closer goroutine 必须在 for-range 循环之前启动，不然进入 for-range 循环后，没有任何操作关闭通道，closer goroutine 则永远没有机会启动。

### 并行度控制

程序的并行度太高、无限制的并行通常不是一个好主意，因为系统中总有限制因素，例如，对于计算性应用 CPU 的核数，对于磁盘 I/O 操作磁头和磁盘的个数，下载流所使用的网络带宽，或 Web 服务本身的容量。

两个方法控制并发度：

1. 借助 buffer channel 控制最大的并发度，每次启动一个新的 goroutine 前先向 buffer channel 中写入一个元素，结束后从 buffer channel 中读取一个元素。从而控制只有一个 goroutine 结束后才能够启动新的 goroutine

```go
func main() {
    max := 10
    threshold := make(chan struct{}, max)
    for {
        threshold<- struct{}{}  // 获取令牌
        go handle()
    }
}

func handle() {
    defer func() {
        <-threshold             // 释放令牌
        }
    // do something
}
```

2. 启动特定数量的 goroutine，每个 goroutine 都是从 channel 中读取任务然后执行

```go
type Task struct{
    task string
}

func main() {
    max := 10
    tasks := make(chan Task)
    for i := 0; i < max; i++ {
        go handle(tasks)
    }
    for i := 0; i < 100; i++{
        tasks <- Task{task: fmt.Sprintf("%d", i)}
    }
    close(tasks)
}

func handle(taskList <-chan Task) {
    for task := range taskList {
        fmt.Println(task.task)
    }
}
```

## select

```go
select{}    // 永远等待
```

空的 select 几乎不会消耗 CPU，我做了以下实验得出该结论：

```go
func main() {
    runtime.GOMAXPROCS(2)   // 设置该程序使用最多的OS线程
    go func() {
        loop:
            for{
                goto loop
            }
    }()
    select {

    }
}
```

```go
func main() {
    runtime.GOMAXPROCS(2)
    go func() {
        loop:
            for{
                goto loop
            }
    }()
    loop:
        for {
            goto loop
        }
}
```

通过 top 命令查看当前运行的程序，查看到 CPU 占用率是 100%，如果 `select{}` 换成 `for{}` CPU 占用率是 200%。

## time.Tick

`time.Tick` 函数的行为很像创建一个 goroutine 在循环里面调用 `time.Sleep`，然后在它每次醒来时发送事件。如果停止监听 tick，但是计时器 goroutine 还在运行，徒劳地向一个没有 goroutine 在接收的通道中不断发送————**goroutine 泄露**

Tick 函数很方便使用，但是它仅仅在应用整个生命周期都需要时才适合。否则，我们需要使用以下这个模式：

```go
ticker := time.NewTicker(time.Second)
<-ticker.C      // 从 ticker 通道中获取元素
ticker.Stop()   // 释放资源，终止 ticker 的 goroutine
```

## 通知 goroutine 取消

chan 的零值是 nil，nil 通道有时非常有用，因为对 nil 通道发送和接收将永远阻塞。

一个 goroutine 无法直接观察到另一个 goroutine 的状态，有时候我们需要通过广播通知多个 goroutine 取消执行。这个功能可以配合关闭 channel 和 select 来做到。

如果一个 channel 已经关闭了，那么从该 channel 中读取数据不会阻塞当前 goroutine。

创建一个 channel 专门用于通知某些注册退出消息的 goroutine 退出执行，不向该 goroutine 中发送任何消息，通过关闭 channel，使得其他读操作，比如 select 中的 case 可以执行，每个注册该消息的 goroutine 都能收到退出消息，从而实现广播退出消息的功能。

 ```go
func main() {
    closeSignal := make(chan struct{})
    go handle(closeSignal)
    time.AfterFunc(5 * time.Second, func() {
        close(closeSignal)
    })
    time.Sleep(10 * time.Second)
    fmt.Println("main exit")
}

func handle(closeSignal chan struct{}) {
    loop:
        for {
            if cancelled(closeSignal){
                fmt.Println("Get cancel signal")
                break loop
            }
            fmt.Println("running...")
            time.Sleep(time.Second)
        }
        fmt.Println("goroutine exit")
}

func cancelled(closeSignal chan struct{}) bool {
    select {
    case <-closeSignal:
        return true
    default:
        return false
    }
}
 ```

## 使用共享变量实现并发

> 布要通过共享内存来通信，而应该通过通信来共享内存

可以限制只有一个 goroutine 能够访问变量，从而达到防止了并发带来的读写不安全。如以下的 balance 变量只能够在运行 teller 的 goroutine 访问。

```go
var deposits = make(chan int)
var balances = make(chan int)
type withdrawStruct struct{
    amount chan int
    result chan bool
}
var withdraw = withdrawStruct{amount: make(chan int), result: make(chan bool)}

func Deposit(amount int) {
    deposits <- amount
}

func Balance() int {
    return <- balances
}

func Withdraw(amount int) bool {
    withdraw.amount<- amount
    result := <-withdraw.result
    return result
}

func teller() {
    // 将 balance 变量限制在 teller goroutine 内
    // 避免多个 goroutine 访问带来的异常
    balance := 0
    for {
        select {
        case balances <- balance:
        case amount := <-deposits:
            balance += amount
        case amount := <-withdraw.amount:
            if amount >= balance {
                withdraw.result<- false
            } else {
                balance -= amount
                withdraw.result<- true
            }
        }
    }
}

func main() {
    go teller()
    for i := 0; i < 200; i++ {
        go Deposit(1)
    }
    for i := 0; i < 50; i++ {
        go Withdraw(1)
    }
    for i := 0; i < 10; i++ {
        go func() {
            fmt.Println(Balance())
        }()
    }
    time.Sleep(time.Second)
    fmt.Println(Balance())
}
```

或者使用 chan 来 pipline 中传递某个类型的指针，但是如果上层流水线 goroutine 将指针传递给下一级后便不会再通过指针访问变量，从而达到任何时刻变量只对一个 goroutine 可见。

比如蛋糕师做蛋糕分为三道工序，由三个师傅分别负责做蛋糕、加糖衣、加奶油。只要蛋糕传递给下一个师傅后，上一个师傅就不再对蛋糕做任何修改。

```go
type Product struct {
    state string
}

func Cake(cake chan<- *Product) {
    c := Product{state:"cake"}
    cake <- &c
}

func Sugar(cake <-chan *Product, ice chan <- *Product) {
    c := <-cake
    c.state = "sugar"
    ice <- c
}

func ice(ice chan *Product, product chan <- *Product) {
    c := <- ice
    c.state = "ice"
    product <- c
}
```

第三种方法避免竞态是允许多个 goroutine 同时访问同一变量，但在同一时间只有一个 goroutine 可以访问。采用*互斥机制*

无论是为了保护包级别的变量，还是结构中的字段，当使用一个互斥量 Mutex 时，都应该确保 Mutex 以及被保护的变量都没有导出。

### `sync.Mutex` vs `sync.RWMutex`

**sync.RWMutex** 仅在绝大部分 goroutine 都在获取读锁且锁竞争比较激烈时（即一般 goroutine 都要等待后才能获得锁），RWMutex 才有优势。因为 RWMutex 需要更加复杂的内部簿记工作，所以在竞争不激烈时 **sync.RWMutex** 比 **sync.Mutex** 慢。

如果计算机中服务器有多个 CPU，每个 CPU 中都拥有自己的 cache，cache 并不会实时地与内存保持同步，那么可能会导致多个运行在不同 CPU 上的 goroutine 对同一个内存变量观察到不一样的值。

```go
var x, y int
go func() {
    x = 1                       // A1
    fmt.print("y:", y, " ")     // A2
}
go fun() {
    y = 1                       // B1
    fmt.Print("x:", x, " ")     // B2
}
```

我们期待的输出是以下四个之一：

```
y:0 x:1
x:0 y:1
x:1 y:1
y:1 x:1
```

但事实上也可能出现以下两种情况：

```
x:0 y:0
y:0 x:0
```

一个原因可能是如 `x:0 y:0` 情况， A 和 B 运行在不同 CPU 上，导致 B1 在更新 y 的值，但是缓存还未同步到内存中，A2 读到 y 的值已经过期。

另一个原因可能是编译器对指令进行了重排，因为 A1 和 A2在同一个 goroutine 中应该谁先执行都不会影响对方的正确性，也就是经过编译器优化后执行顺序变为：A2 --> A1

**所以尽可能将变量的访问限制在单个 goroutine 中，对于其他变量使用互斥锁。**

### sync.Once 保证函数只被执行一次

有些场景下需要控制某个函数或者变量的方法只能被执行一次，那么我们可以通过 sync.Once 提供的 Do 方法做到。如下：

```go
func main() {
    var once sync.Once
    fn := func() {
        fmt.Println("hello world")
    }
    once.Do(fn)
    once.Do(fn)
}
```

hello world 只被输出一次。
当然如果自行调用 `fn()` 就可以逃避 sync.Once 的控制，看自觉。

## 竞态检测器

Go 语言运行时和工具链装备了一个精致并易于使用的动态分析工具：竞态检测器（race detector）

简单地把 **-race** 命令行参数加入到 go build、go run、go test 命令里边即可以使用该功能。它会让编译器为应用或测试构建一个修改后的版本，这个版本有额外的手法用于高效记录在执行时对共享变量的所有访问，以及读写这些变量的 goroutine 的标示。除此之外，修改后的版本还会记录所有的同步事件，包括 go 语言、通道操作、(*sync.Mutex).Lock 调用、(*sync.WaitGroup).Wait 调用等。

这个工具会输出一份报告，包含变量的标示以及读写 goroutine 当时的调用栈，通常情况下这些信息足以定位问题。

由于额外的簿记工作，带竞态检查的程序在执行时需要更长的时间和更多的内存，但即使很多生产环境的任务，这种额外开支也是可以接收的。对于那些不常发生的竞态，使用竞态检测器可以帮助节省数小时甚至数天的调式时间。

## 一秒内 channel 的读写次数

```go
func main() {
    ch := make(chan string)
    go func(ch <-chan string) {
        for {
            <-ch
        }
    }(ch)
    counter := 0
    deadline := time.After(1 * time.Second)
    flag := true
    for flag {
        select {
        case <-deadline:
            flag = false
            close(ch)
        default:
            ch <- "hello"
            counter++
        }
    }
    fmt.Println(counter)
}
```

在我电脑上输出结果为 **3353895**，也就是一秒内两个 goroutine 通过 channel 进行了三百多万次通信。