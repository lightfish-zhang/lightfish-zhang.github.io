---
layout: post
title: 关于Linux的进程的常识
date:   2016-11-02 14:30:00 +0800
category: Linux 
tag: [linux, process]
---

* content
{:toc}


## 打开终端，开启进程之旅

- 简单讲一下终端，`ps -ax`可以查看不同终端执行的进程，在某终端执行`tty`可以查看该终端等价的特殊文件，玩玩`echo 'hi' > /dev/pts 0`或`echo 'hi' > /dev/tty`

- 当终端执行的时候，开启了一个会话session

```
proc1 | proc2 &
proc3 | proc4 | proc5
```

```
+----------------------------------------------------------------------------+
|                                                                            |
|  +----------------+   +------------------+  +--------------------------+   |
|  |                |   |                  |  |                          |   |
|  |  登录shell      |   |   proc1          |  |  proc3    proc4          |   |
|  |                |   |                  |  |                          |   |
|  |                |   |   proc2          |  |  proc5                   |   |
|  |                |   |                  |  |                          |   |
|  +----------------+   +------------------+  +--------------------------+   |
|                                                                            |
|     进程组                      进程组                      前台进程组         |
|                                                                            |
|                                                                            |
+----------------------------------------------------------------------------+
                               对话期
```

会话是若干个进程组的集合，会话可包含多个进程组，但只能有一个前台进程组（粗浅的讲，前台进程组接管了`/dev/tty`）

- 登录终端时，你或使用本地term软件，或者远程petty，连接Linux的过程中，创建一个新的会话，第一个进程组（login->shell），shell是一个进程组的Leader，也是Session Leader，会话的sid就是shell的pid。
- 执行 `proc1 | proc2 &` ，shell会fork一个子进程，执行`proc1`，同时把该子进程指定为新的进程组Leader，gid就是该子进程的pid，此时，新的进程组出来了，后来的`proc2`会加入在进程组。`&`表示该进程组在后台运行。
- 前台进程组，同上，`proc3 | proc4 | proc5`，在一个进程组执行，shell一般会调用`tcsetpgrp`设置它为前台进程组。因为它们一开始都是shell的子进程，shell会调用wait等待它们三个结束，当作业都结束后，shell会把自己提回前台。
- 另外，如果`proc3 | proc4 | proc5`在执行中，fork出其他子进程，该子进程也属于该进程组，但是，shell是不知道该子进程存在的，shell是不会调用wait等待它结束。
- 当终端shell退出，也就是Session Leader结束后，会向该会话内所有进程组发送信号`SIGHUP`

## 对前台进程组发送信号

- `man 7 signal` 查看各种信号的解释。
- `ctrl-c` 发送 `SIGINT` 信号给前台进程组中的所有进程。常用于终止正在运行的程序。
- `ctrl-z` 发送 `SIGTSTP` 信号给前台进程组中的所有进程，常用于挂起一个进程。
- `ctrl-\` 发送 `SIGQUIT` 信号给前台进程组中的所有进程，终止前台进程并生成 core 文件。
- `ctrl-d` 不是发送信号，而是表示一个特殊的二进制值，表示 `EOF` (end of file)，常用于结束输入，或者退出shell。
- `kill -s SIGXXX $pid` 发送信号

## shell下手动切换后台和前台

- `jobs` 查看此次会话中shell以外的后台进程组
- `fg %d` 把后台进程组切换前台运行
- `bg %d` 把后台挂起的进程组继续运行，与`kill -s SIGCONT $pid`作用一样

## 守护进程、孤儿进程、僵尸进程

### init进程
- 这个进程是系统的第一个进程，pid=1，它负责产生其他所有用户进程，是所有进程的祖先。

### 孤儿进程
- 某进程的父进程异常结束
- 某一条pid=x的进程结束，系统会扫描所有ppid为x的进程，把失去爸爸的进程交给init收养。
- 失去原父亲的守护进程和孤儿进程的区别是概念上的，守护进程的前父进程是人为结束的

### 守护进程
- 命名习惯，Linux的一些程序名以`d`字母结尾，大多是守护进程程序
- 守护进程有两种情况，一个是原父进程结束，被init收养，如果它原本在shell会话中执行，就不再属于shell会话
- 另一种是主动忽略一些信号

```c
//后台进程读取/写入终端输入产生下面两个信号，或者控制终端不存在情况读取和写入会产生
signal(SIGTTOU, SIG_IGN);
signal(SIGTTIN, SIG_IGN);

//按CTRL-C ,CTRL-\ CTRL-Z会向前台进程组发送下面这些信号
signal(SIGINT,  SIG_IGN );
signal(SIGQUIT, SIG_IGN );
signal(SIGTSTP, SIG_IGN );

//终端断开，会给会话组长或孤儿进程组所有成员发送下面信号
signal(SIGHUP,  SIG_IGN );

//还有有些信号也可以由终端shell产生
signal(SIGCONT, SIG_IGN );
signal(SIGSTOP, SIG_IGN );
```

#### 在shell中执行守护进程
- 像`mysqld`,`php-fpm`,`crond`这一类的程序，运行时就是以守护进程运行。
- shell执行一些非守护进程的程序，但是想令其以守护进程的方式运行，可以使用`nohup`，顾名思义，用途就是让提交的命令忽略`hangup`信号`SIGHUP`。

### 僵尸进程的爸爸不等它
- 【维基百科】在类UNIX系统中，僵尸进程是指完成执行（通过exit系统调用，或运行时发生致命错误或收到终止信号所致）但在操作系统的进程表中仍然有一个表项（进程控制块PCB），处于"终止状态"的进程。这发生于子进程需要保留表项以允许其父进程读取子进程的exit status：一旦退出态通过wait系统调用读取，僵尸进程条目就从进程表中删除，称之为"回收（reaped）"。正常情况下，进程直接被其父进程wait并由系统回收。进程长时间保持僵尸状态一般是错误的并导致资源泄漏。
- 虽然僵尸进程的父进程没有wait它，如果父进程结束，僵尸进程就不再是僵尸进程了，它会被init收养。
- init进程周期执行wait系统调用收割其收养的所有僵尸进程。

### shell 操作历史
- 指令`history`可以查看当前用户在shell输入指令的历史记录
- `ctrl-r` 可以快捷搜索历史记录
- 文件保存再用户目录下的`.bash_history`文件
- 这涉及到安全问题
- 在使用命令列時，如果想讓某些指令不要被紀錄在歷史紀錄中，暂时设置全局变量

```
export HISTCONTROL=ignorespace

```
- 永久设置，将以上命令加入文件`/etc/profile`

### 进程的状态

- 通过`ps -eo pid,stat,command`查看各种状态的进程

example:
```
    1 Ss   /sbin/init
    2 S    [kthreadd]
    3 S    [ksoftirqd/0]
    5 S<   [kworker/0:0H]
    7 S    [rcu_preempt]
 1530 SN   /usr/bin/fcitx-dbus-watcher unix:abstract=/tmp/dbus-DygTVJTBrR,guid=d6d201763594c0bab8010cc759699396 1520
 1583 S    /usr/lib/GConf/gconfd-2
 1588 Sl   sogou-qimpanel
 1635 Ssl  /usr/lib/upower/upowerd
```

- < (高优先级进程）
- N（低优先级进程）
- L（内存锁页，即页不可以被换出内存）
- s(该进程为会话首进程）
- l(多线程进程）
- +（进程位于前台进程组）。
- D  不可中断     Uninterruptible sleep (usually IO)
- R    正在运行，或在队列中的进程
- S    处于休眠状态
- T    停止或被追踪
- Z    僵尸进程
- W    进入内存交换（从内核2.6开始无效）
- X    死掉的进程
- 例如：Ssl说明该进程处于可中断等待状态，且该进程为会话首进程，而且是一个多线程的进程。

见 `man ps`

```
PROCESS STATE CODES
       Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process:

               D    uninterruptible sleep (usually IO)
               R    running or runnable (on run queue)
               S    interruptible sleep (waiting for an event to complete)
               T    stopped by job control signal
               t    stopped by debugger during the tracing
               W    paging (not valid since the 2.6.xx kernel)
               X    dead (should never be seen)
               Z    defunct ("zombie") process, terminated but not reaped by its parent

       For BSD formats and when the stat keyword is used, additional characters may be displayed:

               <    high-priority (not nice to other users)
               N    low-priority (nice to other users)
               L    has pages locked into memory (for real-time and custom IO)
               s    is a session leader
               l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
               +    is in the foreground process group
```


# 附录

## 信号
Linux系统中，安装`manpage-dev`，用`man 7 signal`查看

```
Portable Operating System Interface (POSIX)
POSIX.1中列出的信号：

信号 值 处理动作 发出信号的原因

SIGHUP 1 A 终端挂起或者控制进程终止
SIGINT 2 A 键盘中断（如break键被按下）
SIGQUIT 3 C 键盘的退出键被按下
SIGILL 4 C 非法指令
SIGABRT 6 C 由abort(3)发出的退出指令
SIGFPE 8 C 浮点异常
SIGKILL 9 AEF Kill信号
SIGSEGV 11 C 无效的内存引用
SIGPIPE 13 A 管道破裂: 写一个没有读端口的管道
SIGALRM 14 A 由alarm(2)发出的信号
SIGTERM 15 A 终止信号
SIGUSR1 30,10,16 A 用户自定义信号1
SIGUSR2 31,12,17 A 用户自定义信号2
SIGCHLD 20,17,18 B 子进程结束信号
SIGCONT 19,18,25 进程继续（曾被停止的进程）
SIGSTOP 17,19,23 DEF 终止进程
SIGTSTP 18,20,24 D 控制终端（tty）上按下停止键
SIGTTIN 21,21,26 D 后台进程企图从控制终端读
SIGTTOU 22,22,27 D 后台进程企图从控制终端写
```

## man命令

```
(1)用户在shell环境中可以操作的指令或可执行文件
(2)系统核心可呼叫的函数与工具等
(3)一些常用的函数与库文件,大部分为C的库
(4)装置文档的说明,通常在/dev下的档案
(5)配置文件或者是某些档案的格式
(6)游戏
(7)惯例和协议等,如Linux文件系统,网络协议,ASCII code等
(8)系统管理员可用的管理指令
(9)跟kernal有关的文件
```
