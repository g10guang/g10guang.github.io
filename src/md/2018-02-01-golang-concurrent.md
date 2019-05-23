---
layout: post
title:  "Golang 并发基础"
date:   2018-02-06 09:00:05 +0800
categories: golang
---

# goroutine

> A goroutine is a light weight thread managed by the Go runtime.

```golang
go f(x, y, z)
```

`f, x, y, z` 的计算会发生在当前 goroutine，但执行在另一个新的 goroutine

goroutine 运行在同一个地址空间，所以访问共享内存时需要同步

# channel

Go 哲学不要使用共享内存通信，而是通过 channel 通信来达到共享内存的目的

channel 是用于 goroutine 之间的通信

通道是一个通信管道，可以通过通道操作符 `<-` 来接收，发送消息（PS: 估计类似与进程管道进行通信）

表现同时也像队列，先进先出。普通 channel 可以用来做同步控制(synchronization)，因为 channel 的一方必须等另一方准备完成才能进行读写，比如：

Sender 把一个数据放进 channel，Reader 还未进行读，此时 Sender 再次把数据放进 channel 就会阻塞 Sender，知道 Reader 把数据从 channel 读取出来

一个 goroutine 可以写和读 channel，但是不能够读自己写的数据，读自己写的数据将会发生阻塞，如果所有 goroutine 都 asleep 那么 go runtime 认为出现了死锁

```golang
ch <- v     // 发送 v 到 channel ch
v := <-ch   // 从 channel ch 接收一个值，并且赋予 v
```

数据流动方向就是箭头方向，`<-`

创建 channel

```golang
ch := make(chan int)
```

默认情况下，直到对方准备好，才开始发送和接收消息。这提供了 goroutine 之间的同步机制，而不是采用特殊的锁或条件变量

# buffered channel

channel 可以是有缓存的，提高缓存的长度用来初始化 channel

```golang
ch := make(chan int, 2)
```

> 向满的 buffered channel 中输入数据，或者是向空 buffered channel 中读取数据，会发生错误

# Range and Close

发送者可以 `close` 关闭 channel，表明没有再多的元素被发送；

接受者可以测试一个 channel 是否已经关闭 `v, ok := <- ch`

> 只有发送者才能够关闭 channel，向已经关闭了的 channel 发送数据会引发 `panic`

`ok == false` 如果 channel 中没有更多元素，channel 已经被关闭

使用 for 循环读取 channel 中的数据，直到 channel 被挂壁

`for i := range ch`

> 不像文件，关闭 channel 并不是必须的。只有当需要通知接受者没有更多的元素时，才需要主动关闭 channel，例如通知接受者停止 `for i := range ch` 循环

读取 buffered channel 目前元素个数 `length` 和可以容纳的最多元素个数 `capability`

```golang
len(ch)
cap(ch)
```

从 channel 中不断发送和读取 fibonacci 数列

```golang
func fibonacci(n int, ch chan int) {
    x, y := 0, 1
    for i := 0; i < n; i++ {
        ch <- y
        x, y = y, x + y
    }
    close(ch)
}

func main() {
    ch := make(chan int, 10)
    go fibonacci(cap(ch), ch)
    // 不断地读取元素，直到 channel closed
    for i := range ch {
        fmt.Println(i)
    }
}
```

# select

select 从多个 channel 中随机选取一个可读或者可写的 channel，执行该 case。

```golang
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- y:
			x, y = y, x+y
		// 从 quit 通道中读取元素，但是没有发生赋值
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		// 从 c 中读取 10 个元素，然后向 quit 中发送信号，让程序退出
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

## time.After

用于控制超时 select

```golang
select {
    case v := <-ch:
        doSomething()
    // time.After 可以保证一定时间后 channel 即可通信
    case <-time.After(1 * time.Second):
        timeout()
}
```

# sync.Mutex

sync.Mutex 用于同步，比如上锁等操作

```golang
mux.Lock()      // 上锁
defer mux.Unlock()  // 解锁
```

goroutine 不是 thread，每一个 goroutine 都拥有一个自己的调用栈。goroutine 开销很低，可以同时开启上千个 goroutine。

有可能一个 thread 中有上千个 goroutine。goroutine 多路动态复用 thread，也就是说真正执行计算的还是 thread，但是一个 thread 可能利用异步等技术来 concurrent 并发多个 goroutine 执行，并且减少切换 thread 上下文的成本

