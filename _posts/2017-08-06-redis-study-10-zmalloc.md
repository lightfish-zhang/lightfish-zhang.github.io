---
layout: post
title: redis源码学习笔记-内存分配
date:   2017-08-03 20:30:00 +0800
category: redis 
tag: [memory]
---


* content
{:toc}

# zmalloc内存分配实现

## 概念

Redis对C语言的malloc/free做了小小的封装，维持了一个全局变量`used_memory`已使用内存大小，可以获知内存的使用情况。封装的亮点在于使用了线程的互斥锁，避免该变量被脏读脏写。

## 源码分析

### 变量声明

```c
// 维护一个全局遍历，表示Redis已经使用了多少内存
static size_t used_memory = 0;
// 在多线程时，对内存空间操作时，加锁控制，避免`used_memory`变脏
static int zmalloc_thread_safe = 0;
// 互斥锁
pthread_mutex_t used_memory_mutex = PTHREAD_MUTEX_INITIALIZER;
```

### 函数封装

- `zmalloc()`其实对`malloc()`封装上加了对`used_memory`变量的修改与抢占互斥锁，代码如下　

```c
void *zmalloc(size_t size) {
    void *ptr = malloc(size+PREFIX_SIZE);

    if (!ptr) zmalloc_oom_handler(size);
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

```c
#ifdef HAVE_ATOMIC
//...
#define update_zmalloc_stat_add(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)
#endif

//...

#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \
    } \
} while(0)
```

- `zfree`的实现与`zmalloc`类似，这里不铺开了

## 参考文献

- [Redis 设计与实现](http://redisbook.com/)
- [带注释的Redis源码](https://github.com/huangz1990/redis-3.0-annotated)