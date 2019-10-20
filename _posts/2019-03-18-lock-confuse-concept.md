---
layout: post
title: '互斥锁，同步锁，临界区，互斥量，信号量，自旋锁之间联系是什么'
tags: [read]
---

都在讲一件事：**【并发】**

而且可以再多加几个词：竞态条件，竞态资源，轮询忙等，锁变量，原子性，TSL，阻塞，睡眠，唤醒，管程。这样可以构成一个完整的逻辑链条。不要强记名词，要理解背后的逻辑。（注！下面讲的都专指线程，不是进程）

这些词的内在逻辑如下：

1. 核心矛盾是**“竞态条件”，**即多个线程同时读写某个字段**。**
2. 竞态条件下多线程争抢的是**“竞态资源”**。
3. 涉及读写竟态资源的代码片段叫**“临界区”**。
4. 保证竟态资源安全的最朴素的一个思路就是让临界区代码**“互斥”**，即同一时刻最多只能有一个线程进入临界区。
5. 最朴素的互斥手段：在进入临界区之前，用if检查一个bool值，条件不满足就**“忙等”**。这叫**“锁变量”**。
6. 但锁变量不是线程安全的。因为“检查-占锁”这个动作不具备**“原子性”**。
7. **“TSL指令”**就是原子性地完成“检查-占锁”的动作。
8. 就算不用TSL指令，也可以设计出线程安全的代码，有一种既巧妙又简洁的结构叫**“自旋锁”**。当然还有其他更复杂的锁比如“Peterson锁”。
9. 但自旋锁的缺点是条件不满足时会**“忙等待”**，需要后台调度器重新分配时间片，效率低。
10. 解决忙等待问题的是：“**sleep”**和“**wakeup”**两个原语。sleep阻塞当前线程的同时会让出它占用的锁。wakeup可以唤醒在目标锁上睡眠的线程。
11. 使用sleep和wakeup原语，保证同一时刻只有一个线程进入临界区代码片段的锁叫**“互斥量”**。
12. 把互斥锁推广到"N"的空间，同时允许有N个线程进入临界区的锁叫**“信号量”**。
13. 互斥量和信号量的实现都依赖TSL指令保证“检查-占锁”动作的原子性。
14. 把互斥量交给程序员使用太危险，有些编程语言实现了**“管程”**的特性，从编译器的层面保证了临界区的互斥，比如Java的synchronized关键字。
15. 并没有“同步锁”这个名词，Java的synchronized正确的叫法应该是“互斥锁”，“独占锁”或者“内置锁”。但有的人“顾名思义”叫它同步锁。



几个点稍微展开一下，

首先，不是所有变量都可以是竞态资源。以Java为例，表示对象状态的成员字段可以构成竞态资源。方法内部的局部变量就不是竞态资源，因为局部变量的生命周期仅局限于方法栈，不能横跨多个线程。

```java
public class EvenGenerator {
    private int even = 0;     // 竟态资源

    public int nextEven() {   // |
        ++even;                   // | -> 临界区
        ++even;                   // |
        return even;            // |
    }
}
```

上面代码里的EvenGenerator专门生成偶数。但并发场景下，它会错误返回奇数。原因就是当多个线程拿到同一个EvenGenerator对象的引用以后，比如线程A刚执行完第一次自增操作被挂起，线程B接手进行2次自增以后，返回的就是奇数。然后线程A继续执行，再自增一次以后，也返回奇数。此时even成员字段构成“竞态资源”。访问竟态资源的代码nextEven()函数就是临界区。并发环境中，对象的公有状态（能通过公有方法访问也算）暴露给多个线程就构成竞态条件，是很危险的。



然后最直观的保护nextEven()函数的代码会把调用写成下面这样，就是“锁变量”，

```java
if (!occupied) {                              // 检查
    occupied = true;                       // 占锁
    critical_rigion();                        // 临界区
    occupied = false;                     // 释放锁
}
```

但这个做法并没有卵用。因为A线程完全可能在检查完occupied锁变量，确认锁没有被占用以后立刻被挂起。B线程抢占锁。这时候再切回A线程，因为已经检查过锁变量，它也占锁，进入临界区。这时候就同时有两个线程站在锁上，互斥失败。



自旋锁的关键就是用一个while轮询，代替if检查状态，这样就算线程切出去，另一个线程也因为条件不满足循环忙等，不会进入临界区。这是一个非常常用的结构，不光用在自旋锁，基本是使用条件变量wait()，notifyAll()时候的一种惯用法。

```java
// 线程A
while (true) { 
    while (turn != 0) {}         // 锁被占，循环忙等。
    critical_rigion(); 
    turn = 1;                      // 释放锁
    noncritical_rigion(); 
} 
// 线程B
while (true) { 
    while (turn != 1) {}         // 锁被占，循环忙等
    critical_rigion(); 
    turn = 0;                    // 释放锁
    noncritical_rigion(); 
}
```



但刚才说了自旋锁的缺点是循环忙等。如果并发的线程不像进程调度那样在时间片用完以后会自动切换上下文，就会形成死锁。所以最好在条件不满足的时候，让出线程的控制权，让其他线程有机会执行来使条件满足。这就是sleep原语做的事情。并且配套的wakeup原语会在条件满足的情况下唤醒。

结合TSL指令原子性的“检查-占锁”，以及sleep阻塞并让出线程执行权的思想，就是“互斥量”做的事。下面是pthread_mutex_lock的实现（摘自《现代操作系统》）

```c
mutex_lock:
    TSL REGISTER,MUTEX    |将互斥量复制到寄存器，并且将互斥量重置为1
    CMP REGISTER,#0         |互斥量是0吗？
    JZE ok                          |如果互斥量为0，解锁，返回
    CALL thread_yield          |互斥量忙，调度另一线程
    JMP mutex_lock            |稍后再试
ok: RET                            |返回调用这，进入临界区
```