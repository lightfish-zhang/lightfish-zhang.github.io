---
layout: post
title: nginx源码学习笔记-gdb的使用
date:   2017-06-02 20:00:00 +0800
category: nginx 
tag: [gdb]
---

* content
{:toc}

## 前言

本篇学习笔记，实质是笔者对[深入剖析Nginx][1]的读后感与调试源码的实践记录

## 利用gdb调试

### 修改编译配置 
- 使`gcc`带上`-g -O0`，以下方法任选一
    + make CFLAGS="-g -O0"
    + auto/configure --with--cc-opt='-g -O0'
- 宏，如果不事先实现处理宏，在gdb调试中，将无法查看这些宏的定义以及展开形式
    + `-g`改为`-ggdb3`

### gdb跟踪进程

- 找到需要调试的进程
    + `ps -efH|grep nginx|grep -v grep`找到需要调试的进程，可见的进程有`nginx master`和`nginx worker`，`nignx`源码实现给进程加上`title`
    + 对某一进程调试，`gdb -p $pid`, 或者在`gdb`内命令行执行`attach $pid`

- nginx`fork`子进程，如何调试
    + 修改配置项，使`ngixn`处理请求交付给`worker`的进程设为1，修改`nginx.conf`的`worker_processes 1;`
    + nginx以`daemon`运行，方式不是屏蔽`signal`而是制造孤儿进程，也就是创建子进程后把父进程`exit(0)`，子进程被`init`收养而成为`daemon`
    + 在调试`daemon`程序，可以在`gdb`内设置`set follow-fork-mode child`, `gdb`是默认跟踪fork()之后的父进程，需要做此设置。
    + 但是`nginx`创建工作进程也是使用`fork`,使用上一条方法会有些麻烦，如果需要调试`master`进程，可以修改`nginx.conf`的`daemon off;`
    + 灵活变通，切换`follow-fork-mode`标记，在多处恰当的地方下断点，执行到预想的代码.
    + 由前几条方法，我们可以灵活地调试`master`与`worker`进程，当然`nginx.conf`还有一个配置`master_process on\off;`，可以设置监控进程逻辑和工作逻辑共同工作在一个进程。

- nginx指定配置

+ shell执行

```sh
# 在编译过的源码文件夹下执行，加`sudo`是因为nginx需要在系统绑定端口以及需要读写/usr/local/nginx/log等文件
sudo gdb --args ./objs/nginx -c `pwd`/conf/nginx.conf
````

+ 或者在`gdb`内执行

```sh
file ./objs/nginx
set args -c /usr/local/nginx/conf/nginx.conf
r
# or
r -c /usr/local/nginx/conf/nginx.conf
```

### gdb的`watch`命令使用

- 打断点`b filename.c:1234`，在`filename.c`文件的1234行
- `r`, `run`执行
- `p abc`, `print`输出变量abc的值，如`0x1234`
- `watch 0x1234`, 当地址的值`0x1234`被修改，执行过程会被停止，且gdb抓取变化前后的值

### nginx对gdb的支持

- `nginx.conf`配置`debug_points stop/abort;`
- 如果nginx遇到严重错误，会调用`ngx_debug_point()`函数，如果`debug_points stop;`，nginx进程会进入暂停状态（`ts`/`TASK_STOPPED`），此时可使用gdb进入该进程查看上下文信息
- 命令`ps -aux|grep nginx|grep -v grep`可看出nginx的状态，或者`ps -eo pid,stat,command|grep nginx|grep -v grep`
- 如果`debug_points abort;`，操作系统有所处理，可以通过core文件调试

```
# 如果修改源码，主动调用ngx_debug_point()
ulimit -c unlimited
./nginx
gdb ./core -q
# 查看上下文
(gdb) bt\up\list
```

## 参考文献

- [深入剖析Nginx][1]

[1]:https://book.douban.com/subject/23759678/