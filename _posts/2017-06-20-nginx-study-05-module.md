---
layout: post
title: nginx源码学习笔记-模块
date:   2017-06-20 20:30:00 +0800
category: nginx
tag: [nginx]
---

* content
{:toc}


## 前言

Nginx的模块非常多，可以认为所有代码都是以模块的形式组织的，另外，Nginx的模块不想Apache或Lighttpd那样使用`.so`动态库，而是所有模块源文件编译到一个二进制执行文件中，所以选择不同的模块功能，需要重新编译Nginx

## 模块的大致类别

- `handlers` 协同完成客户端请求的处理，产生响应数据，例如：
    + `ngx_http_rewrite_module` 处理客户端请求地址的重写
    + `ngx_http_static_module` http的静态资源的请求
    + `ngx_http_log_module` 负责记录请求访问日志
    + ....

- `filters` 对`handlers`产生的响应数据做各种过滤处理(增／删／改), 例如：
    + `ngx_http_not_modified_filter_module` 通过时间戳判断前后两次请求的响应数据没有发生实质变化，直接响应`304 Not Modified`，让客户端使用本地缓存

- `upstream` Nginx可以充当反向代理`Reverse Proxy`的角色, 例如
    + `ngx_http_proxy_module`，对客户端的请求负责转发，并将真实服务器响应数据回传给客户端

- `load-balance` Nginx充当中间代理角色，在真实服务端多于一个时，如何选择对应的服务端处理，如负载均衡
    + `ngx_http_upstream_ip_hash_module`

## 模块的数据结构

### 模块的结构体

可见`ngx_conf_file.h`

```c
struct ngx_module_s {
    ngx_uint_t            ctx_index;    // 当前模块在同类模块中的序号
    ngx_uint_t            index;        // 当前模块在所有模块中的序号

    /*...*/

    ngx_uint_t            version;      // 当前模块的版本号

    void                 *ctx;          // 当前模块特有的数据
    ngx_command_t        *commands;     // 指向当前模块配置项解析数组
    ngx_uint_t            type;         // 模块类型

    /*...*/
};

```

- `ngx_module_s`的`type`与`ctx`有对应关系，而且`type`目前只有5个可能的值
    + type = `NGX_CORE_MODULE`, ctx = `ngx_core_module_t`
    + type = `NGX_EVENT_MODULE`, ctx = `ngx_event_module_t`
    + type = `NGX_CONF_MODULE`, ctx = `NULL`
    + type = `NGX_HTTP_MODULE`, ctx = `ngx_http_module_t`
    + type = `NGX_MAIL_MODULE`, ctx = `ngx_mail_module_t`

### 模块的配置项的数据结构

```c
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

- 解析配置项相关的函数如下，更多见`ngx_conf_file.c`
    + `ngx_conf_parse()` 一开始在`ngx_cycle.c`中`ngx_init_cycle()`函数中第一次被调用，传入配置文件地址的字符串指针，它是一个间接递归函数，遇到特殊配置指令(include,events,http,server,location 等)，会再次调用ngx_conf_parse()
    + `ngx_conf_read_token()` 简单配置的token指令，一个或多个
    + `ngx_conf_handler()` 会执行`ngx_command_s.set()`函数指针


## handler 模块

。。。待续



## 参考文献

- [深入剖析Nginx][1]

[1]:https://book.douban.com/subject/23759678/