---
layout: post
title: nodejs学习笔记——node与libuv
date:   2017-08-23 20:30:00 +0800
category: javascript
tag: [libuv]
---


* content
{:toc}


## 前言

node的最重要的事件循环`event loop`，是通过`libuv`库实现的，这是google开发的C语言库，兼容不同的操作系统

## 事件类型

- Network I/O事件， 根据OS平台不同，分别使用Linux上的epoll，OSX和BSD类OS上的kqueue，SunOS上的event ports以及Windows上的IOCP机制。
- File I/O事件，则使用thread pool，在其中进行阻塞式I/O。利用thread pool的方式实现异步请求处理，在各类OS上都能获得很好的支持。
    + 社区有人提出，在Linux下使用原生的NIO(非阻塞io)替换thread pool的建议 ,测试发现有3%的提升. 考虑到 NIO 对内核版本的依赖，利用thread pool的方式实现异步请求处理，在各类OS上都能获得很好的支持，应该是libuv开发者权衡再三的结果。

- 时间事件， 在`event loop`处理完后，从时间事件的表中获取下一次最近事件的时间间隔`timeout`，将`timeout`设置为等待事件的超时时间`uv_run(loop, timeout);`，周而复始

## 源码分析

### 学习资源

[官方文档](http://docs.libuv.org)
[社区文档](http://nikhilm.github.io/uvbook/)
[libuv常见示例](https://github.com/thlorenz/libuv-dox)

### 从Linux角度去看

- 笔者是Linux爱好者（其实不懂其他操作系统），基于Linux环境下去研究libuv源码
- libuv中对Linux封装的代码主要声明在`libuv/src/unix/linux-core.c`
- 笔者一开始就去找epoll系统调用，发现`libuv`使用了`syscall`系统调用
    + 提高UNIX环境下的用户程序移植性
    + 进程可以跳转到的内核位置叫做`sysem_call`。这个过程检查系统调用号，这个号码告诉内核进程请求哪种服务。然后，它查看系统调用表`sys_call_table`找到所调用的内核函数入口地址。接着，就调用函数，等返回后，做一些系统检查，最后返回到进程

```c
int uv__epoll_create(int size) {
#if defined(__NR_epoll_create)
  return syscall(__NR_epoll_create, size);
#else
  return errno = ENOSYS, -1;
#endif
}

int uv__epoll_ctl(int epfd, int op, int fd, struct uv__epoll_event* events) {
#if defined(__NR_epoll_ctl)
  return syscall(__NR_epoll_ctl, epfd, op, fd, events);
#else
  return errno = ENOSYS, -1;
#endif
}

int uv__epoll_wait(int epfd,
                   struct uv__epoll_event* events,
                   int nevents,
                   int timeout) {
#if defined(__NR_epoll_wait)
  return syscall(__NR_epoll_wait, epfd, events, nevents, timeout);
#else
  return errno = ENOSYS, -1;
#endif
}
```

- 另外，多进程之间分享文件描述符的`sendmsg/resvmsg`

```c
int uv__sendmmsg(int fd,
                 struct uv__mmsghdr* mmsg,
                 unsigned int vlen,
                 unsigned int flags);
int uv__utimesat(int dirfd,
                 const char* path,
                 const struct timespec times[2],
                 int flags);
```

- 获取时间`int clock_gettime(clockid_t clk_id, struct timespec *tp)`
- 修改进程的名字`int prctl(PR_SET_NAME, title)`
- 获取cpu信息，直接通过文件路径获取
```c
static unsigned long read_cpufreq(unsigned int cpunum) {
  unsigned long val;
  char buf[1024];
  FILE* fp;

  snprintf(buf,
           sizeof(buf),
           "/sys/devices/system/cpu/cpu%u/cpufreq/scaling_cur_freq",
           cpunum);

  fp = fopen(buf, "r");
  if (fp == NULL)
    return 0;

  if (fscanf(fp, "%lu", &val) != 1)
    val = 0;

  fclose(fp);

  return val;
}
```
- 获取进程使用物理内存的大小，通过读取文件`/proc/self/stat`获取`procinfo`结构体，里面包含`rss`成员，根据结构体字节对齐的规则，找到`rss`偏移量，读取`rss`。笔者奇怪的是，为什么不直接声明一个结构体指针，通过`procinfo.rss`读取呢？

```c
int uv_resident_set_memory(size_t* rss) {
    //...
}
```

- 获取执行文件的路径`readlink("/proc/self/exe", buffer, n)`
- 多线程程序使用`epoll`时，要注意的信号掩码处理，最好使用`epoll_pwait`替代`epoll_wait`，看下`libuv`对`epoll_wait`的封装，`errno == ENOSYS`表示系统没有这个接口调用

```c
void uv__io_poll(uv_loop_t* loop, int timeout) {
  //....
  for (;;) {
    if (sigmask != 0 && no_epoll_pwait != 0)
      if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        abort();

    if (sigmask != 0 && no_epoll_pwait == 0) {
      nfds = uv__epoll_pwait(loop->backend_fd,
                             events,
                             ARRAY_SIZE(events),
                             timeout,
                             sigmask);
      if (nfds == -1 && errno == ENOSYS)
        no_epoll_pwait = 1;
    } else {
      nfds = uv__epoll_wait(loop->backend_fd,
                            events,
                            ARRAY_SIZE(events),
                            timeout);
      if (nfds == -1 && errno == ENOSYS)
        no_epoll_wait = 1;
    }
```


### 看看libuv是怎么工作的

- 一个比较简单的例子，查看本机文件内容`uvcat`

```c
#include <assert.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <uv.h>

void on_read(uv_fs_t *req);

uv_fs_t open_req;
uv_fs_t read_req;
uv_fs_t write_req;

static char buffer[1024];

static uv_buf_t iov;

void on_write(uv_fs_t *req) {
    if (req->result < 0) {
        fprintf(stderr, "Write error: %s\n", uv_strerror((int)req->result));
    }
    else {
        uv_fs_read(uv_default_loop(), &read_req, open_req.result, &iov, 1, -1, on_read);
    }
}

void on_read(uv_fs_t *req) {
    if (req->result < 0) {
        fprintf(stderr, "Read error: %s\n", uv_strerror(req->result));
    }
    else if (req->result == 0) {
        uv_fs_t close_req;
        // synchronous
        uv_fs_close(uv_default_loop(), &close_req, open_req.result, NULL);
    }
    else if (req->result > 0) {
        iov.len = req->result;
        // 第三个参数是文件描述符，将iov缓存写到标准输出
        uv_fs_write(uv_default_loop(), &write_req, 1, &iov, 1, -1, on_write);
    }
}

void on_open(uv_fs_t *req) {
    // The request passed to the callback is the same as the one the call setup
    // function was passed.
    assert(req == &open_req);
    if (req->result >= 0) { // req->result为open返回的文件描述符
        iov = uv_buf_init(buffer, sizeof(buffer));
        /*
        # 读取文件
        - 使用uv的默认loop
        - 使用uv封装的`uv_fs_s`
        - 目标文件的文件描述符
        - flag
        - mode
        - 回调函数 on_read
        */
        uv_fs_read(uv_default_loop(), &read_req, req->result,
                   &iov, 1, -1, on_read);
    }
    else {
        fprintf(stderr, "error opening file: %s\n", uv_strerror((int)req->result));
    }
}

int main(int argc, char **argv) {
    /*
    # 打开文件
    - 使用uv的默认loop
    - 使用uv封装的`uv_fs_s`，打开文件成功后，成员变量result为文件描述符
    - 路径名
    - flag 只读
    - mode
    - 回调函数 on_open
    */
    uv_fs_open(uv_default_loop(), &open_req, argv[1], O_RDONLY, 0, on_open);

    // uv的执行，当loop中的事件都处理完毕，返回
    uv_run(uv_default_loop(), UV_RUN_DEFAULT);

    uv_fs_req_cleanup(&open_req);
    uv_fs_req_cleanup(&read_req);
    uv_fs_req_cleanup(&write_req);
    return 0;
}

```

- 这个例子中，`uv_run()`执行，其内部是`while`循环，`UV_RUN_DEFAULT`表示事件都处理完后，跳出循环，`uv_run()`返回
- 这个例子的代码不难看，沿着回调函数一直看就行，如果程序回调函数一多，就容易陷入回调地狱，像`node.js`还没有`async/await`或者`yeild`的时候，也容易陷入这个境地

#### 如何加入文件事件

- `uv_default_loop()`第一次执行，初始化工作，主要工作有:
    + `uv__platform_loop_init()`对操作系统的io多路复用的事件监听句柄的初始化
    + `uv_rwlock_init()`与`uv_mutex_init()`对libuv的锁与互斥量初始化

- `uv_fs_open()`的执行过程，源码中, 主要执行两个代码段的宏定义`INIT`,`POST`, `INIT`是初始化一些变量，有意思的是`POST`,派发事件到loop的过程：
- 文件事件，libuv的宏定义`POST`的执行过程：

```c
#define POST                                                                  \
  do {                                                                        \
    if ((cb) != NULL) {                                                       \
      uv__work_submit((loop), &(req)->work_req, uv__fs_work, uv__fs_done);    \
      return 0;                                                               \
    }                                                                         \
    else {                                                                    \
      uv__fs_work(&(req)->work_req);                                          \
      uv__fs_done(&(req)->work_req, 0);                                       \
      return (req)->result;                                                   \
    }                                                                         \
  }                                                                           \
  while (0)
```

- `uv__work_submit()`会调用`uv_once(&once, init_once);`，其实`uv_once()`是`pthread_once()`的封装，使`init_once()`这个函数在一个进程（多个线程）中只执行一次，`init_once()`是libuv的线程池的初始化

```c
static void init_once(void) {
    //...
  for (i = 0; i < nthreads; i++)
    if (uv_thread_create(threads + i, worker, NULL))
      abort();
```

- `uv__work_submit()`接下来会调用`post()`，抢占锁后，把文件事件的加入队列

#### 多线程处理文件事件

- `uv_run()`->`uv__io_poll()`
    - Linux环境下，调用epoll接口，但是libuv的文件事件不使用epoll，而是使用线程池去执行本机文件io任务
    - 除了epoll的调用外，`uv__io_poll()`还处理的队列中数据，已完成任务或者任务之后还有回调需要继续执行等，都处理好
- `libuv/src/threadpool.c`的静态函数`worker()`，就是libuv的线程池的各个子线程执行体
    + 当`wq`队列为空时，线程阻塞等待条件变量`uv_cond_wait()`，Linux是`pthread_cond_wait()`，细节是：释放mutex锁，进入条件变量等待的队列，获取到条件变量的同时占用锁`mutex`.
    + 获取到条件变量，也就是`wq`队列不为空，子线程从队列`QUEUE_HEAD(&wq)`获取任务，然后释放锁`mutex`
    + 获得任务就执行呗，`w->work(w)`，处理完后，将执行完后的数据发送`uv_async_send(&w->loop->wq_async)`，然后子线程继续循环之前的步骤

