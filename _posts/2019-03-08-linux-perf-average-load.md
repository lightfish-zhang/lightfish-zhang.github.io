---
layout: post
title: Linux 进程性能问题——平均负载 Load 问题
date: 2019-03-08 20:30:00 +0800
category: Linux
tag: [perf]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201903/cpu_logo.jpeg
icon: note
---

* content
{:toc}


## 前言

有一回面试，面试官提了一个问题，cpu 使用率不高，但是 Load (平均负载) 很高，你如何查找问题？

当时我不明白 Load 的意思，面试官解释说这个指标反映不可中断状态的进程比较多。我遂根据过往后端开发经验，回答可能系统中 io 阻塞比较多，多发于网络 io 问题，用命令 `netstat -tnp` 看看 tcp 连接中 time_wait 状态多不多...

我知道我的回答很片面，事后复习，做笔记。

## 什么是平均负载

熟悉 Linux 者知道，使用 `top` `uptime` 命令可以查看 `load average` 指标。

使用 `man uptime` 查看 Load average 解释:


> System load averages is the average number of processes that are either in a runnable or uninterruptable state.   A  process  in  a  runnable state  is  either using the CPU or waiting to use the CPU.  A process in uninterruptable state is waiting for some I/O access, eg waiting for disk.  The averages are taken over the three time intervals.  Load averages are not normalized for the number of CPUs in a system, so a  load average of 1 means a single CPU system is loaded all the time while on a 4 CPU system it means it was idle 75% of the time.

理解关键地方，平均负载是指，在单位时间内，系统中处于 **可运行状态** 与 **不可中断状态** 的平均进程数，简称**平均活跃进程数**。值得注意的是，**它与 CPU 使用率没有直接关系**

使用命令 `ps aux` 可以查看进程的状态 stat，如本文要注意的：

- R 状态，可运行状态 ( Running / Runnable )，正在使用 CPU 或者正在等待 CPU 的进程
- D 状态，不可中断状态( Uninterruptitle Sleep, 又称 Disk Sleep )，正处于内核态关键流程中的进程，并且是不可中断的。

D 状态为何不可打断呢，举个例子，系统调用起硬件设备的 I/O 响应，为了保证数据的一致性，在磁盘设备返回数据前，它是不能倍其他进程或者中断打断的，如果被打断，就容易造成磁盘数据与进程数据不一致的问题。于是，不可中断(D)状态是系统对进程与硬件设备的一种保护机制。

**平均活跃进程数**，严格意义上，它是活跃进程数的指数衰减平均值（某个量的下降速度和它的值成比例）。通常情况下，理解为**单位时间上的活跃进程数**即可。

### CPU 利用率与平衡负载

从 CPU 角度来说，Load average 只是反映单位时间内占用 CPU 的进程数量，而 CPU 利用率与进程数量没有直接关系，我们可以使用命令 `top` `vmstat` 查看 CPU 的利用率，有以下几个指标：

- %us：表示用户空间程序的cpu使用率（没有通过nice调度）

- %sy：表示系统空间的cpu使用率，主要是内核程序。

- %ni：表示用户空间且通过nice调度过的程序的cpu使用率。

- %id：空闲cpu

- %wa：cpu运行时在等待io的时间

- %hi：cpu处理硬中断的数量

- %si：cpu处理软中断的数量

- %st：被虚拟机偷走的cpu


## 如何衡量合理的平均负载

一般来讲，Load average 低于 CPU 数量的话，机器性能满足服务需求，超出一些也没关系，Load average 不直接代表 CPU 利用率，可能是 io 阻塞比较多。当 Load average 高于 CPU 数量的 70%，就可能导致进程响应变慢，进而影响服务的正常功能。

### 从历史变化量来看

一般来讲，`top` `uptime` 提供 load average 三个时间点的指标，分别是：1分钟、5分钟、15分钟。这反映了系统最近的状态变化趋势。在实际生产环境中，我们需要做长期的监控记录。如果有异常的数值变化，比如平均负载数是CPU的两倍，需要分析调查问题。

### 从平衡负载与 CPU 利用率 这两类指标综合分析

两类指标的不同，组合出以下几种可能情况：

- Load average 高，CPU use 高，要么运行了 CPU 密集型进程(线程)，要么有大量等待 CPU 的进程（线程）调度
- Load average 高，CPU use 底，运行了 IO 密集型进程
- 两者都比较低，正常
- Load average 底，CPU use 高，这是不存在的


## 模拟案例与工具

我们如何分析平衡负载与 CPU 利用率这两类指标不同组合的案例，寻找造成指标变化的来源？

以下环境为 Linux Arch 4.19 / 4 CPU / 8G Memory

### 工具列表

- `stress` 系统压力测试工具
- `sysstat` 性能分析工具包：
    + `mpstat` 多核 CPU 分析性能工具，mp 的意思是 **multi processors** （多处理器）
    + `pidstat` 进程性能分析工具，pid 意为进程 ID。它用于查看进程的 CPU、内存、I/O以及上下文切换等指标


### 模拟场景

使用 stress 可以模拟以下场景

- CPU密集型进程

```shell
# 模拟一个进程， 对 cpu 使用率 100%，限时 600s
stress --cpu 1 --timeout 600
```

- IO 密集型进程

stress 的 `-i` 选项，spawn N workers spinning on sync()

```shell
# 模拟一个进程不停的执行 sync
stress -i 1 --timeout 600
```

- 大量进程的场景

```shell
# 模拟16个进程， 对 cpu 使用率 100%，限时 600s
stress --cpu 16 --timeout 600
```

### 工具指标

- `mpstat -P ALL 5` 监控所有 CPU，每隔5秒输出一组数据，注意指标 `%usr` 使用率，`%iowait` IO 阻塞时间，从这可以判断是 CPU 密集型还是 IO 密集型

- `pidstat -u 5 1` 统计间隔5秒内，使用过 CPU 的进程的数据，注意指标 `%usr` 使用率，`%wait` 等待使用 CPU 的时间，从这可以判断是否进程（线程）过多

