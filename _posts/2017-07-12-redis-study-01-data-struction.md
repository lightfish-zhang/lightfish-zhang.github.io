---
layout: post
title: redis源码学习笔记-6种数据结构
date:   2017-7-12 20:30:00 +0800
category: redis 
tag: [algorithms]
---


* content
{:toc}


## redis的动态字符串

### 动态字符串的数据结构

- 代码 /src/sds.h

```c
/*
 * 保存字符串对象的结构
 */
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

- `buf`遵循c语言的字符串以`\0`结尾的惯例，以直接使用c语言标准库的函数。
- `len`不包含`\0`的1byte
- `buf`的长度 = `len` + `free` + 1
- redis有`SDS`的API操作，如`STRLEN`命令

#### 改善c语言字符串的问题

- c字符串不保存长度，所以获取长度的时间复杂度为O(n)，`SDS`结构体保存长度，获取长度的时间复杂度为O(1)
- c标准函数的`char *strcat(char *dest, const char *src)`用作拼接字符串，但是需要`dest`留有足够的空间，需要程序员控制。而`SDS`的API需要修改字符串时，会先检查空间，按需修改空间大小，防止出现空间溢出的问题
- 分配内存空间是一个耗时的操作，对于io密集的数据库redis, 它的优化策略是空间预分配和惰性空间释放
    + 空间预分配，长度少于1MB时，`len`=`free`；长度大于等于1MB时，`free`=1MB
    + 惰性空间释放，缩短字符串时不会回收空间，在有需要时才真正释放空间

- c字符串的字符必须符合某种编码如`ASCII`,并且除了末尾外其他元素不得含有`\0`，这些限制使c字符串只能保存文本数据，不能保存如图片、压缩文件、音频等二进制数据
- `SDS`的`buf`保存的是字节数组，它的API都是二进制安全的`binary-safe`，使用`len`来判断字符串是否结束，而不使用`\0`来判断末尾
- `buf`使用`\0`结尾，兼容部分c字符串函数，重用`string.h`，如`strcasecmp()`


## 链表

### redis实现的链表的数据结构

```c
/* Node, List, and Iterator are the only data structures used currently. */

/*
 * 双端链表节点
 */
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;

/*
 * 双端链表迭代器
 */
typedef struct listIter {

    // 当前迭代到的节点
    listNode *next;

    // 迭代的方向
    int direction;

} listIter;

/*
 * 双端链表结构
 */
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;
```

#### 链表的特点

- 双端，节点有`prev`/`next`，获取前后节点的复杂度为O(1)
- 无环，表头的`prev`=NULL，表尾的`next`=NULL，对链表的遍历以NULL结束
- 带表头指针和表尾指针，`list`有`head`/`tail`
- 带链表长度计数器，`list`有`len`，获取链表长度时间复杂度为O(1)
- 多态，node的`value`为`void *`，该链表结构可以用于不同的类型的值



## 字典

### 概念

- 字典，是一种保存键值对(key-value pair)的抽象数据结构，又称为符号表`symbol table`,关联数组`associative array`,映射`map`
- 字典的每一个键是唯一的，与一个值进行关联
- 在redis中，对数据库的增删改查是构建在对字典的操作之上的

### 源码分析

#### 哈希表数据结构

- 代码定义`src/dict.h`

```c
/*
 * 哈希表
 *
 * 每个字典都使用两个哈希表，从而实现渐进式 rehash 。
 */
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;
/*
 * 哈希表节点
 */
typedef struct dictEntry {
    
    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```

- redis使用`MurmurHash2`算法计算哈希值，参考 http://code.google.com/p/smhasher/
- 键冲突，哈希冲突的键值，使用单向链表存储，新节点总是添加链表的表头，这样添加的复杂度为O(1)


#### 哈希表的扩展与收缩 —— rehash

- 各个名词
    + `ht`哈希表，`ht[0]`是rehash前的哈希表，`ht[1]`是rehash后的哈希表
    + `ht.used` 当前包含的键值对数量
    + `load factor`=`ht.used`/`ht.size`，负载因子为哈希表已保存字节数与哈希表大小的比

- 哈希表的rehash
    + 扩展，分配更大的空间，`ht[1].size`=`ht[0].size * 2`
    + `rehash`，重新计算键的哈希值和索引值，然后将键值对放置到`ht[1]`哈希表的相应的位置上
    + 收缩，当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作

- 子进程存在时，用来判断是否扩展操作的负载因子的临界值会提高
    + 如`BGSAVE`,`BGREWRITEAOF`命令，程序会开子进程去处理
    + 提高扩展操作的临界值，避免不必要的内存写入操作（父子进程同时扩展？）

- rehash的过程是渐进式的
    + 当`ht[0]`的键值对是百万计时，显然，短时间不能一次性全部refash到`ht[1]`，数据库不能花费长时间在这上面
    + 分多次、渐进式地执行refash，是取于平衡的操作
    + 字典中维持一个索引计数器，`rehashidx`，初始值0
    + rehash进行期间，每次对字典的增删改查，都会顺便将`ht[0]`的一个键值对rehash到`ht[1]`，且`rehashidx += 1`
    + rehash结束，`rehashidx`设为-1
    + 由此可见，rehash将计算工作均摊到字典的每个操作上，分而治之
    + rehash进行期间，对一个键值的查找，先查`ht[0]`，若不存在就查`ht[1]`
    + rehash进行期间，新增操作则只对`ht[1]`，最终使`ht[0]`变成空表


#### 源码实例

```c
/*
 * 返回字典中包含键 key 的节点
 *
 * 找到返回节点，找不到返回 NULL
 *
 * T = O(1)
 */
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    unsigned int h, idx, table;

    // 字典（的哈希表）为空
    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */

    // 如果条件允许的话，进行单步 rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 计算键的哈希值
    h = dictHashKey(d, key);
    // 在字典的哈希表中查找这个键
    // T = O(1)
    for (table = 0; table <= 1; table++) {

        // 计算索引值
        idx = h & d->ht[table].sizemask;

        // 遍历给定索引上的链表的所有节点，查找 key
        he = d->ht[table].table[idx];
        // T = O(1)
        while(he) {

            if (dictCompareKeys(d, key, he->key))
                return he;

            he = he->next;
        }

        // 如果程序遍历完 0 号哈希表，仍然没找到指定的键的节点
        // 那么程序会检查字典是否在进行 rehash ，
        // 然后才决定是直接返回 NULL ，还是继续查找 1 号哈希表
        if (!dictIsRehashing(d)) return NULL;
    }

    // 进行到这里时，说明两个哈希表都没找到
    return NULL;
}
```


## 跳跃表

### 源码分析

#### 数据结构

```c
/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;

/*
 * 跳跃表
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;

```

- 跳跃表在单向链表的基础上
    + 节点的`* next`变成了`level[]`，可以说`level[]`是`* next`的数组，存储了后面第1个节点的指针、第2个节点的指针。。。第n个节点的指针
    + 跳跃表`skiplist`的属性`level`保存了表中层数最大的节点的层数。

- 比单向列表的优越点
    + 单向链表的查找，一个一个节点遍历的，时间复杂度为O(n), n为节点数
    + 而跳跃表，如果节点是按节点某属性的值大小排序的，也就是有序链表，当节点加上跳跃点后，那么遍历的效率就会大大增加
    + 举个例子，链表是排序的，并且节点中还存储了指向前面第二个节点的指针的话，那么在查找一个节点时，仅仅需要遍历N/2个节点即可。


#### 应用

- 在redis中，跳跃表作为有序集合键的底层实现之一，应用场景:
    + 一个有序集合包含的元素数量比较多
    + 有序集合中元素的成员（member）是比较长的字符串

- 在redis中，跳跃表节点的层高都是1~32之间的随机数
- 在同一个跳跃表中，多个节点可以包含相同的分值，但每个节点的成员对象必须是唯一的
- 跳跃表中的节点按照分值大小进行排序，当分值相同时，节点按照成员对象的大小排序


## 整数集合

### 源码分析

#### 数据结构

```c
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

- `encoding`属性，
    + `INTSET_ENC_INT16` 
    + `INTSET_ENC_INT32` 
    + `INTSET_ENC_INT64` 

- `contents[]`，整数集合，每个元素从小到大排序，数组中不出现重复项
- API与复杂度
    + `insetAdd`/`insetRemove`，在c语言的数组中插入或新的元素，时间复杂度O(N)
    + `insetNew`, 创建一个新的整数集合，O(1)
    + `insetFind` 检查给定值是否存在与集合，因为是有序数组，可以使用二分法，O(logN)
    + `intsetRandom` 随机返回一个元素 O(1)
    + `intsetGet` 取指定索引的元素 O(1)
    + `intsetLen` 返回元素个数 O(1)
    + `intsetBlobLen` 返回集合所占内存字节数

## 压缩列表

### 概念

- 是为了尽可能地节约内存而设计的特殊编码双端链表。没有定义c语言的结构体，而是特殊的二进制格式。

#### 源码中的注释和翻译

```
The ziplist is a specially encoded dually linked list that is designed
to be very memory efficient. 

Ziplist 是为了尽可能地节约内存而设计的特殊编码双端链表。

It stores both strings and integer values,
where integers are encoded as actual integers instead of a series of
characters. 

Ziplist 可以储存字符串值和整数值，
其中，整数值被保存为实际的整数，而不是字符数组。

It allows push and pop operations on either side of the list
in O(1) time. However, because every operation requires a reallocation of
the memory used by the ziplist, the actual complexity is related to the
amount of memory used by the ziplist.

Ziplist 允许在列表的两端进行 O(1) 复杂度的 push 和 pop 操作。
但是，因为这些操作都需要对整个 ziplist 进行内存重分配，
所以实际的复杂度和 ziplist 占用的内存大小有关。

----------------------------------------------------------------------------

ZIPLIST OVERALL LAYOUT:
Ziplist 的整体布局：

The general layout of the ziplist is as follows:
以下是 ziplist 的一般布局：

<zlbytes><zltail><zllen><entry><entry><zlend>

<zlbytes> is an unsigned integer to hold the number of bytes that the
ziplist occupies. This value needs to be stored to be able to resize the
entire structure without the need to traverse it first.

<zlbytes> 是一个无符号整数，保存着 ziplist 使用的内存数量。

通过这个值，程序可以直接对 ziplist 的内存大小进行调整，
而无须为了计算 ziplist 的内存大小而遍历整个列表。

<zltail> is the offset to the last entry in the list. This allows a pop
operation on the far side of the list without the need for full traversal.

<zltail> 保存着到达列表中最后一个节点的偏移量。

这个偏移量使得对表尾的 pop 操作可以在无须遍历整个列表的情况下进行。

<zllen> is the number of entries.When this value is larger than 2**16-2,
we need to traverse the entire list to know how many items it holds.

<zllen> 保存着列表中的节点数量。

当 zllen 保存的值大于 2**16-2 时，
程序需要遍历整个列表才能知道列表实际包含了多少个节点。

<zlend> is a single byte special value, equal to 255, which indicates the
end of the list.

<zlend> 的长度为 1 字节，值为 255 ，标识列表的末尾。

ZIPLIST ENTRIES:
ZIPLIST 节点：

Every entry in the ziplist is prefixed by a header that contains two pieces
of information. First, the length of the previous entry is stored to be
able to traverse the list from back to front. Second, the encoding with an
optional string length of the entry itself is stored.

每个 ziplist 节点的前面都带有一个 header ，这个 header 包含两部分信息：

1)前置节点的长度，在程序从后向前遍历时使用。

2)当前节点所保存的值的类型和长度。

The length of the previous entry is encoded in the following way:
If this length is smaller than 254 bytes, it will only consume a single
byte that takes the length as value. When the length is greater than or
equal to 254, it will consume 5 bytes. The first byte is set to 254 to
indicate a larger value is following. The remaining 4 bytes take the
length of the previous entry as value.

编码前置节点的长度的方法如下：

1) 如果前置节点的长度小于 254 字节，那么程序将使用 1 个字节来保存这个长度值。

2) 如果前置节点的长度大于等于 254 字节，那么程序将使用 5 个字节来保存这个长度值：
   a) 第 1 个字节的值将被设为 254 ，用于标识这是一个 5 字节长的长度值。
   b) 之后的 4 个字节则用于保存前置节点的实际长度。

The other header field of the entry itself depends on the contents of the
entry. When the entry is a string, the first 2 bits of this header will hold
the type of encoding used to store the length of the string, followed by the
actual length of the string. When the entry is an integer the first 2 bits
are both set to 1. The following 2 bits are used to specify what kind of
integer will be stored after this header. An overview of the different
types and encodings is as follows:

header 另一部分的内容和节点所保存的值有关。

1) 如果节点保存的是字符串值，
   那么这部分 header 的头 2 个位将保存编码字符串长度所使用的类型，
   而之后跟着的内容则是字符串的实际长度。

|00pppppp| - 1 byte
     String value with length less than or equal to 63 bytes (6 bits).
     字符串的长度小于或等于 63 字节。
|01pppppp|qqqqqqqq| - 2 bytes
     String value with length less than or equal to 16383 bytes (14 bits).
     字符串的长度小于或等于 16383 字节。
|10______|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
     String value with length greater than or equal to 16384 bytes.
     字符串的长度大于或等于 16384 字节。

2) 如果节点保存的是整数值，
   那么这部分 header 的头 2 位都将被设置为 1 ，
   而之后跟着的 2 位则用于标识节点所保存的整数的类型。

|11000000| - 1 byte
     Integer encoded as int16_t (2 bytes).
     节点的值为 int16_t 类型的整数，长度为 2 字节。
|11010000| - 1 byte
     Integer encoded as int32_t (4 bytes).
     节点的值为 int32_t 类型的整数，长度为 4 字节。
|11100000| - 1 byte
     Integer encoded as int64_t (8 bytes).
     节点的值为 int64_t 类型的整数，长度为 8 字节。
|11110000| - 1 byte
     Integer encoded as 24 bit signed (3 bytes).
     节点的值为 24 位（3 字节）长的整数。
|11111110| - 1 byte
     Integer encoded as 8 bit signed (1 byte).
     节点的值为 8 位（1 字节）长的整数。
|1111xxxx| - (with xxxx between 0000 and 1101) immediate 4 bit integer.
     Unsigned integer from 0 to 12. The encoded value is actually from
     1 to 13 because 0000 and 1111 can not be used, so 1 should be
     subtracted from the encoded 4 bit value to obtain the right value.
     节点的值为介于 0 至 12 之间的无符号整数。
     因为 0000 和 1111 都不能使用，所以位的实际值将是 1 至 13 。
     程序在取得这 4 个位的值之后，还需要减去 1 ，才能计算出正确的值。
     比如说，如果位的值为 0001 = 1 ，那么程序返回的值将是 1 - 1 = 0 。
|11111111| - End of ziplist.
     ziplist 的结尾标识

All the integers are represented in little endian byte order.

所有整数都表示为小端字节序。
```

```
/* 
空白 ziplist 示例图

area        |<---- ziplist header ---->|<-- end -->|

size          4 bytes   4 bytes 2 bytes  1 byte
            +---------+--------+-------+-----------+
component   | zlbytes | zltail | zllen | zlend     |
            |         |        |       |           |
value       |  1011   |  1010  |   0   | 1111 1111 |
            +---------+--------+-------+-----------+
                                       ^
                                       |
                               ZIPLIST_ENTRY_HEAD
                                       &
address                        ZIPLIST_ENTRY_TAIL
                                       &
                               ZIPLIST_ENTRY_END

非空 ziplist 示例图

area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                        ZIPLIST_ENTRY_TAIL
*/
```


## 参考文献

- [Redis 设计与实现](http://redisbook.com/)
- [带注释的Redis源码](https://github.com/huangz1990/redis-3.0-annotated)