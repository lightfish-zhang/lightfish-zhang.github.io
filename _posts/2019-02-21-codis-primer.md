---
layout: post
title: 分析 Codis 3.x 的设计思想
date: 2019-02-21 20:30:00 +0800
category: Redis
tag: [design]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/redis-icon-logo.png
icon: note
---

* content
{:toc}

## 前言

去年公司的业务扩张，原有单机 Redis 服务容量不足以使用，于是需要部署 Redis 集群，根据自身情况，选择 Codis 开源项目做二次开发，本文记录下我对 Codis 设计思想的理解。

## 简单描述

摘抄 [github.com/CodisLabs/codis](https://github.com/CodisLabs/codis) 的简单介绍：

> Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有显著区别, 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。

![codis architecture](https://github.com/CodisLabs/codis/raw/release3.2/doc/pictures/architecture.png)

Codis 是一个优秀的分布式 Redis 方案，我以 QA 的形式来梳理它的设计思想，如：

- 负载均衡与路由方案如何实现？
- 作为内存型数据库，如何保障性能不降的情况下，实现动态扩容？
- 服务解耦设计如何，是否方便部署与维护？
...

另外，假如遇到宕机事故，如何恢复数据，有没有备用方案？

注：以下基于 Codis 3.2 版本进行分析

## Codis 如何实现动态扩容？

细化问题：

- 当系统需要扩容或者缩容时，如何使各个组件的 slots 状态保持一致，避免数据出错？这个问题在很多对数据一致性有要求的分布式系统里都需要解决。

关于这个问题，先了解 Codis 的主要组件：

- Redis Group, 一个 Redis 主从集群，作为存储节点。Codis 对 Redis 二次开发，增加了额外的数据结构，以支持 slot 有关的操作以及数据迁移指令。
- Codis Dashboard，负责 slot 的分配与元数据管理，协调整个 auto rebalance 流程。在一个 Codis 集群中，只有 0 或 1 个 codis-dashboard 服务。
- Codis Proxy，无状态的代理，对外提供 Redis 读写服务
- Codis coordinator，一致性协调服务，可选 zookeeper, etcd, filesystem，golang 代码中提供了接口让人实现其他。

### 动态扩容的关键——数据迁移

为了保持服务在动态扩容中的可用性，需要一边迁移数据一边提供服务，迁移的技巧是 **同步一致，化整为零**

- 在 auto rebalance 中不影响 Redis 集群的性能，系统同时只会对几个 slot 进行迁移，尽量不影响其他 slot 的读写。
- 数据迁移的粒度优化到 key，针对单个 key 进行迁移，大Key若能拆分成小Key分批次异步迁移、并在迁移过程中该Key可读、不可写，只要迁移速度够快，这对业务而言是可以接受的。

### 数据迁移的核心过程

#### 同步一致

在分布式架构中，进行数据迁移，需要保证各个节点的状态一致。Codis 通过多阶段状态机实现，类似分布式事务中的多阶段提交协议。核心流程如下：

参考代码 github.com/CodisLabs/codis/pkg/topom/topom_slots.go

- Dashboard 接收到运维人员下达的迁移指令，更新 coordnator 上的 slot 状态为待迁移（pending）。（slot上的key只读不写？）
- Dashboard 异步定期检查 coordnator 上是否有待迁移状态（pending）的 slot，若有则改为准备中状态（preparing）, Dashboard 将此状态同时分发到所有 Proxy，若有异常Proxy应答失败，则无法进入迁移，状态回退。（代码用了 switch case 语句的技巧）
- 若所有 Proxy 应答成功，则进入准备就绪状态（prepared），Dashboard 将此状态同时分发到所有 Proxy，Proxy 收到此状态后,访问此 slot 中的 Key 的业务请求将被阻塞等待，若有Proxy应答失败，则会立刻回退到上个状态
- 若所有 Proxy 应答成功，则进入迁移中状态（migrating），Dashboard 将此状态同时分发到所有 Proxy，Proxy 收到此状态后,不再阻塞对迁移 slot 中的 Key 访问，若业务请求 Key 属于待迁移哈希，首先会从迁移源Redis中读取数据，写到目的端Redis中去，然后再获取/修改数据返回，这是其中一种迁移方式，被动迁移，Dashboard也会发起主动迁移，直至数据迁移结束

通过多阶段的状态提交和细粒度、ms级别的锁，Codis优雅的解决了迁移过程中的数据一致性。

#### 化整为零

Redis 过去的同步迁移方案存在 数据大的key迁移慢、读写阻塞的问题

列举一个案例，同步迁移一个 1000 万元素的 ZSET，流程细分如下：
- 第一步 Encode，source Redis 需要将执行 rdbSave 将整个 ZSET 序列化成 payload，这一步消耗约 10s
- 第二步 Network，通过网络传输到 target Redis 消耗约 1.5s， target Redis 收到数据后
- 第三步 Decode，执行 rdbLoad 将其反序列化成内存中的数据结构，这消耗约 36s
- 第四步 Remove，最后 target Redis 删除迁移完成的 key 消耗约 6s，整个迁移过程，target Redis 是完全阻塞的，不能提供读写访问

若要提升迁移性能，必须在以上四个流程上面做优化。改为异步迁移，细化对 key 的操作

- 拆分 Encode 过程，对于大key,不再使用 rdbSave 对数据进行 encoding，而是通过指令拆解，redis中的数据结构(list,set,hash,zset)都可以等价的拆分成若干个添加指令，比如含有1000万元素的zset,可以拆分成10万个zadd指令，每个zadd指令添加100个数据
- Network 网络传输步骤，改为异步IO，发送数据不再阻塞
- target Redis 接收的迁移数据不再是 rdb 二进制数据，不用进行反序列化，只需接收添加指令直接更新对应的内存结构，这里使用一些 trick，比如预分配更多的内存避免数据结构增长过快导致频繁申请内存
- 异步 Remove 步骤，异步删除Key，解决同步删除Key耗时问题。通过额外的工作线程异步删除key，不再阻塞redis主线程

以 1000w 的 ZSET 为例，异步迁移方案比同步迁移的耗时少6倍。在迁移过程中，该 key 可读不可写，在 source Redis 处读，迁移完成后，切换到读 target Redis。

异步迁移方案是Codis核心作者spinlock提出的，正式合入Redis 4.2版本。


## Codis 的高可用如何？

分析高可用的方法论？ 待写。。。

## Codis 的运维成本？

主流的 Redis 分布式方案，可以划分为基于Proxy中心节点和无中心节点，实际上大多数人更偏爱基于Proxy中心节点架构设计，运维成本更低、更加可控。



## Reference

- [大规模 codis 集群的治理与实践](https://cloud.tencent.com/developer/article/1006262)