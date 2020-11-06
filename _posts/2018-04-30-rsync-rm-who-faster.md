---
layout: post
title: 删除操作，rsync与rm谁更快？
date:   2018-04-30 19:30:00 +0800
category: shell 
tag: [shell]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/2018/Red-delete-button.jpg
icon: note
---

* content
{:toc}

## 前言

公司有台服务器产生太多临时文件，同事在删除文件的时候，说使用 rsync 会更快一些，使用 rm 可能会把机器搞挂，还引用网上一篇文章说　

"rsync所做的系统调用很少：没有针对单个文件做lstat和unlink操作。命令执行前期，rsync开启了一片共享内存，通过mmap方式加载目录信息。只做目录同步，不需要针对单个文件做unlink"

我对此抱有好奇与怀疑，在我的Linux知识中，从 Linux 中删除文件，需要是文件的硬连接数n_link归零、进程正在打开该文件的数n_count归零，才可以触发文件系统对 inode 与对应磁盘块的回收，实现删除操作。这是金科玉言，彻底删除文件的系统调用必然用到 unlink 与 close。

对于网传的理论，我在互联网上仔细搜索，发现太多转载的雷同的文章，却没有精华文章对上面的话做详细的解释。我决定自己研究下

## 删除大量文件的方法

网传这样的方法

例如删除某目录下一万个以上小文件，使用 `rm * -rf`，既有操作上的风险，又耗时。

建议使用 `rsync`

```
mkdir /tmp/blank_dir
rsync --delete-before -a -H -v /tmp/blank_dir/ target_dir
```
- 先建立空目录，再同步空目录到目标目录
- 注意以上命令中 blank_dir 后带 `/`

rsync 选项说明：
--delete-before 接收者在传输之前进行删除操作
--progress 在传输时显示传输过程
--a 归档模式，表示以递归方式传输文件，并保持所有文件属性
--H 保持硬连接的文件
--v 详细输出模式
--stats 给出某些文件的传输状态

### 为什么删除一万个文件以上, rsync 比 rm 快

从根本入手，直接查看系统调用情况，于是动手测试

实验环境：Linux Arch 4.19

创建一定数量的空白文件，分别使用 rm 与 rsync 删除，并使用 `strace -c` 统计系统调用，需要额外注意的是 rsync 在本地使用了三个进程(generator, sender, receiver)，所以需要 `-f` 选项告诉 strace 同时跟踪所有fork和vfork出来的进程。(由于 strace 输出的信息太多，为了阅读体验，打印详情在附录)

第一次，创建10个文件，分别删除，查看统计输出

查看 rm 的系统调用，耗时0.000000，总次数62

```
for i in $(seq 10); do touch tmp_$i;done 
strace -c rm * -rf
```

查看 rsync 的系统调用，总耗时0.008647，总次数365

```
for i in $(seq 10); do touch tmp_$i;done
strace -c -f rsync --delete -a -H ../blank_dir/ ./
```

因为10个文件的删除，几乎看不到时间，我第二次测试，删除一万个文件，结果：

rm 的系统调用，总耗时0.201209，总次数20063

rsync 的系统调用，总耗时0.625734，总次数20374

从这个结果来看，似乎 rsync 比 rm 要慢，这里有我使用 `strace -f` 统计 rsync 三个进程总耗时的原因，改用使用 time 命令来计时，删除一万个文件以上，rsync 确实是比 rm 快上一些，那是因为我电脑cpu在三个以上，三个进程的rsync当然快一些

### 我的结论

网传的 "删除多个文件，rsync 比 rm 快" 的方法，我认为不一定准确，理由如下：

- 我电脑版本为Linux Arch 4.19，测试中 rm 与 rsync 同样使用了 mmap 的磁盘映射内存的性能优化操作，这个可以在附录中看到。
- 无论是 rm 还是 rsync，都使用了 unlink， unlink 的次数等同文件数量, 这与网传的不符合。
- 可能旧版本 Linux 的 rm 优化做的不好，对目录的文件列表，缺少优化的缓存处理
- rm 命令一般是单进程的，而 rsync 有三个不同职责的进程(sender, receiver, generator)，在多核机器上，rsync 执行效率更高，但是在单核机器可能表现比较低
- 网传的 rsync 删除多文件的效率高是因为“目录同步”，我多加搜索也没有详细说明，认为这是误传。

我想，可能是有人对 rsync 的评测不严谨，在本地删除文件时，漏了检查 rsync 的两个进程的系统调用，才导致的以讹传讹。

### 附录

第一次，创建10个文件，分别删除，查看统计输出

使用 rm :

```
for i in $(seq 10); do touch tmp_$i;done 

strace -c rm * -rf
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         4           read
  0.00    0.000000           0         6           close
  0.00    0.000000           0         3           fstat
  0.00    0.000000           0         1           lstat
  0.00    0.000000           0         4         1 lseek
  0.00    0.000000           0         8           mmap
  0.00    0.000000           0         4           mprotect
  0.00    0.000000           0         1           munmap
  0.00    0.000000           0         3           brk
  0.00    0.000000           0         1           ioctl
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         2         1 arch_prctl
  0.00    0.000000           0         3           openat
  0.00    0.000000           0        10           newfstatat
  0.00    0.000000           0        10           unlinkat
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000                    62         3 total
```

使用 rsync, strace 的输出中可以看到 rsync fork 出两个子进程

```
for i in $(seq 10); do touch tmp_$i;done       
strace -c -f rsync --delete -a -H ../blank_dir/ ./
strace: Process 17207 attached
strace: Process 17208 attached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 61.13    0.005286         587         9         5 wait4
  5.39    0.000466          46        10           unlink
  3.80    0.000329           9        34         1 select
  3.59    0.000310           8        37           mmap
  3.27    0.000283           5        51           read
  2.88    0.000249          62         4           getdents64
  2.68    0.000232          29         8           munmap
  2.43    0.000210           5        37           close
  2.36    0.000204           8        23         4 openat
  2.32    0.000201          10        19           write
  2.32    0.000201          14        14           lstat
  1.28    0.000111           5        19           fstat
  1.14    0.000099          12         8         8 connect
  0.83    0.000072           9         8           socket
  0.73    0.000063          31         2           utimensat
  0.67    0.000058           3        17           fcntl
  0.49    0.000042          14         3           socketpair
  0.45    0.000039           5         7           lseek
  0.40    0.000035          17         2         2 nanosleep
  0.38    0.000033           3        11           mprotect
  0.37    0.000032           2        12           rt_sigaction
  0.24    0.000021          10         2         1 stat
  0.23    0.000020          10         2         2 rt_sigreturn
  0.22    0.000019           9         2           chdir
  0.22    0.000019           9         2           getgroups
  0.15    0.000013           6         2           clone
  0.00    0.000000           0         6           brk
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         1         1 access
  0.00    0.000000           0         2           dup2
  0.00    0.000000           0         1           getpid
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           kill
  0.00    0.000000           0         1           getcwd
  0.00    0.000000           0         2           umask
  0.00    0.000000           0         1           geteuid
  0.00    0.000000           0         1           getegid
  0.00    0.000000           0         2         1 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.008647                   365        25 total
```

删除 10 个文件，看起来 rsync 的系统调用次数比 rm 要多，我决定加大文件数量测试

第二次测试，删除一万个文件

```
for i in $(seq 10000); do touch tmp_$i;done  
strace -c rm * -rf                                
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 70.08    0.141015          14     10000           unlinkat
 29.67    0.059692           5     10000           newfstatat
  0.08    0.000158           6        24           brk
  0.04    0.000083          10         8           mmap
  0.02    0.000048          12         4           mprotect
  0.02    0.000036          12         3           openat
  0.02    0.000035           5         6           close
  0.01    0.000025           6         4           read
  0.01    0.000024          24         1           munmap
  0.01    0.000023           5         4         1 lseek
  0.01    0.000019           6         3           fstat
  0.01    0.000013           6         2         1 arch_prctl
  0.01    0.000011          11         1         1 access
  0.00    0.000010          10         1           execve
  0.00    0.000009           9         1           ioctl
  0.00    0.000008           8         1           lstat
------ ----------- ----------- --------- --------- ----------------
100.00    0.201209                 20063         3 total
```

```
strace -c -f rsync --delete -a -H ../blank_dir/ ./
strace: Process 16414 attached
strace: Process 16415 attached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 61.75    0.386408       42934         9         5 wait4
 19.18    0.120021          12     10000           unlink
 15.97    0.099912           9     10004           lstat
  2.21    0.013827        1063        13           getdents64
  0.14    0.000860          20        41           mmap
  0.12    0.000755          32        23         4 openat
  0.10    0.000604          11        54           read
  0.08    0.000479          12        37           close
  0.07    0.000422          35        12           munmap
  0.05    0.000338          30        11           mprotect
  0.05    0.000284          35         8         8 connect
  0.04    0.000251          13        19           fstat
  0.04    0.000239          19        12           rt_sigaction
  0.04    0.000236           5        40         1 select
  0.03    0.000192          24         8           socket
  0.03    0.000157           7        22           write
  0.02    0.000156          26         6           brk
  0.01    0.000084           4        17           fcntl
  0.01    0.000079          39         2           utimensat
  0.01    0.000078          11         7           lseek
  0.01    0.000060          20         3           socketpair
  0.01    0.000041          41         1           getcwd
  0.01    0.000036          18         2           umask
  0.00    0.000027          27         1           rt_sigprocmask
  0.00    0.000026          13         2           chdir
  0.00    0.000025          25         1         1 access
  0.00    0.000022          11         2         1 stat
  0.00    0.000022          22         1           geteuid
  0.00    0.000021          10         2         1 arch_prctl
  0.00    0.000020          20         1           execve
  0.00    0.000019          19         1           getegid
  0.00    0.000014           7         2         2 nanosleep
  0.00    0.000010           5         2           clone
  0.00    0.000009           4         2         2 rt_sigreturn
  0.00    0.000000           0         2           dup2
  0.00    0.000000           0         1           getpid
  0.00    0.000000           0         1           kill
  0.00    0.000000           0         2           getgroups
------ ----------- ----------- --------- --------- ----------------
100.00    0.625734                 20374        25 total
```
