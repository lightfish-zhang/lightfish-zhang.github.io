---
layout: post
title: nginx源码学习笔记-日志追踪
date:   2017-06-06 20:00:00 +0800
category: nginx 
tag: [nginx]
---

* content
{:toc}


## 前言

本篇学习笔记，实质是笔者对[深入剖析Nginx][1]的读后感与调试源码的实践记录

### 编译时，加上日志代码

- 修改`auto/option`的宏定义

```
# 设置NGX_DEBUG为YES,　如果没设置，很多日志逻辑代码将在make编译时直接跳过。比如对单链接的debug_connection调试指令、分模块日志调试debug_http等
NGX_DEBUG=YES
```

- 执行`configure`时加上`--with-debug`，实质和上一条方法一样.

### 修改配置文件`nginx.conf`

- 默认 error_log  logs/error.log  error; 在`objs/ngx_auto_config.h`有定义
- 日志级别有 从低到高, `debug\info\notice\warn\error\crit\alert\emerg`
- 或者按模块输出日志，　`debug_core\debug_alloc\debug_mutex\debug_event\debug_http\debug_imap`
- 如 `error_log  logs/error.log  debug_http;` nginx会输出info到emerg日志，而debug日志只输出http相关
- 在nginx的配置文件中，error_log有`作用域`，在不同的配置文件或者不同块中，可以定义不同的错误日志文件路径和级别，这是信息切割过滤的手段

### 针对特定链接打日志

添加配置

```
events {
    debug_connection 192.168.1.1;
    debug_connection 192.168.10.0/24;
}
```

## 参考文献

- [深入剖析Nginx][1]

[1]:https://book.douban.com/subject/23759678/