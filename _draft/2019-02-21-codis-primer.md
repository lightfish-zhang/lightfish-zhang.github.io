---
layout: post
title: 分布式 Redis 方案 Codis 3.x 的设计思想
date: 2019-02-21 20:30:00 +0800
category: Redis
tag: [design]
thumbnail: 
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


