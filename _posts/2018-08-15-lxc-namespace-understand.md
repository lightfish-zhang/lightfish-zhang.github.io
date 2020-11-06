---
layout: post
title: Linux 虚拟容器系列——认识Namespace
date: 2018-08-15 20:30:00 +0800
category: Linux container
tag: [lxc]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/Linux_Containers_logo.png
icon: note
---


* content
{:toc}



## 前言

本文从浅显的角度讲解 Linux container 技术，配合一些命令、代码执行的步骤来帮助人直观的理解 namespace 在 Linux 的存在意义与作用，教会人如何去使用它。

以下命令与代码的执行环境：Arch Linux 4.19.16

## 认识 Linux 下的 namespace

认识新事物，首先要学会观察它，

- 第一步，执行 `lsns`命令，查看 Linux 下的 namespace

> lsns 列出了所有当前访问 namespace 的信息。namespace 的标识符是一个 inode 号。

Example，我电脑上的 namespace

```
➜  ~ sudo lsns
        NS TYPE   NPROCS   PID USER      COMMAND
4026531835 cgroup    216     1 root      /sbin/init
4026531836 pid       194     1 root      /sbin/init
4026531837 user      216     1 root      /sbin/init
4026531838 uts       216     1 root      /sbin/init
4026531839 ipc       216     1 root      /sbin/init
4026531840 mnt       209     1 root      /sbin/init
4026531860 mnt         1    40 root      kdevtmpfs
4026532008 net       193     1 root      /sbin/init
4026532202 mnt         1   246 root      /usr/lib/systemd/systemd-udevd
4026532259 mnt         1   539 root      /usr/bin/NetworkManager --no-daemon
4026532261 pid        22  1625 lightfish /usr/lib/chromium/chromium --type=zygote
...
```

如上所示，lsns 打印出我电脑上的namespace，以及访问这些 namespace 的进程信息。仔细数数，可以发现，Linux 的祖先进程 init 使用了 6 种 namespace，它们分别是: cgroup, pid, user, uts, ipc, mnt， 这6个 namespace 也代表了 Linux 系统下的大多数资源。以 Linux 内核下定义的宏定义说明，在 Linux 中可通过 man clone 命令参阅，以下摘抄部分:

```
名称        宏定义             隔离内容
Cgroup      CLONE_NEWCGROUP   Cgroup root directory (since Linux 4.6)
IPC         CLONE_NEWIPC      信号量、消息队列和共享内存 (since Linux 2.6.19)
Network     CLONE_NEWNET      网络设备、网络栈、端口等等 (since Linux 2.6.24)
Mount       CLONE_NEWNS       挂载点（文件系统） (since Linux 2.4.19)
PID         CLONE_NEWPID      进程编号 (since Linux 2.6.24)
User        CLONE_NEWUSER     U用户和用户组 (started in Linux 2.6.23 and completed in Linux 3.8)
UTS         CLONE_NEWUTS      主机名与域名 (since Linux 2.6.19)
```

Tip: 在 Linux “一切皆文件”的哲学下，除了用 lsns -p $pid 命令来查看某一个进程的 namespace 外，还可以查看 `/proc/$pid/ns` 特殊目录下的文件，proc 目录是 Linux 内核资源映射到文件系统的，都分配有 inode 节点号。

```
➜  ~ sudo ls -l /proc/1/ns
total 0
lrwxrwxrwx 1 root root 0 2月   4 23:25 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 2月   4 23:25 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 2月   4 23:25 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 2月   4 23:25 net -> 'net:[4026532008]'
lrwxrwxrwx 1 root root 0 2月   4 23:25 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 2月   6 15:18 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 2月   4 23:25 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 2月   4 23:25 uts -> 'uts:[4026531838]'
```

### namespace 的存活

一般来说当一个namespace中的所有进程都退出时，该namespace将会被销毁。也有例外，比如 `ip netns add net0` 命令创建一个名为 net0 的 net namespace 之后，没有进程去使用这个 net0，它也不会被销毁。因为从根本上来说，因为它的 inode 节点映射的文件存在。

哈，Linux “一切皆文件”的哲学，只有当 indoe 节点映射的文件都被删除、且没有进程使用这个 inode，inode 对应的资源才会被回收。

于是，如果要保存某一个进程下的 namespace，处理使用新进程继承这些 namespace，还可以通过 mount 方法来挂载这些 namespace 到另外一个文件，例如 `mount --bind /proc/$pid/ns/net netFile`，这个命令使某一个进程的 net namespace 的 inode 的链接数 +1。那么，即便$pid进程被销毁，只要 netFile 没有被 unlink 删除，该 net namespace 便一直存在。其他进程可以通过 setns 系统调用加入这个 net namespace 。


## namespace 相关API

### clone

- clone, 创建一个新进程，与 fork 类似，但是功能更多，fork 创建的子进程会共享或者复制父进程的资源，而 clone 可以选择性的继承父进程的 namespace 资源，且可以创建并使用新的 namespace。

clone 的 C 语言函数定义：

```c
       #define _GNU_SOURCE
       #include <sched.h>

       int clone(int (*fn)(void *), void *child_stack,
                 int flags, void *arg, ...
                 /* pid_t *ptid, void *newtls, pid_t *ctid */ );
```

- 参数child_func传入子进程运行的程序主函数
- 参数child_stack传入子进程使用的栈空间
- 参数flags表示使用哪些CLONE_*标志位
- 参数args则可用于传入用户参数


### setns

- setns，将当前进程加入到已有的namespace中

setns 的 C 语言函数定义:

```c
       #define _GNU_SOURCE             /* See feature_test_macros(7) */
       #include <sched.h>

       int setns(int fd, int nstype);
```

- 参数 fd 表示我们要加入的namespace的文件描述符。上文已经提到，它是一个指向/proc/[pid]/ns目录的文件描述符，可以通过直接打开该目录下的链接或者打开一个挂载了该目录下链接的文件得到。
- 参数 nstype, 指定一个或者多个上面的CLONE_NEW*，选择类型


### unshare

unshare 使当前进程退出指定类型的namespace，并加入到新创建的namespace

unshare 的 C 语言函数定义:

```c
       #define _GNU_SOURCE
       #include <sched.h>

       int unshare(int flags);
```

- 参数flags表示使用哪些CLONE_*标志位

调用unshare()的主要作用就是不启动一个新进程就可以起到隔离的效果，相当于跳出原先的namespace进行操作。这样，你就可以在原进程进行一些需要隔离的操作。Linux中自带的unshare命令，就是通过unshare()系统调用实现的


## Reference

- [Namespace概述](https://segmentfault.com/a/1190000006908272)
- [命名空间资源隔离](https://linux.cn/article-5057-1.html)
