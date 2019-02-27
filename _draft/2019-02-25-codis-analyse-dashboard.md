---
layout: post
title: Codis 3.x 的 dashboard 源码阅读
date: 2019-02-25 20:30:00 +0800
category: Redis
tag: [design]
thumbnail: 
icon: note
---

* content
{:toc}

## 前言

Codis 是非常棒的 Redis 分布式方案，源码量不多，不管是阅读学习还是二次开发，都比较容易，受益良多。

注：源码版本 Codis 3.2

## dashboard 的主要作用

codis-dashboard 是 codis 的集群管理中心，一个 codis 集群只有一个 dashboard，它负责 proxy 与 server 的添加/删除/数据迁移。

## dashboard 的启动



