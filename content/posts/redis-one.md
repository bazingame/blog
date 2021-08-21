---
title: "Redis设计与实现之「数据结构」"
summary: Redis 数据结构
date: 2021-08-10
weight: 1
tags: ["redis"]
---

## 基本类型

Redis 对外暴露五种数据类型

- **STRING**
- **LIST**
- **HASH**
- **SET**
- **ZSET**

## 对象、类型、编码

以上每种类型，在底层并不止一种实现。

Redis 创建了一个对象系统，五种基本类型对应五种对象，每个对象底层对应多种数据结构（多种编码）

redis 对象结构体 :

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; 
    int refcount;
    void *ptr;
} robj;
```

- ***type\*** 对象的类型。共有5种，可以使用 `TYPE` 命令查看值对象类型
- ***encoding\*** 对象的编码方式。共有8种，可以使用 `OBJECT ENCODING` 命令查看编码方式
- ***lru\***  记录最后一次被访问时间，用于在特定模式下的优先释放
- ***refcount\*** 对象的计数器，用于内存资源回收
- ***ptr\***  指向真正的底层数据结构，由 ***encoding\*** 决定

Redis 数据库中每个键值对的键和值都是一个对象，键总是一个 **字符串对象**，值则是五种之一。

Redis 每种类型的对象至少有两种编码方式（对应的数据结构）

### 为什么这样设计？

通过 ***encoding\*** 属性来设定对象所使用的编码， 而不是为特定类型的对象关联一种固定的编码， 可以极大提升了 Redis 的灵活性和效率， 因为 Redis 可以根据不同的使用场景来为一个对象设置不同的编码， 从而优化对象在某一场景下的效率。

举例：在列表对象包含的元素比较少时， Redis 使用压缩列表作为列表对象的底层实现：

- 因为压缩列表比双端链表更节约内存， 并且在元素数量较少时， 在内存中以连续块方式保存的压缩列表比起双端链表可以更快被载入到缓存中

- 随着列表对象包含的元素越来越多， 使用压缩列表来保存元素的优势逐渐消失时， 对象就会将底层实现从压缩列表转向**功能更强**、也更适合保存**大量**元素的双端链表上面

  ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d807531d-c661-43d6-ae05-c75458f96fb5/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d807531d-c661-43d6-ae05-c75458f96fb5/Untitled.png)

  压缩列表

  ![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a9c71c04-e433-477d-8bc6-c1ef30431250/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a9c71c04-e433-477d-8bc6-c1ef30431250/Untitled.png)

  双端链表

## 底层基本数据结构

### SDS  (simple dynamic string)

```c
struct sdshdr {
    int len; 
    int free; 
    char buf[];
};
```

- ***len\***      buf 中已占用空间的长度
- ***free\***    buf 中剩余可用空间的长度
- ***buf[]\***  数据空间

设计非常像 Golang 中的 **slice** ，具体为：

- 取长度、容量复杂度都是 *O(1)。*
- 空间预分配。**SDS :**  `new_len = old_len < 1MB ? (2 * old_len) : (1 MB + old_len)` **slice:** `new_len = old_len < 1000 ? (2 * old_len) : (1.25 * old_len)`
- 惰性空间释放 。变短之后只更新 ***free\*** 属性，空间不回收。
- 区别是 **SDS** 中 buff 只可以存放字符串，**slice** 可以支持任何类型。

最新版进化为多种 **sdshdr** 实现，***len\*** 等属性可以指定数据类型 。

### LinkedList 链表

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3d4c37f4-8218-4c72-9486-bb53b1fde325/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3d4c37f4-8218-4c72-9486-bb53b1fde325/Untitled.png)

- 特点：双端，无环，有头、尾指针，有长度
- ****value\*** 可以指向任意类型数据
- ****dup\***，****free\*** ，****match\***  根据 value 类型设定复制，释放，对比函数，实现多态

### HashTable 字典

底层哈希表

```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx;
    int iterators;
} dict;
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/27ae4b4f-a8af-4c89-91dd-51674aa10936/dict.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/27ae4b4f-a8af-4c89-91dd-51674aa10936/dict.png)

- 字典→ 哈希表 → 哈希节点
- 每个字典有两个哈希表 ，用于扩容 rehash
- 哈希算法(MurmurHash2)、哈希冲突等等 过

**Rehash**

为了让哈希表的负载因子维持在合理范围内，字典自动进行扩缩容

- 扩缩的时机：负载因子(used/size)阈值：扩 1/5 ，缩：0.1

为什么会有5？当执行 `BGSAVE` 等命令时，有子程序进行复制，为了不必要的内存操作，节约内存

- 扩缩目标空间：`first(x ≥ used * 2), x = 2^n`。eg: used = 5，则取第一个大于等于 10 (2 * 5) 的 2的n次幂，即 n = 4 时的16
- 扩的方式：
  - ht[0] 数据挨个 rehash 到 ht[1]
  - 释放 ht[0]
  - ht[1] 变为 ht[0]
  - 在 [1] 位置新建空白 ht[1]
- rehash 不是一次性执行完的，是渐进式执行，一带一路 (每次字典的CRUD操作时带一个索引上的键值重新rehash)

避免集中rehash带来的大量计算导致的服务不可用

### SkipList 跳跃表

```c
typedef struct zskiplistNode {
    robj *obj;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8bd05471-7aa8-47e0-b713-87ca9f757cbb/zskiplist.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8bd05471-7aa8-47e0-b713-87ca9f757cbb/zskiplist.png)

- ***length\***         除去头结点的节点个数
- ***level\***            除去头结点的最高层数
- ****obj\***             成员对象
- ***score\***          分值
- ***backward\***  后退指针
- ***forward\***      前进指针
- ***span\***           前进跨度

每次插入一个节点，层数随机生成，补全前进指针

。。。

### IntSet 整数集合

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/275c4fc2-1c48-413b-87c7-9ae9958a4bd5/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/275c4fc2-1c48-413b-87c7-9ae9958a4bd5/Untitled.png)

- ***encoding\*** 指定编码方式，int16,32,64
- 具有升降级操作。每次升降级所有元素都要类型转换，分配内存空间

升降级可以提升灵活性和节约内存，再次榨干

### ZipList 压缩列表

为节约内存而开发的，特殊编码连续内存的顺序型数据结构!

```c
<zlbytes><zltail><zllen><entry><entry><zlend>

<previouts_entry_len><encoding><content>
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/363d5981-1357-44b2-854c-ae1435e78ae1/ziplist.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/363d5981-1357-44b2-854c-ae1435e78ae1/ziplist.png)

- ***zlbytes\***                          总字节数
- ***zltail\***                               尾节点到起始地址字节数
- ***zllen\***                               节点数量
- ***entry\***                              节点
- ***zlend\***                              结尾标识
- ***previouts_entry_len\***    上一个节点的长度
- ***encoding\***                       节点的编码方式。无符号整数、1、3有符号整数等
- ***content\***                          字节数组或整数

压缩列表遍历是从后往前遍历, 前一个节点的地址 `p = c - c.previouts_entry_len`

有小概率的连锁更新，导致所有节点空间重新分配。

### QuickList TODO

3.2版本后新引入

## 编码的转换与命令执行

以上多种数据类型在不同场景下会进行自动转换，从而提升使用效率。

### 字符串对象

在 **int**、**embstr**、**raw**之间转换

- **int** 即为纯数字
- **raw** 即为 **SDS**
- **embstr** 为 **redisObject** 和 **SDS** 一起申请内存，两个数据结构在内存为连续空间。该类型只读，若修改则转换成 **raw**

可以降低内存申请次数、释放次数，更好的利用缓存优势（连续空间）

字符串对象还会被其他对象嵌套（only one）

浮点数什么的会被转成 **embstr**

转换时机为： 执行各种操作，现有类型不满足时，如对 **int** 执行 **APPEND** ，会自动转换为 **raw**

### 列表对象

在 **ziplist** 和 **linkedlist** 之间转换

转换时机为：!字符串元素长度都小于n字节 || 元素数量大于m个

3.2 版本后只有 **quicklist** 一种

### 哈希对象

在 **ziplist** 和 **hashtable** 之间转换

**ziplist**中的每个节点，以kv相邻的形式保存

转换时机为：!键值字符串长度都小于n字节 || 键值对数量大于m个

### 集合对象

在 **intset** 和 **hashtable** 之间转换

转换时机为： !所有元素都是整数 || 数量大于n个

### 有序集合对象

在 **ziplist** 和 **skiplist** 之间转换

**ziplist** 中的每个节点，以 member,score 相邻的形式保存

对于 **skiplist** 编码方式，会同时使用跳跃表和字典来实现

只有跳跃表，按成员查找分值的操作复杂度会上升 只有字典，排序会很慢

是否会浪费内存？不担心，相同元素和分值会共享

转换时机为:  !元素长度都小于n字节 || 数量大于m个

[不同类型和编码的对象](https://www.notion.so/2734f3e0ea5f4ec5897d3727a97bce23)

### 命令执行

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bc82a363-0611-4bb6-a49e-d5f890f70d48/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bc82a363-0611-4bb6-a49e-d5f890f70d48/Untitled.png)

在各种命令执行时，先进行类型检查，是否是该类型支持的命令

类型检查通过后，根据编码，按照指定数据结构命令的实现进行方法调用 （多态）

## 在节约内存上做的其他努力

### 内存回收

- 看 **redisObject** 里的 **refcount** 属性
- 初始化为1
- 当被新程序使用时，会+1，不再使用时 -1
- 当为0时就释放

### 对象共享

初始化服务器时，整数（0-9999）字符串对象值会提前创建，用到直接使用共享，而不是新创建

## Reference

[go语言的slice和redis的SDS对比](https://studygolang.com/articles/28262)

[Redis的List，从linkedlist和ziplist再到quicklist](https://blog.liexing.me/2019/12/28/from-ziplist-linkedlist-to-quicklist/)
