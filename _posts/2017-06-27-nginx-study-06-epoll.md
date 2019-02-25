---
layout: post
title: nginx源码学习笔记-epoll模型
date:   2017-06-27 20:30:00 +0800
category: nginx
tag: [epoll]
---

* content
{:toc}

# Nginx的epoll模型

## 前言

epoll模型是Nginx的高性能的基石

## IO多路复用模型

### IO多路复用模型的接口

- 不同平台支持不同的io多路复用模型，如linux的epoll,FreeBSD的kqueue, 对于跨平台开发的Nginx，就需要抽象出统一的接口
- `ngx_event_action_t`接口
    + `init` 初始化
    + `add` 将某描述符的某个事件(可读/可写)添加到多路复用监控里
    + `del` 将某描述符的某个事件(可读/可写)从多路复用监控里删除
    + `enable` 启用对某个指定事件的监控
    + `disable` 禁用对某个指定事件的监控
    + `add_conn` 将指定连接关联的描述符加入到多路复用监控里
    + `del_conn` 将指定连接关联的描述符从多路复用监控里删除
    + `process_changes` 监控的事件发送变化，只有kqueue用到这个接口
    + `process_event` 阻塞等待事件发送，对发送的事件进行逐个处理
    + `done` 回收资源

- 详细实现代码在`ngx_event.c`, 

### epoll模型

```c
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, int timeout, const sigset_t *sigmask);
```

```c
// /usr/include/sys/epoll.h
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	  /* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;
```

- `epoll`同时监控的描述符个数的最大值是`cat /proc/sys/fs/file-max`,也就是进程可打开文件描述符个数限制，笔者机子`Linux 4.9.40-1`, 该数目是1198690
- `epoll_create()` 创建一个epoll的句柄，参数size，告诉内核监听的描述符数目的最大值，请求内核为存储事件分配空间，所以在epoll用完后，需要close(epfd)，回收分配空间
- `epoll_ctl()` 向内核注册、删除或修改事件，按惯例，执行成功返回0，错误返回-1同时设置errno
  + 第一个参数epfd是函数`epoll_create()`的返回值
  + 第二个参数op，动作：注册新的fd`EPOLL_CTL_ADD`，修改已注册的fd的监听事件`EPOLL_CTL_MOD`，删除...`EPOLL_CTL_DEL`
  + 第三个参数fd, 需要监听的描述符
  + 第四个参数event，是`epoll_event`结构体，告诉内核需要监听什么事件

- `epoll_event` 结构体，字段`events`有：
  + `EPOLLIN` 普通数据可读
  + `EPOLLOUT` 发生挂起
  + `EPOLLPRI` 高优先级数据可读
  + `EPOLLERR` 发生错误
  + `EPOLLHUP` 发生挂起
  + `EPOLLET` 将epoll设为边缘触发模式`Edge Triggered`，epoll默认是水平触发`Level Triggered`

- `epoll_wait()` 等待事件发生，执行成功则返回发生事件的描述符的数目，错误返回-1同时设置errno
  + 第一个参数epfd是函数`epoll_create()`的返回值
  + 第二个参数events，从内核接受发送事件的集合
  + 第三个参数maxevents，指定一次获取的最大值，理所当然，其值不得大于`epoll_create(size)`的size
  + 第四个参数timeout指定epoll_wait()函数调用等待时间多久(单位毫秒)，取值-1则无限等待到事件或信号发送, 取值0则立即返回，大于0则阻塞等待(process stat为sleep)，睡眠时间固定为指定的时间

- 水平触发与边缘触发，这个与电路的高低电平的触发方式类似，笔者听硬件开发的同学说，水平触发是一种状态，边缘触发是带有时序的（这样理解好像莫名其妙哈）
  + epoll的水平触发，举例，可读事件返回的就绪的fd，也就是fd有数据，如果没有去读取数据(accept/recv/read), 那么下一次epoll_wait()会再一次返回这个fd
  + epoll的边缘触发，与水平触发比较，没有向fd读取数据，下一次epoll_wait()不会再次返回该fd，除非fd被再一次写入"新的数据"
  + 从两者区别可知，边缘触发仅支持非阻塞non-block的fd, 这样才能保证，就算服务端不读取数据，客户端可以继续往该fd写入数据
  + 对大并发的系统，从性能上，边缘触发比水平触发更有优势，但是对编程的要求也更高

- 在Nginx中，监听套接口`listen socket`(如主机的80端口)，是以水平触发的，而连接套接口`connection socket`(如客户端对80端口的一个连接)，是以边缘触发的，其中原由,笔者放在下一篇

#### 参考例子 

- socket + epoll的代码范例,  https://github.com/lightfish-zhang/linux_practise_c/blob/master/socket/epoll-example.c

## Nginx对epoll模块的封装

- 暴露的事件的相关方法 `ngx_event.h`, 在Nginx其他代码中，都是调用以下宏定义的方法

```c
#define ngx_add_event        ngx_event_actions.add
#define ngx_del_event        ngx_event_actions.del
#define ngx_add_conn         ngx_event_actions.add_conn
#define ngx_del_conn         ngx_event_actions.del_conn
```

```c
typedef struct {
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    ngx_int_t  (*process_changes)(ngx_cycle_t *cycle, ngx_uint_t nowait);
    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                   ngx_uint_t flags);

    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;


extern ngx_event_actions_t   ngx_event_actions;
```

- epoll模块`ngx_epoll_module.c`, 只有支持epoll的系统使用这个文件

```c
ngx_event_module_t  ngx_epoll_module_ctx = {
    &epoll_name,
    ngx_epoll_create_conf,               /* create configuration */
    ngx_epoll_init_conf,                 /* init configuration */

    {
        ngx_epoll_add_event,             /* add an event */
        ngx_epoll_del_event,             /* delete an event */
        ngx_epoll_add_event,             /* enable an event */
        ngx_epoll_del_event,             /* disable an event */
        ngx_epoll_add_connection,        /* add an connection */
        ngx_epoll_del_connection,        /* delete an connection */
        NULL,                            /* process the changes */
        ngx_epoll_process_events,        /* process the events */
        ngx_epoll_init,                  /* init the events */
        ngx_epoll_done,                  /* done the events */
    }
};
```

- 编译前配置，源码文件`auto/os/linux`

```
if [ $ngx_found = yes ]; then
    have=NGX_HAVE_CLEAR_EVENT . auto/have
    CORE_SRCS="$CORE_SRCS $EPOLL_SRCS"
    EVENT_MODULES="$EVENT_MODULES $EPOLL_MODULE"
    EVENT_FOUND=YES
fi
```

## Nginx的事件处理

Nginx里，关注的事件是依附在socket描述符上，在一个流程处理中，在不同阶段，对事件的关注也有所不同

- 新建连接socket，一开始必定监听可读事件
- 读取完所有请求信息并正常处理后，将关注socket的可写事件，从而知道响应信息顺利发送给客户端
- 善加利用epoll_wait()的timeout参数，可以判断超时事件，如响应超时
- 根据当前处理阶段不同，事件处理回调函数也可能不同，比如，新建socket连接，处理客户端请求头与处理客户端请求体的回调函数不一样

### 事件以及回调处理函数

以下是Nginx对http请求响应的正常处理的流程

- 第1步，监听`读`, 回调函数`ngx_http_init_request()`
- 第2步，监听`写`, 回调函数`ngx_http_empty_handler()`(该函数啥都不干，仅打个日志)
- 第3步，监听`读`, 回调函数`ngx_http_process_request_line()`
- 第4步，监听`读`, 回调函数`ngx_http_process_request_headers()`
- 第5步，监听`读`, 回调函数`ngx_http_request_handler()`
- 第6步，监听`写`, 回调函数`ngx_http_request_handler()`
- 第7步，监听`写`, 回调函数`ngx_http_empty_handler()`
- 第8步，监听`读`, 回调函数`ngx_http_keepalive_handler()`

逻辑过程

- 第1步，accept()新建socket，监听可读事件，获取客户端的请求信息
- 第2步，读完客户端信息之后就是进行初始化等准备工作，此时不关注写事件，所以用`ngx_http_empty_handler()`，即啥都不做，仅打印日志
- 第3，4步，对请求头、请求体处理
- 第5步，监听可读事件，回调是`ngx_http_request_handler()`，获取响应数据（资源）
- 第6步，监听可写事件，回调是`ngx_http_request_handler()`，也就是把响应数据全部发回到客户端connfd
- 第7步，监听可写事件，从而知道响应信息顺利发送给客户端
- 第8步，与客户端保持`keepalive`状态
  + 如果客户端有新的数据发到，在`ngx_http_keepalive_handler()`将读到对应的数据，并且调用`ngx_http_init_request`初始化，开始新的请求处理
  + 如果客户端关闭了连接，那么Nginx同样获得一个可读事件，调用`ngx_http_keepalive_handler()`却读不到数据，于是关闭连接、回收资源


## 参考文献

- [深入剖析Nginx][1]

[1]:https://book.douban.com/subject/23759678/