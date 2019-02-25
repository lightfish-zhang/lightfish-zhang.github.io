---
layout: post
title: 对计算机名词的中文名称的误解
date:   2017-08-05 20:30:00 +0800
category: computer 
tag: [chinese]
---

* content
{:toc}

# 前言

在编程生涯中，遇到计算机名词的中文名称，遇到一些误解和疑惑，在此篇文章列出来，不定时更新

# 大小端

## 笔者经历

参与过物联网开发的项目，笔者做服务端，跟硬件设备通信时，对接数据的工作上，与嵌入式工程师约定协议与传输格式，因为大端的数据比较容易调试(从低地址到高地址debug出来的数据以四个字节的十六进制格式看，就是发送数据的样子)，大多数情况下，网络传输的字节是大端方式，而x86架构的计算机是以小端方式存储数据。笔者开发时亲自写过从大端数据到x86服务器可读的小端数据的相互转化的代码。但之后过了一段时间，笔者再次回忆大小端的区别的时候，发觉`记不住`大小端的区别了！

## 正确的记忆方法

- 大小端是多字节数据的存储顺序
- 不建议用中文记忆，`端`这个字不好记。用英文记忆比较好，大端的英文即是`big end`, 小端是`little end`，例如`node.js`的`buffer`模块中，对大小端的api方法都有`BE`或`LE`词缀
- 英文扩展记忆
    + `big end` = `big address save the end of data`
    + `little end` = `little address save the end of data`
- 举个例子，一个`int32`类型的数据以十六进制表示`0xabcd`，(十六进制很适合表示字节序列)，`a`是最高字节的值，`d`是最低字节的值，人的习惯认知下，`d`是这个数据十六进制形式`0xabcd`的尾部`end`，用大端方式存储的话，`d`所在的存储地址最大，用小端方式存储的话，`d`所在的存储地址最小。

## 判断当前计算机环境是大端还是小端

- C语言

利用C语言的特性，数组的第0个元素的存储地址是所有元素中最低位的，实现如下

```c
// int 类型4个字节，若转成字符串，str[0]为最低位的内存地址，判断*(str[0])的值
int endFlag = 0x0001;
#define is_big_end() ((*(char *)&endFlag)==0)
#define is_little_end() ((*(char *)&endFlag)==1)
```

# 套接字

- 英文明明是`socket`，浅白的翻译就是`插座`嘛
- `字`是啥，

## C语言的套接字例子，匿名管道

```c
int channel[2];
if (socketpair(AF_UNIX, SOCK_STREAM, 0, ngx_processes[s].channel) == -1){
    //error
}else{
    //channel[0] and channel[1] like fd，you can read, write or close
}
```

...待写