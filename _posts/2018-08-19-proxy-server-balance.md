---
layout: post
title: 架构思考-代理服务与负载均衡算法
date:   2018-08-19 20:30:00 +0800
category: Server
tag: [server]
thumbnail: https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/2018/rocks-balancing.jpg
icon: design
---


* content
{:toc}

## 思考

就是同一个服务要部署多个节点，然后一个请求任务过来，那么这个请求分发给哪个服务呢

工程的优化演进

1、最简单的循环调用

2、按人工标记的权重轮询

3、用 指数加权移动平均 来预测服务延迟曲线，把请求分发给延迟最小的服务节点

EWMA exponentially weighted moving average


https://blog.buoyant.io/2016/03/16/beyond-round-robin-load-balancing-for-latency/

预测服务各节点的延时，并与未完成请求数加权，再选择消耗成本最低的服务节点

因为每台机器的性能不同，简单的 round robin 只是平均的把服务丢到各个服务，或者统计每个节点未完成请求数，选择最小那个。

如果加上EWMA预测延时的算法，就更好做负载均衡了

避免了人工标记权重的主观性（像 nignx 的 weight round robin），由算法来预测各个节点的延时曲线，在不同机器性能与网络带来负载能力差异下，更有针对性的分发请求到各个节点