---
layout: post
title: 阅读tinyhttp源码
date:   2017-11-15 20:30:00 +0800
category: http
tag: [net]
---


* content
{:toc}


## 前言

tinyhttp是500行的实现http服务器的c语言代码，阅读一下挺简单的，感觉都可以背下来了，本日记记录阅读过程中的资料查找，思路等等

## 常用的函数

### 转换ip地址

```c

    typedef uint32_t in_addr_t;

    struct in_addr {
        in_addr_t s_addr;
    };
    in_addr_t inet_addr(const char *cp);

    int inet_pton(int af, const char *src, void *dst)

```

#### 名词

- 网络字节序与主机字节序，网络字节序一般是大端，主机字节序视操作系统决定
- 

#### 函数列表

- `inet_addr()` 输入：ip地址的字符串，返回：若字符串有效则将字符串转换为32位二进制`网络字节序`的IPV4地址，否则为INADDR_NONE。只支持ipv4
- `inet_pton()` 这个函数转换字符串到网络地址，第一个参数af是地址簇，第二个参数`src是来源地址，第三个参数`dst接收转换后的数据。

