---
layout: post
title: 理解RTMP协议——chunk格式
date:   2019-02-13 20:30:00 +0800
category: RTMP
tag: [protocol]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/Live-Streaming.jpg
icon: book
---

* content
{:toc}


## RTMP 的 message 与 chunk

message 是 RTMP 中的 M，是消息的单位

```
RTMP Message Header
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
    | Message Type| Payload length|  
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
    |       Timestamp             |  
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
    |       Stream ID       |  
    +-+-+-+-+-+-+-+-+-+-+-+-+
```

- Message Type（1 byte），消息类型很重要，它代表了这个消息是什么类型，当写程序的时候需要根据不同的消息，做不同的处理。
- Payload length（3 bytes)，表示负载的长度（big-endian 格式）
- Timestamp (4 bytes)，时间戳（big-endian 格式），超过最大值后会翻转
- Stream ID (3 bytes) ，消息流ID（big-endian 格式），用于区分不同流的消息。
- Message Payload，真实的数据

### message 的类型

消息主要分为三类: 协议控制消息、数据消息、命令消息等

- 协议控制消息
    + Message Type ID = 1~6，主要用于协议内的控制

- 数据消息
    + Message Type ID = 8 9 18
        - 8: Audio 音频数据
        - 9: Video 视频数据
        - 18: Metadata 包括音视频编码、视频宽高等信息。

- 命令消息 Command Message (20, 17)
    + 此类型消息主要有　NetConnection　和　NetStream　两个类，两个类分别有多个函数，该消息的调用，可理解为远程函数调用。

更多的了解见 [Adobe’s Real Time Messaging Protocol](https://www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf) 的 5.4 章节

### Chunk —— 网络中实际发送的内容

rtmp 的 message 会切分为 n 个 chunk，再通过 tcp 协议传输

为什么 rtmp 基于 tcp 协议，tcp 协议已经有化整为零的方式， rtmp 还需要将 message 划分更小的单元 chunk 呢？

分析原因：

- tcp 协议划分一个个 tcp 报文，是为了在网络传输层上保障数据连续性，丢包重发等特性
- rtmp 划分 chunk 消息快，是为了在网络应用层上实现低延迟的特性，防止大的数据块(如视频数据)阻塞小的数据块(如音频数据或控制信息)

#### RTMP 的 chunk 设计思想

在互联网中传输数据时, 消息(Message)会被拆分成更小的单元, 称为消息块(Chunk)。RTMP Chunk Stream 层级允许在Message stream 层次，将大消息切割成小消息，这样可以避免大的低优先级的消息(如视频消息)阻塞小的高优先级的消息(如音频消息或控制消息)。

重复强调，RTMP 是设计用来多路复用的特点，传输内容有视频、音频、控制命令。其中一个非常重要的概念是 `multiplexing` (复用)

不同类型的消息会被分配不同的优先级，当网络传输能力受限时，优先级用来控制消息在网络底层的排队顺序。

> 比如当客户端网络不佳时，流媒体服务器可能会选择丢弃视频消息，以保证音频消息可及时送达客户端。

Chunk 的大小设置，通过 Message Type = 1 的控制消息声明

如果 message length 大于 max chunk size，则需要将这个message切分为多个 chunk 。前面几个 chunk size 必须是 max size，最后一个就是剩余的大小。

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-message-alloc-chunk.webp)

以上图为例，Message大小为 300 bytes，默认Chunk size 为 128 bytes，进行拆分成chunk的过程。

接下来，探寻 chunk 的结构

```
RTMP Chunk Header
    +-------------+----------------+-------------------+-----------+  
    | Basic header|Chunk Msg Header|Extended Time Stamp|Chunk Data |  
    +-------------+----------------+-------------------+-----------+ 
        1 byte     (0,3,7,11 byte)     (0,4 byte)
```

> 设计基于TCP协议的上层协议时，为了防止粘包问题，一般的方法有：1、使用分隔符； 2、在报文header中声明长度。


```
RTMP Chunk Basic Header (1 byte)
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-
    | format | chunk stream id |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-
        2 bits      6 bits      
```

RTMP Chunk Header 的长度不是固定的，由RTMP Chunk Basic Header 前2位二进制决定，有4种类型。chunk stream id 的范围 3～65599,0~2作为保留。

- format = `00`，Chunk Header length = 12 bytes，在一个 chunk 流的开始、时间戳返回的时候必须有这种块，比如：onMetaData, 音视频流刚开始的绝对时间戳，控制消息

```
Basic header + Chunk Msg Header
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | format | chunk stream id | timestamp | message length | msg type id | msg stream id |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        2 bits      6 bits        3 bytes       3 bytes         1 bytes        4 bytes
```

- format = `01`，Chunk Header length = 8 bytes，对于可变大小消息的chunk流，在第一个消息之后的每个消息的第一个块应该使用这个格式


```
Basic header + Chunk Msg Header
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | format | chunk stream id | timestamp | message length | msg type id |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      2 bits       6 bits        3 bytes      3 bytes         1 bytes    
```


- format = `10`，Chunk Header length = 4 bytes，对于固定大小消息的chunk流，在第一个消息之后的每个消息的第一个块应该使用这个格式

```
Basic header + Chunk Msg Header
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
    | format | chunk stream id | timestamp |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
      2 bits       6 bits           3 bytes   
```

- format = `11`，Chunk Header length = 1 bytes，当一个消息被分成多个块,除了第一块以外,所有的块都应使用这种类型

```
Basic header + Chunk Msg Header
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-
    | format | chunk stream id | 
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-
      2 bits       6 bits         
```

注意 `timestamp` 的长度为 3 bytes，当 `timestamp` 被设置为 `0x00ffffff`，chunk header 会加上 `Extended Time Stamp` 字段，否则 `Extended Time Stamp` 不会出现。

- 因为一个流当中可以传输多个Chunk，那么多个Chunk怎么标记同属于一个 Message 的呢？
    + 是通过Chunk Stream ID 区分的，同一个Chunk Stream ID 必然属于同一个 Message
    + 因为TCP的有序，所以同一个 Message 中不同的 Chunk 会先后抵达。


#### 协议控制消息的chunk

Message type 1~6:

- 1，Set Chunk Size 设置块的大小，通知对端用使用新的块大小,共4 bytes

```
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |Basic Header|Message Header|Ex Timestamp|Set chunk size  |  
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- 2，Abort Message 取消消息，用于通知正在等待接收块以完成消息的对等端,丢弃一个块流中已经接收的部分并且取消对该消息的处理，共4 bytes。

```
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |Basic Header|Message Header|Ex Timestamp|Chunk Stream ID |  
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- 3，Acknowledgement 确认消息，客户端或服务端在接收到数量与窗口大小相等的字节后发送确认消息到对方。窗口大小是在没有接收到接收者发送的确认消息之前发送的字节数的最大值。服务端在建立连接之后发送窗口大小。本消息指定序列号。序列号,是到当前时间为止已经接收到的字节数。共4 bytes。

```
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Basic Header|Message Header|Ex Timestamp| Sequence Number|  
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- 4， User Control Message 用户控制消息，客户端或服务端发送本消息通知对方用户的控制事件。本消息承载事件类型和事件数据。消息数据的头两个字节用于标识事件类型。事件类型之后是事件数据。事件数据字段 
是可变长的。

```
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+
    |Basic Header|Message Header|Ex Timestamp| Event Type| Event Data|  
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++-+-+-+
```

- 5，Window Acknowledgement Size 确认窗口大小,客户端或服务端发送本消息来通知对方发送确认(致谢)消息的窗口大小,共4 bytes.

```
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  
    |Basic Header|Message Header|Ex Timestamp| Acknowledgement Window size |     
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  
```

- 6，Set Peer Bandwidth 设置对等端带宽，客户端或服务端发送本消息更新对等端的输出带宽。发送者可以在限制类型字段（1 bytes）把消息标记为硬(0),软(1),或者动态(2)。如果是硬限制对等端必须按提供的带宽发送数据。如果是软限制,对等端可以灵活决定带宽,发送端可以限制带宽?。如果是动态限制,带宽既可以是硬限制也可以是软限制。

```
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  
    |Basic Header|Message Header|Ex Timestamp| Acknowledgement Window size |  
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++  
    |  Limit type  |  
    ++++++++++++++++  
```


#### 音频消息的chunk

Message type = 8，Audio message, 客户端或服务端发送本消息用于发送音频数据。消息类型 8 ,保留为音频消息

以 FLV ACC 的 RTMP Audio Chunk 为例

协议层：

```
        　协议层                封装层
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++ 
    |RTMP Chunk Header | FLV AudioTagHeader | FLV AudioTagBody |     
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

封装层(FLV)：

```
FLV AudioTagHeader

    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++ 
    |SoundFormat | SoundRate | SoundSize | SoundType | AACPacketType |   
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
       4 bits       2 bits      1 bit       1 bit        8 bits
```

#### 视频消息的chunk

Message type = 9， Video message, 客户端或服务端使用本消息向对方发送视频数据。消息类型值 9 ,保留为视频消息。

以 FLV H.264/AVC 的 RTMP Video Chunk 为例

协议层：

```
        　协议层                封装层
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    |RTMP Chunk Header | FLV VideoTagHeader | FLV VideoTagBody |     
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

封装层(FLV)：

```
FLV VideoTagHeader

    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++ 
    |Frame Type | CodecID | AVCPacketType | CompositionTime |    
    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++
       4 bits      4 bits     1 byte          3 bytes
```

编码层：

```
FLV VideoTagBody 

    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    |   Size   | AVCDecoderConfigurationRecord or ( one or more NALUs ) |    
    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
      4 bytes
```

NALU: Network Abstract Layer Unit 网络抽象层单元



## 总结

- RTMP 低延迟的特性，来自 多路复用，消息分块，消息分优先级 的方法


## Reference

- [Adobe’s Real Time Messaging Protocol](https://www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
- [RTMP协议入门](https://www.jianshu.com/p/715f37b1202f)
- [网宿超大规模直播运营优化之旅](https://toutiao.io/posts/8u3n7d/preview)
- [TCP连接那些事](https://coolshell.cn/articles/11564.html)
- [RTMP H5直播流技术解析](https://zhuanlan.zhihu.com/p/51509123)
- [RTMP协议的message](https://www.zybuluo.com/sheepbao/note/498380)