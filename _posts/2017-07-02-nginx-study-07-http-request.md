---
layout: post
title: nginx源码学习笔记-http请求与处理
date:   2017-07-02 20:30:00 +0800
category: nginx
tag: [http]
---

* content
{:toc}


## 配置文件解析

配置文件

```
http{
    server{
        listen 80;
        server_name www.xxx.com;
    }
}
```

在`ngx_http_core_module.c`对应的配置文件解析的结构体

```c
{ ngx_string("http"),
    NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
    ngx_http_block,
    0,
    0,
    NULL },

    ngx_null_command
};
{ ngx_string("listen"),
    NGX_HTTP_SRV_CONF|NGX_CONF_1MORE, // 在server块内|一个以上的token
    ngx_http_core_listen,             // 配置指令处理回调函数
    NGX_HTTP_SRV_CONF_OFFSET,         // 主要由NGX_HTTP_MODULE类型模块所使用，其指定当前配置项所在的大致位置
    0,
    NULL },
{ ngx_string("server_name"),
    NGX_HTTP_SRV_CONF|NGX_CONF_1MORE,
    ngx_http_core_server_name,
    NGX_HTTP_SRV_CONF_OFFSET,
    0,
    NULL },
```


### 配置项解析过程

- 以上配置的回调函数在`ngx_http.c`
- `listen`配置项的解析之路: `ngx_http_core_listen()`->`ngx_http_add_listen()`->`ngx_http_add_address()`->`ngx_http_add_server()`
- 当Nginx的http配置块解析完毕，在`ngx_http_block()`最后会调用`ngx_http_optimize_servers()`创建对应的监听套接口的"结构体"，未真正创建套接口，结构体`ngx_listening_t`放在全局变量`cycle->listening`
- 真正创建套接口以及套接口选项操作在`ngx_connection.c`
    + 函数是`ngx_open_listening_sockets()`和`ngx_configure_listening_sockets()`
    + 上两个函数在Nginx主进程初始化`ngx_init_cycle()`结尾处调用
    + 创建套接字的`ngx_socket()`，根据宏定义看，unix环境下调用`socket()`, win32环境下调用`WSASocket()`
    + 以unix环境下，创建套接字以及套接口选项设置的套路是: `socket()`->`setsocketopt()`->`bind()`->`listen()`


## work进程监听socket

- 关键问题是避免竞争与负载均衡
- 创建套接字，在Nginx主进程做好，在fork之前，这样工作进程都会继承已经初始化好的监听套接口
    + 遍历`cycle->listening`数组，根据用户配置而赋予不同的默认特性，比如收包/发包缓存区大小，
    + `cycle->listening`数组的每一个ls元素，其fd就是一个可用的监听套接口描述符

- worker进程初始化和调用过程
    + 初始化函数`ngx_worker_process_cycle()`->`ngx_worker_process_init()`->`ngx_event_process_init()`
    + 在`ngx_event_process_init()`中，使用到了全局变量`ngx_use_accept_mutex`，当Nginx开启多进程负载均衡，该变量赋值1
    + 在`ngx_event_process_init()`中，对每一个监听套接口创建对于的`connection`连接对象，且事件的回调函数设置为`ngx_event_accept()`
```c
/* ngx_event_process_init() */
    for (i = 0; i < cycle->listening.nelts; i++) {
        c = ngx_get_connection(ls[i].fd, cycle->log);
        /* ... */
        rev->handler = ngx_event_accept;
    }
```

    + 工作进程的主要执行体`ngx_process_cycle.c`的`ngx_worker_process_cycle()`中主要是一个无限for循环，最重要的函数是`ngx_process_events_and_timers()`

### worker进程获取socket的客户端连接

```c
/* ngx_event.c  ngx_process_events_and_timers() 部分代码 */
ngx_process_events_and_timers(ngx_cycle_t *cycle){
    /*...*/
    if (ngx_use_accept_mutex) {
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }

            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;

            } else {
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }
    delta = ngx_current_msec;
    (void) ngx_process_events(cycle, timer, flags);
    /*...*/
    if (ngx_posted_accept_events) {
        ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    }
    /*...*/
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }
    /*...*/
    if (ngx_posted_events) {
        if (ngx_threaded) {
            ngx_wakeup_worker_thread(cycle);

        } else {
            ngx_event_process_posted(cycle, &ngx_posted_events);
        }
    }
    /*...*/
}

```

```c
/* ngx_event_accept.c  ngx_event_accept() 相关代码 */
ngx_accept_disabled = ngx_cycle->connection_n / 8
                        - ngx_cycle->free_connection_n;
```

```c
/* ngx_epoll_module.c  传参flag带有NGX_POST_EVENTS调用的代码 */
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags){
    if (flags & NGX_POST_EVENTS) {
                    ngx_locked_post_event(wev, &ngx_posted_events);
}
```

- 套接口
    + 监听套接口`listen socket`(如主机的80端口)
    + 连接套接口`connection socket`(如客户端对80端口的一个连接)

- 多进程对一个监听套接口(listen)的可读事件的抢占
    + worker进程抢占`accept_mutex`锁
    + 互斥锁`ngx_trylock_accept_mutex()`等相关函数，通过Nginx的共享内存机制实现
    + 抢到锁的进程，会给`flag`打上`NGX_POST_EVENTS`标记，将大部分事件延迟到释放锁之后再去处理，把锁尽快释放，缩短自身持有锁的时间
    + `ngx_locked_post_event()`维护一个链表，缓存了所有事件；新建连接缓存事件`ngx_posted_accept_events`全部被处理后就必须释放持有锁，释放后执行该连接套接口connection上的`ngx_posted_events`事件
    + 没抢到锁的进程，会把事件监控机制阻塞点如`epoll_wait()`的超时时间`timer`限制为较短的范围，默认500ms, 配置项`accept_mutex_delay`，这样，频繁地从阻塞跳出来去争抢互斥锁

- 抢占`accept_mutex`后，执行的堆栈：`ngx_process_events()`->`ngx_epoll_process_events()`->`epoll_wait()`->`rev->handler(rev)`(上文提到的`ngx_event_accept()`)

- `ngx_event_accept()`函数分析
    + 判断`ngx_accept_disabled`值是否大于0，判断当前进程是否过载
    + `ngx_cycle->connection_n` 表示一个工作进程的最大承受连接数，配置项是`worker_connections`

- 监听套接口的可读事件时设置以水平触发的
    + 每一个worker进程，只从监听套接口读取一个连接套接口，而epoll_wait()有可能返回多个连接套接口，采用水平触发才能保证其他连接套接口可以被其他worker进程读取
    + 如果采用边缘触发，就会漏掉其他客户端连接
    + 配置项`multi_accept`默认off, 也就是worker进程默认获取监听套接口上的可读事件后，只接受一个客户端请求`accept()`，这样工作进程的负载将更加均衡，如果`multi_accept`设为`off`,工作进程会在`do{}while`中`accept`此次`epoll_wait()`的所有客户端连接


## http请求处理的过程

### 请求数据的格式

- `RFC 2616`文档对http请求响应消息格式描述, 请求消息格式`BNF`巴克斯诺尔格式如下(https://tools.ietf.org/html/rfc2616#page-35)

```
        Request       = Request-Line              ; Section 5.1
                        *(( general-header        ; Section 4.5
                         | request-header         ; Section 5.3
                         | entity-header ) CRLF)  ; Section 7.1
                        CRLF
                        [ message-body ]          ; Section 4.3
```

```
      Request-Line   = Method SP Request-URI SP HTTP-Version CRLF
```

```
       Method         = "OPTIONS"                ; Section 9.2
                      | "GET"                    ; Section 9.3
                      | "HEAD"                   ; Section 9.4
                      | "POST"                   ; Section 9.5
                      | "PUT"                    ; Section 9.6
                      | "DELETE"                 ; Section 9.7
                      | "TRACE"                  ; Section 9.8
                      | "CONNECT"                ; Section 9.9
                      | extension-method
       extension-method = token
```

```
    Request-URI    = "*" | absoluteURI | abs_path | authority
```

- `CRLF` = '\r\n'

### 处理请求头

- 第一步，读取`Request-Line`数据，通过`ngx_http_read_request_header()`将数据缓存到`r->header_in`，数据可能分多次到达，所以存在数据的移动操作
- 第二步，解析`Request-Line`数据，通过`ngx_http_parse_request_line()`解析数据，根据上文的结构
- 第三步，存储解析结果，设置相关值。`ngx_http_request_t`对象`r`的相关字段，如`uri`(/path/xxx),`method_name`(GET),`http_protocol`(HTTP/1.0)
- 上面执行成功后，以为初步判断这是一个合法的http客户端请求，接下来就解析其他请求头，`general-header`,`request-header`,`entity-header`等

```
       general-header = Cache-Control            ; Section 14.9
                      | Connection               ; Section 14.10
                      | Date                     ; Section 14.18
                      | Pragma                   ; Section 14.32
                      | Trailer                  ; Section 14.40
                      | Transfer-Encoding        ; Section 14.41
                      | Upgrade                  ; Section 14.42
                      | Via                      ; Section 14.45
                      | Warning                  ; Section 14.46

       entity-header  = Allow                    ; Section 14.7
                      | Content-Encoding         ; Section 14.11
                      | Content-Language         ; Section 14.12
                      | Content-Length           ; Section 14.13
                      | Content-Location         ; Section 14.14
                      | Content-MD5              ; Section 14.15
                      | Content-Range            ; Section 14.16
                      | Content-Type             ; Section 14.17
                      | Expires                  ; Section 14.21
                      | Last-Modified            ; Section 14.29
                      | extension-header

       extension-header = message-header
```

### 数据响应

#### GET静态资源的响应

- 简单的GET请求最终会被`ngx_http_static_module`模块的`ngx_http_static_handler()`函数处理，结合location配置的根目录得到磁盘文件的路径
- 文件属性，相关的响应头，比如`Content-Length`,`Last_Modified`，如果请求带上了时间戳，那么Nginx就可能直接返回304状态码
- 具体代码不做分析了...

#### 子请求

- 子请求是Nginx所特有的设计，主要是为了提高Nginx内部对单个客户端请求处理的并发能力，进一步提高效率, 举个例子:

```
location / {
    root html;
    index index.html;
    add_before_body /before.html;
    add after_body /after.html;
}
```

- 以上的配置会让Nginx在一次请求中并行发起对index.html, before.html, after.html的子请求

### 连接关闭

- 回收资源

#### keepalive 机制

- HTTP/1.1协议，标准要求连接默认被保持，所以请求头`Connection: Keep-Alive;`不再有意义，而请求头`Connection: Close;`则可以明确要求不再进行keepalive连接保持
- 配置项`keepalive_timeout`默认75秒，

## 参考文献

- [深入剖析Nginx][1]

[1]:https://book.douban.com/subject/23759678/