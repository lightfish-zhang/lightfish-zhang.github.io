---
layout: post
title: redis源码学习笔记-server的概念
date:   2017-7-28 20:30:00 +0800
category: redis 
tag: [server]
---


* content
{:toc}

## server概念

## 源码分析

### 数据结构

#### 服务端

Redis服务器的`redisServer`结构体，一眼看过去，有两百多个成员。。。，这里只贴需要讲的

```c
struct redisServer {
    redisDb *db;    /* a array which save all db */                
    int dbnum;      /* Total number of configured DBs */
}
```

```c

- `dbnum`属性的值由服务器配置的database选项决定，默认16，所以Redis默认创建16个数据库，也就是`db`数组有16个元素
- 默认情况下，Redis客户端的目标数据库为0号数据库，通过命令`SELECT`可以切换数据库


#### 数据库

```c
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */

    // 正处于阻塞状态的键
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */

    // 可以解除阻塞的键
    dict *ready_keys;           /* Blocked keys that received a PUSH */

    // 正在被 WATCH 命令监视的键
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */

    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */

    // 数据库号码
    int id;                     /* Database ID */

    // 数据库的键的平均 TTL ，统计信息
    long long avg_ttl;          /* Average TTL, just for stats */

} redisDb;
```

- Redis是一个键值对数据库服务，服务器的每一个数据库都由结构体`redisDb`表示
- 成员`dict`保存了数据库中的所有键值对，我们称这个字典为键空间`key space`
- 键空间的键也就是数据库的键，每一个键都是一个字符串对象
- 键空间的值也就是数据库的值，每个值是一种redis对象，如: 字符串对象，列表对象，哈希表对象，集合对象，有序集合对象
- 增删改查命令示例如下

```
127.0.0.1:6379[2]> set message "hello world"
OK
127.0.0.1:6379[2]> object refcount message
(integer) 1
127.0.0.1:6379[2]> del message
(integer) 0
127.0.0.1:6379[2]> object refcount message
(nil)
127.0.0.1:6379[2]> rpush alist "a" "b" "c"
(integer) 3
127.0.0.1:6379[2]> lpop alist
"a"
127.0.0.1:6379[2]> lpop alist
"b"
127.0.0.1:6379[2]> hset book name "redis in action"
(integer) 1
127.0.0.1:6379[2]> hget book name
"redis in action"
```

#### 过期字典与键有效期

- 设置键的生存时间或过期时间
    + 命令`expire`设置有效期，单位秒
    + 命令`expireat`设置过期时间，使用UNIX时间戳
    + 命令`ttl`返回剩余时间，单位秒
    + 命令`pttl`返回，单位毫秒
    + 命令`persist`，解除有效期

```
127.0.0.1:6379[2]> set message "hello word"
OK
127.0.0.1:6379[2]> get message
"hello word"
127.0.0.1:6379[2]> expire message 5
(integer) 1
127.0.0.1:6379[2]> get message  // in 5 seconds
"hello word"
127.0.0.1:6379[2]> get message  // after 5 seconds
(nil)
```

- 键的有效期的数据结构

```
typedef struct redisDb {
    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */
}
```

- 结构体`redisDb`的成员`expires`为字典类型，保存了数据库中所有键的过期时间，我们称之为过期字典
    + 过期字典的键是一个指针，这个指针指向键空间中的某个键对象
    + 过期字典的值是一个`long long`类型的整数，这个整数保存了键所指向的过期时间，毫秒精度的UNIX时间戳

- 过期键删除策略，可能的方法有：
    + 定时删除，设置键的过期时间的同时，创建删除该键的定时器，缺点是可能会创建太多定时器
    + 惰性删除，放任键过期不管，只是在`get`命令执行时，检查过期时间，如果过期则删除，没过期就返回，缺点是垃圾内存可能过多
    + 定期删除，程序设置定时器，定期检查`expires`过期字典并删除过期的键，缺点是删除不及时

- Redis采用的过期删除策略是惰性删除与定期删除的配合使用
    + 惰性删除的实现源码在`src/db.c`的函数`int expireIfNeeded(redisDb *db, robj *key)`
    + 定期删除的实现源码在`src/redis.c`的函数`int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now)`


### 与键过期相关功能

#### RDB,AOF等持久化功能

- RDB持久化功能
    + 保存RDB文件时，不会保存过期的键
    + 导入RDB文件时，主服务器不会保存过期的键，从服务器则所有过期没过期的键都会保存，在主从同步时再删除

- AOF持久化功能
    + 写入AOF文件时，不会写入过期的键，同时会追加一条键被删除的命令到AOF文件



### 数据库通知

- 相关源码在`src/notify.c`



## 参考文献

- [Redis 设计与实现](http://redisbook.com/)
- [带注释的Redis源码](https://github.com/huangz1990/redis-3.0-annotated)