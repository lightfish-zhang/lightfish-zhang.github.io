---
layout: post
title: 简述什么是线程
date:   2017-5-20 20:30:00 +0800
category: thread 
tag: [thread]
---

* content
{:toc}
 

函数是多个输入一个输出的代码段，转成状态机，线程调度就是按一定的优先级交替调度不同的函数（线程），有唤醒（状态恢复）-执行-挂起（保存堆栈状态）