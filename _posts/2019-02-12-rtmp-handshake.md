---
layout: post
title: 理解RTMP协议——握手连接
date:   2019-02-12 20:30:00 +0800
category: RTMP
tag: [protocol]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/Handshake.jpg
icon: book
---

* content
{:toc}


## RTMP 的通信机制

### rtmp 客户端与服务端通信的机制

下图是播放器与 rtmp 服务端通信的例子

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/RTMP-player-client-and-server.png)

另外推荐阅读 [nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module) 源码，比如，握手协议相关代码在 `ngx_rtmp_handshake.c` 文件


## RTMP 的握手连接的例子

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-play-process.webp)

### step 1, tcp 三次握手

TCP 握手过程这里不详细展开，参考这篇文章 [TCP 的那些事儿](https://coolshell.cn/articles/11564.html)


### step 2, RTMP 握手验证

RTMP 握手起到验证的作用，RTMP 握手方式主要分为：简单握手与复杂握手

Adobe 协议中描述的是简单握手，而 Adobe 产品 Flash Media Server 采用复杂握手的方式

#### 简单握手

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-handshake.jpg)

与流程图有点不同，握手的实际流程分三个步骤:

- 第一步， Client -> Server，内容是 C0+C1
- 第二步， Server -> Client，内容是 S0+S1+S2
- 第三步， Client -> Server，内容是 C2

我使用 Wireshark 抓包，验证了过程(我使用 nginx-rtmp-module 做服务器，ffmpeg推流，VLC Media Play播放)

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-wireshark-play-1.png)

报文的解释：

```
        C0 与 S0
    +-+-+-+-+-+-+-+-+
    |  version      |
    +-+-+-+-+-+-+-+-+
```

- C0 与 S0
    + C0：客户端发送其所支持的 RTMP 版本号：3~31。一般都是写 3
    + S1：服务端返回其所支持的版本号。如果没有客户端的版本号，默认返回 3


```
        C1 与 S1
    +-+-+-+-+-+-+-+-+-+-+
    |   time (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   zero (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   random bytes    |
    +-+-+-+-+-+-+-+-+-+-+
    |random bytes(cont) |
    |       ....        |
    +-+-+-+-+-+-+-+-+-+-+
```


- C1 与 S1
    + C1/S1 长度为 1536B。主要目的是确保握手的唯一性。
    + 格式为 time + zero + random
    + time 发送时间戳，长度 4 byte
    + zero 保留值 0，长度 4 byte
    + random 随机值，长度 1528 byte，保证此次握手的唯一性，确定握手的对象


```
        C2 与 S2
    +-+-+-+-+-+-+-+-+-+-+
    |   time (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   time2(4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   random bytes    |
    +-+-+-+-+-+-+-+-+-+-+
    |random bytes(cont) |
    |       ....        |
    +-+-+-+-+-+-+-+-+-+-+
```

- C2 与 S2
    + C2/S2 的长度也是 1536B。相当于就是 S1/C1 的响应值，对应 C1/S1 的 Copy 值，在于字段有点区别
    + time， C2/S2 发送的时间戳，长度 4 byte
    + time2， S1/C1 发送的时间戳，长度 4 byte
    + random，S1/C1 发送的随机数，长度为 1528B

RTMP 是用于网络传输的二进制协议，默认使用 Big-Endian 格式，因为 Big-Endian 格式在抓包时可读性较好


#### 复杂握手

对于复杂握手，不使用 Adobe 产品 FMS 的话，简单了解即可

相对于简单握手，复杂握手增加了严格的验证，主要是 random 字段上进行更细化的划分

1528Bytes随机数的部分平均分成两部分，一部分764Bytes存储public key(公共密钥)，另一部分764Bytes存储digest(密文，32字节)。

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-handshake-complex.webp)

从二进制报文的角度，判断复杂握手的特征是，Version部分不为0，服务器端可根据这个来判断是否简单握手或复杂握手。

握手的了解到这里，下面继续看看握手之后的步骤

### step 3, RTMP connect 

RTMP 有一个重要的概念：Application Instance，直观上，可以体现在 rtmp 的 url 上

我测试的推流例子，用到的 url 为: rtmp://192.168.23.152/live/movie

大家可以注意到，上面 wireshark 对 rtmp 抓包的截图中，握手后紧接一个 client->server 的报文 `connect('live')`，

```
1909	36.398483095	192.168.23.152	192.168.23.152	RTMP	282	connect('live')
```

而这个 `live` 就是这次推流的 Application Instance

### step 4, createStream(创建流) --- 创建逻辑通道

上面 wireshark 对 rtmp 抓包的截图中，有下面两行，第一行是 client->server，第二行是 server->client

```
1933	36.484940956	192.168.23.152	192.168.23.152	RTMP	105	Window Acknowledgement Size 5000000|createStream()
1935	36.485004644	192.168.23.152	192.168.23.152	RTMP	109	_result()
1946	36.528398367	192.168.23.152	192.168.23.152	RTMP	168	getStreamLength()|play('movie')|Set Buffer Length 1,3000ms
```

直观地，rtmp://192.168.23.152/live/movie 的 movie 是这次拉流的 `stream`。

createStream 命令用于创建逻辑通道，该通道用于传输视频、音频、metadata。在服务器的响应报文 _result() 中会返回Stream ID，用于唯一的标示该Stream。

getStreamLength 命令用来获取 `movie` 的流的长度

```
Real Time Messaging Protocol (AMF0 Command getStreamLength())
    RTMP Header
    RTMP Body
        String 'getStreamLength'
        Number 3
        Null
        String 'movie'
Real Time Messaging Protocol (AMF0 Command play('movie'))
    RTMP Header
    RTMP Body
        String 'play'
        Number 4
        Null
        String 'movie'
        Number -2000
```

根据 [Adobe’s Real Time Messaging Protocol](https://www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf) 里对 `_result` 命令的定义，上面 body 中第四个字段 "Number 1" 便是此次的 Stream ID

### step n，anything

一般的 rtmp 连接的流程，都如上所示，后面便是命令与音视频数据的消息，比如：

- 播放器的客户端发送play命令来播放指定流，等待服务端传输音视频数据。
- 推流的客户端会发送 publish 命令，开始上传音视频数据。


### step last, deleteStream(删除流)

根据 [Adobe’s Real Time Messaging Protocol](https://www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)

> NetStream sends the deleteStream command when the NetStream object is getting destroyed.
> 当 NetStream 对象销毁的时候发送删除流命令。 

比如，播放器客户端停止播放，可以删除指定Stream ID的流。服务器不用对这条命令发送响应报文。


## Reference

- [Adobe’s Real Time Messaging Protocol](https://www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
- [RTMP协议入门](https://www.jianshu.com/p/715f37b1202f)
- [网宿超大规模直播运营优化之旅](https://toutiao.io/posts/8u3n7d/preview)
- [TCP连接那些事](https://coolshell.cn/articles/11564.html)
- [RTMP H5直播流技术解析](https://zhuanlan.zhihu.com/p/51509123)
- [RTMP协议的message](https://www.zybuluo.com/sheepbao/note/498380)