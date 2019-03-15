---
layout: post
title: Linux 性能诊断——CPU 上下文切换 Context Switches
date: 2019-03-08 20:30:00 +0800
category: Linux
tag: [perf]
thumbnail: https://raw.githubusercontent.com/lightfish-zhang/media-library/master/image/201903/switch-icon.png
icon: note
---

* content
{:toc}

## 前言

Linux 是一个多任务操作系统，当比较多数量的进程抢夺 CPU 时，上下文切换的损耗会显得比较大，本文讲述如何衡量 Linux 系统的上下文切换的性能指标，以及上下文切换的基础知识。

![](https://raw.githubusercontent.com/lightfish-zhang/media-library/master/image/201903/linux-context-switch.jpg)

## 什么是 CPU 上下文切换

简单理解，CPU 上下文切换是更新了 CPU 寄存器的值，或者硬件触发信号，调用中断处理程序。

根据任务不同，有不同场景的上下文切换，有 **特权模式切换**，**进程上下文切换**，**线程上下文切换**，**中断上下文切换**


### 特权模式切换(系统调用)

特权模式切换是指系统调用过程，在同一个进程中进行“用户态－内核态－用户态”的切换过程，一次系统调用，会发生两次 CPU 上下文切换。这过程中，先保存　CPU 寄存器里原来用户态的指令位置，接下来要执行内核态代码，CPU 寄存器更新为内核态指令的位置，最后跳转到内核态运行内核任务。

与进程上下文切换不同的是，系统调用过程中，不会涉及到虚拟内存等进程用户态的资源，也不会切换进程。

了解更多 Linux Protection Rings 特权保护级别，看下图

![](https://raw.githubusercontent.com/lightfish-zhang/media-library/master/image/201903/linux-protection-rings.jpg)

不同级别可以访问的资源级别不一样，如最里层的内核空间 Ring 0 可以直接访问所有资源，最外层的用户空间 Ring 3 只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入内核中，才能访问特权资源。


### 进程上下文切换

进程上下文切换，是指从一个进程切换到另一个进程。进程是由内核来管理和调度的，进程的切换只能发生在内核态。

#### 进程的上下文有什么？

进程的上下文有虚拟内存、栈、全局变量等用户空间的资源，还包括内核堆栈、寄存器等内核空间的状态。

于是，切换进程的过程比系统调用过程要多一步，在保存内核状态与 CPU 寄存器之前，需要把进程的虚拟内存、栈等保存下来；等加载下一个进程的内核态后，需要更新下一个进程的虚拟内存、栈等。

#### CPU 上下文切换耗时

引用测试报告 [How long does it take to make a context switch?](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)，次上下文切换都需要几十纳秒到数微秒的 CPU 时间。这个时间还是相当可观的，特别是在进程上下文切换次数较多的情况下，很容易导致 CPU 将大量时间耗费在寄存器、内核栈以及虚拟内存等资源的保存和恢复上，进而大大缩短了真正运行进程的时间。

#### 切换进程对内存访问性能的影响

 Linux 通过 TLB（Translation Lookaside Buffer）来管理虚拟内存到物理内存的映射关系。当虚拟内存更新后，TLB 也需要刷新，内存的访问也会随之变慢。特别是在多处理器系统上，缓存是被多个处理器共享的，刷新缓存不仅会影响当前处理器的进程，还会影响共享缓存的其他处理器的进程。


#### Linux 如何进行进程调度

- 首先，为了公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。
- 第二，进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。
- 第三，当进程通过睡眠函数 sleep 这样的方法将自己主动挂起时，自然也会重新调度。
- 第四，当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行。
- 最后，发生硬件中断时(比如网卡硬盘键盘等)，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序。


### 线程上下文切换

我们这里讨论同一个进程下多个线程的上下文切换。

线程是调度的基本单位，而进程则是资源拥有的基本单位。于是，内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。

由于同一个进程下，多个线程共享虚拟内存等资源，所以线程的上下文切换，只需切换线程的私有数据、寄存器等不共享的数据。

### 中断上下文切换

中断是什么？

中断其实是一种异步的事件处理机制，可以提高系统的并发处理能力。中断分为两种，硬中断与软中断。

**硬中断**：比如网卡接收数据后，需要通知 Linux 内核有新的数据到了，通过硬件中断的方式发送电信号给内核，内核此时调用中断处理程序来响应下。这一步需要快速响应，防止时间过长影响系统的并发处理能力。于是，内核把网卡的数据读到内存中，更新一下硬件寄存器的状态，表示数据已经读好了。再发送一个软中断信号给用户程序，结束中断处理程序。

**软中断**：用户程序需要上面网卡接收的数据，正处于 read 或者 epoll 的系统调用的 Sleep 状态中。系统如何通知用户程序收数据呢？每个 CPU 都对应一个软中断内核线程，名 `ksoftirqd/$CPU_NUM`，它被软中断信号唤醒，从内存中找到网络数据，按照网络协议栈，对数据进行逐层解析和处理，再发到用户程序上（唤醒用户程序）


为了快速响应硬件的事件，中断处理会打断进程的正常调度和执行，转而调用中断处理程序，响应设备事件。而在打断其他进程时，就需要将进程当前的状态保存下来，这样在中断结束后，进程仍然可以从原来的状态恢复运行。

对同一个 CPU 来说，中断处理比进程拥有更高的优先级，所以中断上下文切换并不会与进程上下文切换同时发生。同样道理，由于中断会打断正常进程的调度和执行，所以**大部分中断处理程序都短小精悍**，以便尽可能快的执行结束。

跟进程上下文不同，中断上下文切换并不涉及到进程的用户态。所以，即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等用户态资源。**中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等。**

另外，跟进程上下文切换一样，中断上下文切换也需要消耗 CPU，切换次数过多也会耗费大量的 CPU，甚至严重降低系统的整体性能。所以，当你发现中断次数过多时，就需要注意去排查它是否会给你的系统带来严重的性能问题。

## 上下文切换频繁的性能问题

过多的上下文切换，会把 CPU 时间消耗在寄存器、内核栈以及虚拟内存等数据的保存和恢复上，缩短进程真正运行的时间，成了系统性能大幅下降的一个元凶。

### 从性能角度看待上下文切换

- **自愿上下文切换**，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
- **非自愿上下文切换**，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

## 模拟场景与测量工具

使用 `sysbench` 命令行工具可以模拟系统多线程调度的瓶颈，它可以造成大量的线程线程，造成频率大的非自愿上下文切换

```
# 以 10 个线程运行 5 分钟的基准测试，模拟多线程切换的问题
sysbench --threads=10 --max-time=300 threads run
```

测量工具： `vmstat`, `pidstat`

- 命令 `vmstat 5`， 每 5 秒输出 1 组数据，指标有：
    + cs（context switch）是每秒上下文切换的次数。
    + in（interrupt）则是每秒中断的次数。
    + r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。
    + b（Blocked）则是处于不可中断睡眠状态的进程数。

- 命令 `pidstat -wt 1` 每隔 1 秒输出 1 组数据，`-w` 表示输出进程切换指标，`-u`表示输出 CPU 使用指标，`-wt`表示输出线程的上下文切换指标

- `watch -d cat /proc/interrupts` 查看 `/proc` 这个虚拟文件目录下，Linux 的中断使用情况，值得注意的指标有：
    + `RES` 重调度中断，这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行，也称为处理器间中断（Inter-Processor Interrupts，IPI）



## Reference 

- [Preemptive Priority-Based Scheduling](http://www.embeddedlinux.org.cn/rtconforembsys/5107final/LiB0024.html)
- [Linux 特权级别 wiki](https://en.wikipedia.org/wiki/Protection_ring)
- [Linux 概念，中断与中断处理](https://zhuanlan.zhihu.com/p/53640307)