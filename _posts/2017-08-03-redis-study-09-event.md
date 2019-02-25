---
layout: post
title: redis源码学习笔记-事件与网络通信
date:   2017-08-03 20:30:00 +0800
category: redis 
tag: [event]
---


* content
{:toc}


# 事件与网络通信

## 概念

- Redis服务是一个事件驱动程序，主要处理的事件类型:
    + 文件事件`file event`,也就是网络通信，通过`socket`套接字，多服务端之间，服务端与客户端之间。
    + 时间事件`time event`，redis服务的一些操作(eg, serverCron函数)，需要在给定时间点执行


## 文件事件

### 事件的API

Redis封装的事件的各种函数，由于不同操作系统提供的文件事件托管通知的接口不一样，所以Redis的预编译命令如下

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

#### 事件循环的数据结构

- Redis以Reactor模式开发了网络事件处理器

```c
typedef struct aeEventLoop {

    // 目前已注册的最大描述符
    int maxfd;   /* highest file descriptor currently registered */

    // 目前已追踪的最大描述符
    int setsize; /* max number of file descriptors tracked */

    // 用于生成时间事件 id
    long long timeEventNextId;

    // 最后一次执行时间事件的时间
    time_t lastTime;     /* Used to detect system clock skew */

    // 已注册的文件事件
    aeFileEvent *events; /* Registered events */

    // 已就绪的文件事件
    aeFiredEvent *fired; /* Fired events */

    // 时间事件
    aeTimeEvent *timeEventHead;

    // 事件处理器的开关
    int stop;

    // 多路复用库的私有数据
    void *apidata; /* This is used for polling API specific data */

    // 在处理事件前要执行的函数
    aeBeforeSleepProc *beforesleep;

} aeEventLoop;

```

#### 以Linux的epoll为例子

- 相关代码在`src/ae_epoll.c`
- 创建`epfd`描述符

```c
state->epfd = epoll_create(1024);
```

- 把需要监听的`fd`关联到`epfd`，由于Redis只需要做网络事件监听，所以只需要监听`EPOLLIN`可读事件与`EPOLLOUT`可写事件(如果是非阻塞，表示缓存空间可以继续接收数据)
- 以下代码，添加事件，epoll使用默认的水平触发的方式，因为`ee.events`没有`EPOLLET`

```c
/*
 * 文件事件状态
 */
// 未设置
#define AE_NONE 0
// 可读
#define AE_READABLE 1
// 可写
#define AE_WRITABLE 2


//...

static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    //...
    int op = eventLoop->events[fd].mask == AE_NONE ?
        EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    // 注册事件到 epoll
    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    // ...

    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    // ...
}
```

- 其他的，删除给定事件，释放 epoll 实例和事件槽，获取可执行事件的函数就不展开说了


#### 网络通信的函数

- 相关代码在`src/networking.c`,`src/anet.c`文件
- `anet.c`封装了对`socket`的相关处理函数，如下，都是socket的tcp的服务端套路
    + 端口`port`，`getaddrinfo()`获取ipv4/ipv6的地址`addrinfo`等信息
    + 创建`socket`，域为本地`AF_LOCAL`，类型为`SOCK_STREAM`，也就是tcp方式，`protocol`为0，使用系统默认
    + 将`socketfd`与`addrinfo`绑定`bind()`
    + 将`socket`的文件描述符设置为非阻塞io，使用文件控制函数`fcntl()`
    + 监听`listen()`套接字
    + 将`socketfd`加入`epoll`监听，当可读时，调用`accept`获取客户端连接的描述符`connfd`，把`connfd`加入`epoll`的监听中

```
/*
 * 创建并返回 socket
 */
static int anetCreateSocket(char *err, int domain) {
    int s;
    if ((s = socket(domain, SOCK_STREAM, 0)) == -1) {
        anetSetError(err, "creating socket: %s", strerror(errno));
        return ANET_ERR;
    }

    /* Make sure connection-intensive things like the redis benchmark
     * will be able to close/open sockets a zillion of times */
    if (anetSetReuseAddr(err,s) == ANET_ERR) {
        close(s);
        return ANET_ERR;
    }
    return s;
}

/*
 * 将 fd 设置为非阻塞模式（O_NONBLOCK）
 */
int anetNonBlock(char *err, int fd)
{
    int flags;

    /* Set the socket non-blocking.
     * Note that fcntl(2) for F_GETFL and F_SETFL can't be
     * interrupted by a signal. */
    if ((flags = fcntl(fd, F_GETFL)) == -1) {
        anetSetError(err, "fcntl(F_GETFL): %s", strerror(errno));
        return ANET_ERR;
    }
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        anetSetError(err, "fcntl(F_SETFL,O_NONBLOCK): %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}

/*
 * accept
 */
static int anetGenericAccept(char *err, int s, struct sockaddr *sa, socklen_t *len) {
    int fd;
    while(1) {
        fd = accept(s,sa,len);
        if (fd == -1) {
            if (errno == EINTR)
                continue;
            else {
                anetSetError(err, "accept: %s", strerror(errno));
                return ANET_ERR;
            }
        }
        break;
    }
    return fd;
}

```

- `networking.c`封装了与客户端通信的数据的处理函数，有服务端与客户端的
- 网络通信有一点需要注意的是大小端数据的问题，大小端也就是字节顺序类型
    + Redis的规范是，所有键值对的数据都以小端表示，网络通信的字节也是约定为小端格式
    + 通过预编译命令，判断主机的环境时大端还是小端，修改代码中的`memrev`的函数
    + 当主机的数据格式为大端时，才会使用到大小端转换的函数`src/endianconv.c`
    + 由上面３点可知，当主机为大端格式，且在写入字典类型的数据才会用到大小端转化函数，eg, intset, ziplist

```c
// src/config.c 
/* Byte ordering detection */
#include <sys/types.h> /* This will likely define BYTE_ORDER */

#ifndef BYTE_ORDER
#if defined(linux) || defined(__linux__)
# include <endian.h>
#else

// .....
```

```c
// src/endianconv.h
/* variants of the function doing the actual convertion only if the target
 * host is big endian */
#if (BYTE_ORDER == LITTLE_ENDIAN)
#define memrev16ifbe(p)
#define memrev32ifbe(p)
#define memrev64ifbe(p)
#define intrev16ifbe(v) (v)
#define intrev32ifbe(v) (v)
#define intrev64ifbe(v) (v)
#else
#define memrev16ifbe(p) memrev16(p)
#define memrev32ifbe(p) memrev32(p)
#define memrev64ifbe(p) memrev64(p)
#define intrev16ifbe(v) intrev16(v)
#define intrev32ifbe(v) intrev32(v)
#define intrev64ifbe(v) intrev64(v)
#endif
```

### 文件事件的数据结构

```c
typedef struct aeFileEvent {

    // 监听事件类型掩码，
    // 值可以是 AE_READABLE 或 AE_WRITABLE ，
    // 或者 AE_READABLE | AE_WRITABLE
    int mask; /* one of AE_(READABLE|WRITABLE) */

    // 读事件处理器
    aeFileProc *rfileProc;

    // 写事件处理器
    aeFileProc *wfileProc;

    // 多路复用库的私有数据
    void *clientData;

} aeFileEvent;
```



## 时间事件

### 时间时间的概念

- 时间事件分为两类:
    + 定时事件，让一段程序到达指定时间之后执行一次
    + 周期性事件，让一段程序每隔指定时间就执行一次

- 区分时间事件的类型，定时还是周期
    + 时间处理器返回一个`AE_NOMORE`的整数值，此为定时时间
    + 时间处理器返回一个非`AE_NOMORE`的整数值，则这个整数值为距离下一次执行的时间间隔，单位毫秒，该事件为周期性事件

### 数据结构

```c
/* Time event structure
 *
 * 时间事件结构
 */
typedef struct aeTimeEvent {

    // 时间事件的唯一标识符
    long long id; /* time event identifier. */

    // 事件的到达时间
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */

    // 事件处理函数
    aeTimeProc *timeProc;

    // 事件释放函数
    aeEventFinalizerProc *finalizerProc;

    // 多路复用库的私有数据
    void *clientData;

    // 指向下个时间事件结构，形成链表
    struct aeTimeEvent *next;

} aeTimeEvent;
```

### 处理时间事件的循环

- 时间事件使用无序链表存储，它并不以到达时间排序，所以要把链表的节点全部遍历一遍，判断到达时间是否小于等于当前执行时间
- 执行完`timeProc()`后，判断返回值，若为-1则删除该事件
    + 链表的删除，将当前`* aeTimeEvent`指针改为`* next`
    + 执行事件释放函数`finalizerProc()`
- 若`timeProc()`返回值不为-1，将该时间的到达时间设置为当前执行时间+返回值

## 处理所有事件的主循环

- 这里把时间事件与文件事件放到一个循环中处理，其实是用到`epoll_wait()`第四个参数`timeout`
    + timeout指定epoll_wait()函数调用等待时间多久(单位毫秒)，取值-1则无限等待到事件或信号发送, 取值0则立即返回，大于0则阻塞等待(process stat为sleep)，睡眠时间固定为指定的时间

- 在循环中
    + 先判断是否有时间时间，若没有时间事件，可以把`timeout`设置为-1，只等待文件事件的到来
    + 若有时间事件，遍历时间事件的链表，获得最近的时间事件的等待时间，如果等待时间为0，则`timeout`设置为0，不阻塞进程，等待时间不为0，则设置`timeout`，阻塞进程一段时间
    + Redis的事件循环，是先处理文件事件再处理事件事件

## 参考文献

- [Redis 设计与实现](http://redisbook.com/)
- [带注释的Redis源码](https://github.com/huangz1990/redis-3.0-annotated)