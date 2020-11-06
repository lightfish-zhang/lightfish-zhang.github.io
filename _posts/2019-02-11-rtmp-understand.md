---
layout: post
title: 理解RTMP协议——简单认识
date:   2019-02-11 20:30:00 +0800
category: RTMP
tag: [protocol]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/video-icon.jpeg
icon: book
---

* content
{:toc}


## 前言

直播行业的兴起，带动了音视频相关技术的发展，本文介绍 RTMP 协议，让人快速理解它。看下面一张视频直播的大体架构图，找找 RTMP 的位置，明白 RTMP 扮演的角色与重要性

![](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-application.jpg)

在上面，RTMP 在视频直播场景的架构中，担任了重要的"血管"般的角色

## 简单介绍

RTMP(Real Time Messaging Protocol)实时消息传送协议是Adobe Systems公司为Flash播放器和服务器之间音频、视频和数据传输开发的私有协议。

RTMP是一个应用层协议，有多路复用的特点，传输内容有视频、音频、控制命令

RTMP 在音视频相关的协议中，它的突出特点是：连接可靠、低延时

## RTMP 基于 TCP

RTMP 是基于TCP的二进制协议，（顺便一提，http为广泛应用的明文协议之一）

![rtmp-base-on-tcp](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/rtmp-base-on-tcp.webp)

RTMP 默认端口 1935

- 基于 TCP 的好处
    + 提高了RTMP的可靠性

- 基于 TCP 的弊端

如果网络条件差时，由于TCP存在重传的机制，所以导致RTMP存在累计延时。当网络状态差时，服务器会将包缓存起来，导致累积的延迟，待网络状况好了，就一起发给客户端。

### 解决 RTMP 累计延时的弊端

解决 RTMP 累计延时的弊端的方法:

- 数据下发的角度，服务端上设置对客户端的缓冲区空间大小，一旦累计的缓冲超过限制，服务端就断开连接，迫使客户端在恢复网络时发起重新连接的请求。
- 流媒体推流(上传)的角度，上传流媒体的发布方既可以是服务器也可以是客户端App，发布方发现当前队列中未处理的的视频和音频帧数累计达到一定数目(如50帧)，则清空该队列，直接处理最新的实时数据（严格意义上，需要保留关键帧，清除预测帧）。


## RTMP 的 HTTP 变种与防火墙

rtmp 有三个变种：

- 工作在TCP之上的明文协议，使用端口1935
- RTMPT封装在HTTP请求之中，可穿越防火墙
- RTMPS类似RTMPT，但使用的是HTTPS连接

穿越防火墙的意思是，可能出于安全考虑，互联网中某一些网络（比如小区、校园网）的防火墙限制了http/https以外的协议访问，只允许访问外网ip的 80 端口与 443 端口，或者还允许其他协议，而明文协议的 rtmp 默认端口 1935 不在防火墙开放访问的端口中，无法建立连接。出于现实考虑，使用 http/https 封装 rtmp 协议，增强兼容性。


## RTMP Server

- FMS Wowza (Flash Media Server)，商业产品，Adobe公司的产品，license非常昂贵。wowza最突出的特定是多终端适应性，这个在如今多媒体融合的网络环境下有很强的实用意义。

以下开源项目：

- [red5](https://github.com/Red5/red5-server), java，比较出名
- [crtmpserver](https://github.com/j0sh/crtmpserver)，C++
- [nginx-rtmp-module](https://github.com/arut/nginx-rtmp-module)，nginx的轻量模块，一般后端工程师比较熟悉 nginx，上手方便



## Reference

- [Adobe’s Real Time Messaging Protocol](https://www.adobe.com/content/dam/acom/en/devnet/rtmp/pdf/rtmp_specification_1.0.pdf)
- [RTMP协议入门](https://www.jianshu.com/p/715f37b1202f)
- [网宿超大规模直播运营优化之旅](https://toutiao.io/posts/8u3n7d/preview)
- [TCP连接那些事](https://coolshell.cn/articles/11564.html)
- [RTMP H5直播流技术解析](https://zhuanlan.zhihu.com/p/51509123)
- [RTMP协议的message](https://www.zybuluo.com/sheepbao/note/498380)