---
layout: post
title:  "读写锁实现"
date:   2018-10-10 20:00:05 +0800
categories: 系统
---

# In Short

面试中被问到如何实现读写锁，个人也比较感兴趣锁是如何实现的，最近也在看《深入理解计算机系统》看到相关内容，做了一个调研，在此 mark down。

> 以下代码是伪代码

# Main

在使用 C 语言时，我们可以采用 Mutex 来进行线程同步，那么 Mutex 是如何实现的呢？

每次调用 lock / unlock 都会触发一个系统调用，让内核来协调多个线程的同步，具体同步方式采用 FIFO 队列，避免饥饿。根据 Linux Man Page 中提到系统调用需要上百条指令，而且如果 lock 失败了，会导致进程（线程）上下文的切换，进程上下文切换涉及到，保存、恢复寄存器值，替换进程虚拟地址映射表，消耗较大。

如果竞争不是很激烈，可以采用 CAS（Compare And Swap）实现用户空间锁（user-space lock），从而避免系统调用和上下文切换。CAS 是原子操作，通过 CPU 指令实现，一直尝试直到获取到锁，这种实现方式称为自旋锁。

类似地 CPU 提供 Test-and-Set 原子指令也可以用作用户空间锁。

同是原子操作，Compare-and-Swap 和 Test-and-Set 区别（伪代码形式呈现，事实上只是一条指令）：

**Compare-and-Swap：**

```c
if (*ptr == oldVal) { *ptr = newVal; return true; }
else return false;
```

**Test-and-Set：**

```c
oldVal = *ptr; *ptr = newVal; return oldVal;
```

需要内核协调同步的方式是引入了内核这 **单点**，像 100 号人想进入由**门卫**把控的只允许若干个人（可能不止一个，如 Semaphore）同时进入的房间一样，线程就像人，内核就像门卫。

使用 CAS 原子操作实现的锁，像 100 号人想要进入没门卫把控的只允许若干个人同时进入的房间，没有门卫单点，但是所有人必须打死打残才能获取进入的机会。

## 借助 Mutex 实现的读写锁

C / C++ / Java 都有关键字 volatile，这关键字的作用是使编译器每次从内存中读取值 v，而不使用寄存器中的缓存。根据计算机系统中的存储结构，值 v 会被缓存在 CPU 中的高速缓存中，高速缓存一般是有多层的，有核独占的和共享的，所以是每次从共享高速缓存中读取值 v。

设有以下代码：

```c
int v = 0;
++v;
++v;
```

翻译成汇编的执行顺序：

1. 加载 v 到寄存器 r 中
2. 自增 r 中的值
3. 自增 r 中的值
4. 将 r 中的值写回到内存

如果在多线程下，每个线程都引用寄存器中的值而不是从内存取，那么值 v 将会有多个不同版本的拷贝。

如果加上 volatile 关键字：

```c
volatile int v = 0;
++v;
++v;
```

翻译成汇编的执行顺序：

1. 加载 v 到寄存器 r 中
2. 自增 r 中的值
3. 将 r 的值覆盖旧 v 的值
4. 加载 v 到寄存器 r 中
5. 自增 r 中的值
6. 将 r 的值覆盖旧 v 的值

仅仅通过 volatile 无法保证更新一个变量线程安全，因为 加载 ==> 更新 ==> 写回 被分开了多步骤实现，也就是读取的值有可能是脏值，写回有可能覆盖另一个线程的写入内容。

使用 kernel 提供的 mutex 实现读写锁的伪代码：

```c
volatile int read_cnt = 0;
mutex_t m;      // 保护 read_cnt 变量
mutex_t w;      // 写锁

void r_lock()
{
    lock(m);
    ++read_cnt;
    if (read_cnt == 1) {
        // first reader access write lock
        lock(w);
    }
    unlock(m);
}

void r_unlock()
{
    lock(m);
    --read_cnt;
    if (read_cnt == 0) {
        // last reader release write lock
        unlock(w);
    }
    unlock(m);
}

void w_lock()
{
    lock(w);
}

void w_unlock()
{
    unlock(w);
}
```

上述代码不是很有必要使用 volatile 保护 read_cnt 变量，因为 read_cnt 只在 r_lock / r_unlock 中访问，而且两个函数边界都是用了 lock(m) / unlock(m) 做保护，在函数执行时都会重新将值从内存加载到寄存器中。

设 cas 的几个接口：

```c
bool cas_set(void *ptr, int oldval, int newval) // return success or not
int cas_inc(void *ptr)  // return old value
int cas_dec(void *ptr)  // return old value
```

使用 CAS 实现读写锁的伪代码：

```c
int read_cnt = 0;
int write_cnt = 0;
int mutex = 1;         // protector

void r_lock()
{
    while (cas_set(&mutex, 1, 0))
        ;
    ++read_cnt;
    // first reader access write lock
    if (read_cnt == 1) {
        while (cas_set(&write_cnt, 0, 1))
            ;
    }
    while (cas_set(&mutex, 0, 1))
        ;
}

void r_unlock()
{
    while (cas_set(&mutex, 1, 0))
        ;
    --read_cnt;
    // last reader release write lock
    if (read_cnt == 0) {
        while (cas_set(&write_cnt, 1, 0))
            ;
    }
    while (cas_set(&mutex, 0, 1))
        ;
}

void w_lock()
{
    while (cas_set(&write_cnt, 0, 1))
        ;
}

void w_unlock()
{
    while (cas_set(&write_cnt, 1, 0))
        ;
}
```

以上方案会导致写饥饿，个人感觉写的优先级应该比读要高，应该优先处理写请求，改版为：

```c
int read_cnt = 0;
int write_cnt = 0;
int mutex = 1;         // protector
volatile int write_request = 0;

void r_lock()
{
    // if write request exist, just wait.
    while (write_request == 1)
        ;
    while (cas_set(&mutex, 1, 0))
        ;
    ++read_cnt;
    // first reader access write lock
    if (read_cnt == 1) {
        while (cas_set(&write_cnt, 0, 1))
            ;
    }
    while (cas_set(&mutex, 0, 1))
        ;
}

void r_unlock()
{
    while (cas_set(&mutex, 1, 0))
        ;
    --read_cnt;
    // last reader release write lock
    if (read_cnt == 0) {
        while (cas_set(&write_cnt, 1, 0))
            ;
    }
    while (cas_set(&mutex, 0, 1))
        ;
}

void w_lock()
{
    while (cas_set(&write_request, 0, 1))
        ;
    while (cas_set(&write_cnt, 0, 1))
        ;
}

void w_unlock()
{
    while (cas_set(&write_request, 1, 0))
        ;
    while (cas(&write_cnt, 1, 0))
        ;
}
```

# End

对于 Java / Golang 这种由虚拟机或运行时库调度用户级线程的编程语言，实现锁并不需要操作系统介入。

对于不同的需求，如每次随机选出一个线程执行，先申请先获得锁等，具体实现方式也不太一样，在解决业务需求如此复杂的场景 Java JDK 内置非常丰富的实现。
