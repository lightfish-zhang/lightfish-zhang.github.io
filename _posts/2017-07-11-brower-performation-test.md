---
layout: post
title: 浏览器页面性能测试工具
date:   2017-7-11 20:30:00 +0800
category: brower 
tag: [brower]
---

* content
{:toc}
 
## 前言

最近认识到浏览器性能测试工具，[Lighthouse][1]与[webpagetest][2]

- Lighthouse, 是一个开源的自动化工具，用于改进网络应用的质量。您可以将其作为一个Chrome扩展程序运行，或从命令行运行。您为 Lighthouse 提供一个您要审查的网址，它将针对此页面运行一连串的测试，然后生成一个有关页面性能的报告。(官网描述)
- webpagetest, 就更加强大，生成的报告非常详细，基本上把一个页面请求到页面渲染显示，从`load`-`render`-`js runtime`-`layout`-`composite`，再到模仿用户交互行为，生成的报告可以精细到js函数调用，每一个资源的请求时间，分析的切片是毫秒级的，详细到一言难尽

## 浏览器的页面的渲染过程

这是另外一篇文章, [传送门][3]

## lighthouse的工作原理与验证

- lighthouse的工作原理，参考官方给出的结构图

![Lighthouse Architecture](https://raw.githubusercontent.com/GoogleChrome/lighthouse/master/assets/architecture.jpg)

- 阅读github上的源代码，进行验证与学习，先看目录

    + `chrome-launcher`　chrome浏览器启动器，可以选择启动headless chrome
    + `lighthouse-cli` 命令行工具
    + `lighthouse-core`　核心代码，包含`Driver`,`Gatherers`,`Audits`
    + `lighthouse-extension` Chrome扩展程序的源码
    + `lighthouse-viewer`  生成报告的工具


... 未完待续

## 附录

### 相关资料

[Lighthouse的源码与文档][1]
[Web Page Test的官方网站　][2]

[1]:https://github.com/GoogleChrome/lighthouse/blob/master/docs/architecture.md

[2]:https://www.webpagetest.org/

[3]:http://blog.lightfish.cn/2017/07/17/brower-render/
