---
layout: post
title: 工作案例分析-号码分配服务设计
date: 2019-02-19 20:30:00 +0800
category: Redis
tag: [design]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201902/basic-knowledge-base-about-icon-design-get-started.png
icon: note
---

* content
{:toc}

## 前言

我的工作记录

## 案例背景

需求：设备号分配，这里的号码要求可以被人记住与使用，与数据 ID 不同，更类似手机号码或者QQ号码。随着业务拓展，旧逻辑的6位号码已经不满足需求量，需要扩张到９位号码。而在分配策略上，原有的号码分配策略单纯依赖于 Mysql 的事务性SQL，逻辑存在冲突，在剩余号码不多且并发高时，会出现数据库死锁，逻辑死循环的问题。于是，我部门需要重新设计号码分配策略。

## 设计思路

为了避免mysql死锁与瓶颈，引入缓存层，过程大体这样：预先分配一小段号码段，放入缓存中，分配新号码时，从该号码段中随机抽取一个新号。另外定时检查缓存中的号码段剩余量，不足时新增一批号码段。

我们选用 Redis 作为缓存层，有以下考虑：
- Redis 的数据操作都在内存中单线程进行，执行快，支持原子操作。
- Redis 的集合(无序)类型与其 API 天然符合这次方案所需的 SQL 原语:
    + （SPOP, SADD, SCARD）
    + SADD，将一个或多个 member 元素加入到集合 key 当中
    + SPOP，移除并返回集合中的一个随机元素
    + SCARD，返回集合 key 中元素的数量


大体的逻辑是这样的:

- 9位号码，完整的号码段是 100000000 ~ 999999999，持久存储的元数据有 base, offset； base 从 100000000 开始，offset 暂定为5000，每次预分配 offset 个号码，然后刷新 base 下一次预分配步骤的起始号码。元数据可以存储在 Mysql 单表单行中。
- 每一次预分配的号码段，都同时存入 Redis 缓存与 Mysql 持久化存储，这有服务器宕机重启，数据丢失的安全考虑。
- 抽取号码的步骤： 从 Redis 随机抽取一个号码 - 分配新号码的若干业务 - 最终从 Mysql 上记录的预分配号码段的表删除该号码
- 号码分配成功的标记是，分配出去的号码在其他业务中有一个 Mysql 关系表，表上设置了唯一索引。在分配时，会开启 Mysql 事务，关键的 SQL 操作是: 号码 X 插入关系表 - 删除预分配号码段的表上的号码 X
- 备注，Mysql 的隔离级别是 innodb 的 repeatable read


## 完善逻辑

有了一个大体的设计思路，还需要从实践开发的角度去完善它，罗列下可能发生的问题：

- 每一个步骤运行，程序因不可抗力崩溃，重启程序之后如何恢复；
- 现代的后端工程师在设计方案时，总要考虑到分布式，高可用，那么，定期查询剩余量 + 预分配的步骤，需要在服务集群中竞选 Master 去执行
- "抽取号码-入库成功" 的步骤 与 "预分配" 的步骤是异步进行的，可能会存在冲突的地方

对上面大体逻辑进行补充的要点：

- 运行 daemon 进程（多个），定期竞选 master，进行操作 —— SCARD 统计缓存中的预分配号码剩余量，满足 剩余量 offset/10 时执行预分配操作，往换成 SADD 号码
- 在预分配操作之前，检查 mysql 中存储的预分配的号码是否与缓存中一致同步，如果不同步，则恢复号码

猜想发生冲突的地方:

- "抽取号码-入库成功" 的步骤 与 "预分配" 的步骤是异步进行，检查 mysql 与 Redis 中的号码是否一致，可能存在问题，比如：

如果有业务 Server 从 Redis 中 SPOP 出一个号，恢复 routine 以为 Redis 没有又得塞回去，然后另一个业务 Server 从 Redis 又把相同的号 SPOP 出来，接下来，会有两个进程使用相同的号执行“号码入库”操作，此时，竞争看谁先把号塞到设置唯一索引的表里

## 性能评估

因为使用到了 Mysql 的唯一索引来确保分配出去的号码不会重复，这是无法避免的措施，预估 “给新设备分配号码” 的 QPS < 1000
