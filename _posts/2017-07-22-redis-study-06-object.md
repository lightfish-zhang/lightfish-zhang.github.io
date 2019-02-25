---
layout: post
title: redis源码学习笔记-对象
date:   2017-7-22 20:30:00 +0800
category: redis 
tag: [object]
---


* content
{:toc}


## 概念

- Redis用到的主要数据结构有：简单动态字符串`SDS`，双端链表，字典，压缩列表，整数集合等，Redis不是直接使用这些基础的数据结构实现键值对数据库的，而是基于它们创建了一个对象系统
- 对象系统包含有，字符串对象，列表对象，哈希对象，集合对象，有序集合对象，这五种类型的对象
- 对象系统实现了内存回收机制，使用的是引用计数法
- 对象系统实现了对象共享机制，使用的也是引用计数法，使得适当条件下，多个数据库键共享同一个对象，节约内存
- 对象带有访问事件记录信息，若开启`maxmemory`功能，空转时间较长的键会被优先删除

## 源码分析

### 数据结构

相关定义在`src/redis.h`

```cpp
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;
```

- 这个结构体有成员的默认值，是C++的语法，不是Ｃ语言的特性
- 成员`type`有:

```c
/* Object types */
// 对象类型
#define REDIS_STRING 0`
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3
#define REDIS_HASH 4
```

### 内存回收

redis使用引用计数法`reference counting`实现垃圾回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在恰当的时候自动释放对象并进行回收。

- 每个对象的引用计数信息，记录在结构体`redisObject`的成员`refcount`
    + 创建新的对象，引用计数初始化为１
    + 对象被一个新程序使用时，引用计数+1
    + 对象不再被一个程序使用时，引用计数-1
    + 对象引用计数变为0时，对象所占的内存会被释放（记得指针改为NULL）

- 相关的API，在`src/object.c`
    + `void incrRefCount(robj *o)` 为对象的引用计数增一
    + `void decrRefCount(robj *o)` 为对象的引用计数减一,当对象的引用计数降为 0 时，释放对象。
    + `robj *resetRefCount(robj *obj)` 这个函数将对象的引用计数设为 0 ，但并不释放对象。用法是这样的 

```c
// 代码写法方便点
functionThatWillIncrementRefCount(resetRefCount(CreateObject(...)));
```


### 对象共享

- 对象的引用计数属性，除了用于内存回收之外，还带有对象共享的作用
- 但是，共享对象时，redis需要先检查给定的共享对象与创建的目标对象是否完全相同，这里的时间复杂度是个问题
    + 如果共享对象是保存整数值的字符串对象，验证操作的复杂度是O(1)，宏定义`REDIS_SHARED_INTEGERS`限制了共享整数对象的整数最大值
    + 如果共享对象是保存字符串的字符串对象，验证操作的复杂度是O(N)
    + 如果共享对象是包含了多个值（或多个对象）的对象，如列表对象或哈希对象，时间复杂度是O(N^2)

- 因此，Redis的共享对象仅限于整数值的字符串对象，且不大于宏定义`REDIS_SHARED_INTEGERS`



## 参考文献

- [Redis 设计与实现](http://redisbook.com/)
- [带注释的Redis源码](https://github.com/huangz1990/redis-3.0-annotated)