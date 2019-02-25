---
layout: post
title: 当你敲下浏览器url会发生什么
date:   2016-12-01 09:20:00 +0800
category: FontEnd
tag: [font-end, net]
---

* content
{:toc}
 
## 前言
当你敲下浏览器url会发生什么, it民工大概有不同的说法，有人说dns解析，有人说三次握手四次挥手，还有人说浏览器一边解析html富文本一边并发请求数据．
各家角度不一样，说的都是对的．下面，我尝试以尽可能详细又简洁地描述这个过程．

## dns解析
- 从浏览器从dns cache查找ip地址，以chrome browser为例， 可以从`chrome://net-internals/#dns`看到. (tips: `chrome://chrome-urls/`有chrome的秘密)．
- 前者没找到，浏览器(应用程序)，会从操作系统的hosts文件查找，以Linux为例，在`/etc/hosts`文件.
- 前者没找到，从dns服务器获取ip地址，如果Linux配置dns服务器地址，在`/etc/resolv.conf`文件.

### dns的一些事儿

- dns劫持,等有趣的事 [dns的一些事儿](http://)

## tcp三次握手与四次挥手

### 参考资料

- [TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)
- [网络](https://github.com/ElemeFE/node-interview/blob/master/sections/zh-cn/network.md#q-tcp-udp)

### 传输数据，

### `http1.1`与`http2`在传输上的区别
- [`http2`的先进之处在哪儿？]()

## 浏览器识别请求类型

### 请求类型为`doc`，常用html格式

- 边解析边请求数据

### 请求类型为其他文件，下载用


## 拓展

- `websocket`, `long-poll`　的应用场景　(像二维码登录的一分钟等待请求，无需用websocket重器，简单的`long-poll`更符合实际)