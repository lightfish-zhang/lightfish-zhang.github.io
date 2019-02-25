---
layout: post
title: nginx源码学习笔记-多进程通信模型
date:   2017-06-14 20:30:00 +0800
category: nginx
tag: [ipc]
---

* content
{:toc}

## 前言

- 学习多进程通信，Nginx是一个很好的学习例子
- 本篇学习笔记，实质是笔者对[深入剖析Nginx][1]的读后感与调试源码的实践记录

## 父子进程通信

### 经典的匿名管道

```c
socketpair(AF_UNIX, SOCK_STREAM, 0, ngx_processes[s].channel)
//and then, fork...
```

- 父进程保留channel[0]，将channel[1]给close
- 子进程保留channel[1]，将channel[0]给close
- close不需要的channel, 一是为了避免读写相互影响，二是不需要的channel也无用，close比较好


## 兄弟进程之间通信

- 父进程广播刚fork的子进程的channel[0]，使子进程之间可以相互通信
- 只要广播给之前fork的其他子进程就可以了
- 刚fork的子进程继承了父进程的文件项表（所有打开的文件描述符），拥有其他进程的channel[0]，无需广播
- 疑惑，子进程继承父进程的fd，这fd是共享的，文件指针偏移量也是共同的，这就导致父子进程对同一个fd操作会导致读写数据出现`穿插`的问题，按道理应该在子进程把共享的`fd`都`dup()`一遍再使用，nginx源码里面没这么做，而且，目前，nginx其实也没有真的用到子进程之间channel通信,也没有子进程通过channel向父进程通信，只有父进程通过channel向子进程通信，大概是nginx的共享内存、信号通信等方式已经满足需要了吧

```c
static void
ngx_pass_open_channel(ngx_cycle_t *cycle, ngx_channel_t *ch)
{
    ngx_int_t  i;
    for (i = 0; i < ngx_last_process; i++) {

        if (i == ngx_process_slot
            || ngx_processes[i].pid == -1
            || ngx_processes[i].channel[0] == -1)
        {
            continue;
        }
    }
    ngx_write_channel(ngx_processes[i].channel[0],
                          ch, sizeof(ngx_channel_t), cycle->log);
}
```

- 到了这一步，思维谨慎的码农肯定想到，每个进程都有各自的文件项表，进程之间的文件描述符是没有直接关系的，不可能直接把进程A的`fd1`的`int`数字给进程B直接使用，本文下面就讲述如何在进程之间传送`fd`


## 进程间传送文件描述符

### 每个进程都有互不相关的文件项表

- 可以用`ls -l /proc/$pid/fd`，查看操作系统中各个进程打开的文件描述符对应的文件，每个进程的`fd`都是`int`类型，但是对应的文件`v-node`是不一样的
- 进程之间传送文件描述符，可以通过`sendmsg()`/`recvmsg()`实现，理所当然，`sendmsg`发送的`fd`与`recvmsg`收到的`fd`是不一样的，但是`fd`都指向同一文件的`v-node`

#### 使用sendmsg/recvmsg发送fd

```
       #include <sys/socket.h>

       ssize_t sendmsg(int socket, const struct msghdr *message, int flags);
       ssize_t recvmsg(int socket, struct msghdr *message, int flags);
```

- `socket`限于unix域协议，发送通道`channel`来自`socketpair(AF_UNIX, SOCK_STREAM, 0, channel)`，也就是同一机器操作系统下的进程之间才可以传送`fd`
- 结构体`msghdr`与`cmsghdr`

```c
 struct msghdr {
    void         *msg_name;       /* optional address */
    socklen_t     msg_namelen;    /* size of address */
    struct iovec *msg_iov;        /* scatter/gather array */
    size_t        msg_iovlen;     /* # elements in msg_iov */
    void         *msg_control;    /* ancillary data, see below */
    size_t        msg_controllen; /* ancillary data buffer len */
    int           msg_flags;      /* flags on received message */
};
struct cmsghdr {
    socklen_t cmsg_len;    /* data byte count, including header */
    int       cmsg_level;  /* originating protocol */
    int       cmsg_type;   /* protocol-specific type */
    /* followed by unsigned char cmsg_data[]; */
};
```

Nginx的代码实现可见源码`ngx_channel.c`文件，本文不贴代码占篇幅了，列出实现传送`fd`的必要点

- `sendmsg`部分，在`ngx_write_channel()`中
    + `msg.msg_control = (caddr_t) &cmsg;`, `msghdr`的`msg_control`指向`cmsghdr`
    + `cmsg.cm.cmsg_type = SCM_RIGHTS;`, 常量`SCM_RIGHTS`表示`传送文件描述符`
    + `ngx_memcpy(CMSG_DATA(&cmsg.cm), &ch->fd, sizeof(int));`, `cmsghdr.cmsg_data`需要分配内存空间，存放文件描述符
- `recvmsg`部分，在`ngx_read_channel()`中
    + 错误处理，例如是传送文件描述符的事件，判断`cmsg.cm.cmsg_len`长度是否是想要的`fd`的`int`长度
    + `ngx_memcpy(&ch->fd, CMSG_DATA(&cmsg.cm), sizeof(int));`复制消息数据(文件描述符)


## 参考文献

- [深入剖析Nginx][1]

[1]:https://book.douban.com/subject/23759678/