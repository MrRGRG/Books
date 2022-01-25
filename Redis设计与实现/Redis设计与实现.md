# Redis设计与实现

## 第1章 引言

### 1.1 Redis版本说明 

本书是基于Redis 3.0的开发版来编写的，

### 1.2 章节编排

本书由“数据结构与对象”、“单机数据库的实现”、“多机数据库的实现”、“独立功能的实现”四个部分组成。

### 1.3 推荐的阅读方法

多级版本用户：推荐按顺序阅读

单机版本用户：可以跳过第三部分

### 1.4 行文规则

名字引用规则：file/name格式（文件/函数）

结构引用规则：struct.property格式

算法规则：除非有额外说明，否则本书列出的算法复杂度一律为最坏情形下的算法复杂度。

代码规则：C语言和Python语言

server和client两个全局变量，其中server表示服务器状态（redis.h/redisServer结构的实例），而client则表示正在执行操作的客户端状态（redis.h/redisClient结构的实例）。

## 第一部分 数据结构与对象

### 第2章 简单动态字符串

#### 2.1 SDS的定义 

每个sds.h/sdshdr结构表示一个SDS值：

```c
struct sdshdr{
    int len;
    int free;
    // 以'\0'结尾，不计入len
    char buf[];
}
```

#### 2.2 SDS与C字符串的区别 

##### 2.2.1 常数复杂度获取字符串长度

C字符串为了获取一个C字符串的长度，必须遍历整个字符串，这个操作的复杂度为O（N）。

SDS，获取一个SDS长度的命令STRLEN，复杂度仅为O（1）。

##### 2.2.2 杜绝缓冲区溢出

C字符串容易造成缓冲区溢出（bufferoverflow）。

SDS杜绝了发生缓冲区溢出的可能性：当SDS API需要对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的要求，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作。

##### 2.2.3 减少修改字符串时带来的内存重分配次数

C字符串每次增长或者缩短一个，程序都总要对保存这个C字符串的数组进行一次内存重分配操作。

SDS实现了两种优化策略

1. 空间预分配
   1. 对SDS进行修改之后，len<1MB，那么程序分配和len属性同样大小的未使用空间，len=free。
   2. 如果对SDS进行修改之后，len>=1MB，那么程序会分配1MB的未使用空间,free=1MB。
2. 惰性空间释放
   1. SDS的字符串缩短操作：使用free属性将这些字节的数量记录起来，不释放空间。

##### 2.2.4 二进制安全

C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据

SDS的API都是二进制安全的（binary-safe），所有SDS API都会以处理二进制的方式来处理SDS存放在buf数组里的数据，程序不会对其中的数据做任何限制、过滤、或者假设，**数据在写入时是什么样的，它被读取时就是什么样**。

##### 2.2.5 兼容部分C字符串函数

SDS可以在有需要时重用＜string.h＞函数库，从而避免了不必要的代码重复

##### 2.2.6 总结

![image-20211208225711162](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20211208225711162.png)

#### 2.3 SDS API

![image-20211208225931735](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20211208225931735.png)

#### 2.4 重点回顾

- Redis只会使用C字符串作为字面量，在大多数情况下，Redis使用SDS（Simple Dynamic String，简单动态字符串）作为字符串表示。

- 比起C字符串，SDS具有以下优点：
  - 常数复杂度获取字符串长度。
  - 杜绝缓冲区溢出。
  - 减少修改字符串长度时所需的内存重分配次数。
  - 二进制安全。
  - 兼容部分C字符串函数。

### 第3章 链表

链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

#### 3.1 链表和链表节点的实现

每个链表节点使用一个adlist.h/listNode结构来表示：

```c
typedef struct listNode {
	struct listNode * prev;
	struct listNode *next;
	void * value;
}
```

使用adlist.h/list来持有链表的话，操作起来会更方便：

```c
typedef struct list {
    listNode *head;
    listNode *tail;
    // 节点复制函数
    void *(*dup)(void *ptr);
    // 节点释放函数
    void (*free)(void *ptr);
    // 节点对比函数
    int (*match)(void *ptr, void *key);
    // 节点数量
    unsigned long len;
} list;
```

### 第4章 字典

#### 4.1  字典的实现

Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

##### 4.1.1 哈希表

redis6之后没了

```c
typedef struct dict {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
	// 哈希表大小掩码，用于计算索引值
    // 总是=size-1
    unsigned long sizemarsk;
	// 该哈希表已有的节点的数量
    unsigned long used;
};
```

大小为4的空哈希表

![image-20211225201810660](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20211225201810660.png)

##### 4.1.2 哈希表节点

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 指向下个哈希表节点，形成链表（解决哈希冲突）
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as returned
                                 * by dictType's dictEntryMetadataBytes(). */
} dictEntry;
```

![image-20211225202054152](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20211225202054152.png)

##### 4.1.3 字典

```c

struct dict {
    // 类型特定函数
    dictType *type;
    // 键值对储存-》6之后把哈希表拆出来
    dictEntry **ht_table[2];
    // 哈希表已存的元素个数
    unsigned long ht_used[2];
	//  如果 rehashidx == -1，则不会进行重新哈希
    long rehashidx; 

    /* 在末尾保留小变量以获得最佳（最小）结构填充 */
    int16_t pauserehash; /* 如果 >0 重新散列暂停（<0 表示编码错误） */
    signed char ht_size_exp[2]; /*大小的指数。 （大小 = 1<<exp） */
};

typedef struct dictType {
    // 计算哈希值的函数
    uint64_t (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(dict *d, const void *key);
    // 复制值的函数
    void *(*valDup)(dict *d, const void *obj);
    // 对比键的函数
    int (*keyCompare)(dict *d, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(dict *d, void *key);
    // 销毁值的函数
    void (*valDestructor)(dict *d, void *obj);
    
    int (*expandAllowed)(size_t moreMem, double usedRatio);
    // 允许 dictEntry 携带额外的调用者定义的元数据。 当分配 dictEntry 时，额外的内存被初始化为 0。
    size_t (*dictEntryMetadataBytes)(dict *d);
} dictType;

```

#### 4.2 哈希算法

Redis使用MurmurHash2算法来计算键的哈希值。

```c

// 使用字典设置的哈希函数，用key计算哈希值
hash = dict->type->hashFunction(key);
// 根据情况不同
index = hash & dict->ht_table[x]
```

#### 4.6 API

```c
/* API */
dict *dictCreate(dictType *type);
int dictExpand(dict *d, unsigned long size);
int dictTryExpand(dict *d, unsigned long size);
int dictAdd(dict *d, void *key, void *val);
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing);
dictEntry *dictAddOrFind(dict *d, void *key);
int dictReplace(dict *d, void *key, void *val);
int dictDelete(dict *d, const void *key);
dictEntry *dictUnlink(dict *d, const void *key);
void dictFreeUnlinkedEntry(dict *d, dictEntry *he);
void dictRelease(dict *d);
dictEntry * dictFind(dict *d, const void *key);
void *dictFetchValue(dict *d, const void *key);
int dictResize(dict *d);
dictIterator *dictGetIterator(dict *d);
dictIterator *dictGetSafeIterator(dict *d);
dictEntry *dictNext(dictIterator *iter);
void dictReleaseIterator(dictIterator *iter);
dictEntry *dictGetRandomKey(dict *d);
dictEntry *dictGetFairRandomKey(dict *d);
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count);
void dictGetStats(char *buf, size_t bufsize, dict *d);
uint64_t dictGenHashFunction(const void *key, size_t len);
uint64_t dictGenCaseHashFunction(const unsigned char *buf, size_t len);
void dictEmpty(dict *d, void(callback)(dict*));
void dictEnableResize(void);
void dictDisableResize(void);
int dictRehash(dict *d, int n);
int dictRehashMilliseconds(dict *d, int ms);
void dictSetHashFunctionSeed(uint8_t *seed);
uint8_t *dictGetHashFunctionSeed(void);
unsigned long dictScan(dict *d, unsigned long v, dictScanFunction *fn, dictScanBucketFunction *bucketfn, void *privdata);
// 获取哈希值
uint64_t dictGetHash(dict *d, const void *key);
dictEntry **dictFindEntryRefByPtrAndHash(dict *d, const void *oldptr, uint64_t hash);
```



### 第5章 跳跃表

1. 跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。
2. Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员（member）是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。
2. 和链表、字典等数据结构被广泛地应用在Redis内部不同，**Redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群节点中用作内部数据结构**
2. 每个跳跃表节点的层高都是1至32之间的随机数

#### 5.1 跳跃表的实现

![image-20220118222242262](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118222242262.png)

```C
typedef char *sds;

/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 表头和表尾
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 层数
    int level;
} zskiplist;
```

1. 层

   每次创建一个新跳跃表节点的时候，程序都根据幂次定律（powerlaw，越大的数出现的概率越小）随机生成一个介于1和32之间的值作为level数组的大小，这个大小就是层的“高度”。

2. 前进指针

   每个层都有一个指向表尾方向的前进指针，用于从表头向表尾方向访问节点

3. 跨度

   层的跨度（level[i].span属性）用于记录两个节点之间的距离：

   跨度实际上是用来计算排位（rank）的：在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位。

4. 后退指针

   节点的后退指针（backward属性）用于从表尾向表头方向访问节点：只能跳一次，不像前进指针可以有好多个

5. 分值和成员

   节点的分值（score属性）是一个double类型的浮点数，跳跃表中的所有节点都按分值从小到大来排序。

   ele 是一个char指针

#### 5.2 跳跃表API

![image-20220118222836367](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118222836367.png)

### 第6章 整数集合

整数集合（intset）是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。

#### 6.1 整数集合的实现

1. 整数集合（intset）是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且**保证集合中不会出现重复元素**。

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 元素数量
    uint32_t length;
    // 保存元素的数组，从小到大且不重复，
    int8_t contents[];
} intset;
```

2. contents数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数组项（item），各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。
3. contents数组的真正类型取决于encoding属性的值:

| encoding类型     | contents数组类型 | 范围                                        |
| ---------------- | ---------------- | ------------------------------------------- |
| INTSET_ENC_INT16 | int16_t          | [-32768，32767]                             |
| INTSET_ENC_INT32 | int32_t          | [-2147483648，2147483647]                   |
| INTSET_ENC_INT64 | int64_t          | [-9223372036854775808，9223372036854775807] |


4. 举例：

   ![image-20220125214434852](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125214434852.png)

#### 6.2 升级

1. 每当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级（upgrade），然后才能将新元素添加到整数集合里面。
2. 步骤：
   1. 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
   2. 将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。
   3. 将新元素添加到底层数组里面。
3. 因为每次向整数集合添加新元素都可能会引起升级，而每次升级都需要对底层数组中已有的所有元素进行类型转换，所以向整数集合添加新元素的时间复杂度为O（N）。
4. 升级之后新元素的摆放位置：因为引发升级的新元素的长度总是比整数集合现有所有元素的长度都大，所以这个新元素的值要么就大于所有现有元素，要么就小于所有现有元素

#### 6.3 升级的好处

1. 一个是提升整数集合的灵活性。C语言是静态类型语言，为了避免类型错误，我们通常不会将两种不同类型的值放在同一个数据结构里面。
2. 另一个是尽可能地节约内存。

#### 6.4 降级

整数集合不支持降级操作，一旦对数组进行了升级，编码就会一直保持升级后的状态。

#### 6.5 整数集合API

![image-20220125215813277](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125215813277.png)

### 第7章 压缩列表

1. **压缩列表（ziplist）是列表键和哈希键的底层实现之一**。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。
2. 当一个哈希键只包含少量键值对，比且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做哈希键的底层实现。

#### 7.1 压缩列表的构成

1. 压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

2. 压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

3. 组成：

   ![image-20220125220451183](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125220451183.png)

4. 各部分说明

   ![image-20220125220526714](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125220526714.png)

#### 7.2 压缩列表节点的构成

1. 每个压缩列表节点可以保存一个字节数组或者一个整数值

2. 字节数组可以是以下三种长度的其中一种：

   1. 长度小于等于63（2 6–1）字节的字节数组；
   2. 长度小于等于16383（2 14–1）字节的字节数组；
   3. 长度小于等于4294967295（2 32–1）字节的字节数组；

3. 整数值则可以是以下六种长度的其中一种：

   1. 4位长，介于0至12之间的无符号整数；
   2. 1字节长的有符号整数；
   3. 3字节长的有符号整数；
   4. int16_t类型整数；
   5. int32_t类型整数；
   6. int64_t类型整数。

4. 每个压缩列表节点都由previous_entry_length、encoding、content三个部分组成

   ![image-20220125221209388](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125221209388.png)

##### 7.2.1 previous_entry_length

1. 节点的previous_entry_length属性以字节为单位，记录了压缩列表中前一个节点的长度。previous_entry_length属性的长度可以是1字节或者5字节：
   1. 如果前一节点的长度小于254字节，那么previous_entry_length属性的长度为1字节：前一节点的长度就保存在这一个字节里面。
   2. 如果前一节点的长度大于等于254字节，那么previous_entry_length属性的长度为5字节：其中属性的第一字节会被设置为0xFE（十进制值254），而之后的四个字节则用于保存前一节点的长度。

2. 程序可以通过指针运算，根据当前节点的起始地址来计算出前一个节点的起始地址。

##### 7.2.2 encoding

1. 节点的encoding属性记录了节点的content属性所保存数据的类型以及长度：

   1. 一字节、两字节或者五字节长，值的最高位为00、01或者10的是字节数组编码：这种编码表示节点的content属性保存着字节数组，数组的长度由编码除去最高两位之后的其他位记录；
   2. 一字节长，值的最高位以11开头的是整数编码：这种编码表示节点的content属性保存着整数值，整数值的类型和长度由编码除去最高两位之后的其他位记录；

2. 字节数组编码![image-20220125234140214](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125234140214.png)

3. 整数编码

   ![image-20220125234250793](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125234250793.png)

##### 7.2.3 content

节点的content属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定。

#### 7.3 连锁更新

1. Redis将这种在特殊情况下产生的连续多次空间扩展操作称之为“连锁更新”（cascade update）
2. 除了添加新节点可能会引发连锁更新之外，删除节点也可能会引发连锁更新。
3. 因为连锁更新在最坏情况下需要对压缩列表执行N次空间重分配操作，而每次空间重分配的最坏复杂度为O（N），所以连锁更新的最坏复杂度为O（N 2）。
4. ziplistPush等命令的平均复杂度仅为O（N），

#### 7.4 压缩列表API

![image-20220125234427023](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125234427023.png)

### 第8章 对象

Redis的对象系统还实现了基于引用计数技术的内存回收机制，当程序不再使用某个对象的时候，这个对象所占用的内存就会被自动释放；另外，Redis还通过引用计数技术实现了对象共享机制，这一机制可以在适当的条件下，通过让多个数据库键共享同一个对象来节约内存。

#### 8.1 对象的类型与编码

1. Redis使用对象来表示数据库中的键和值，每次当我们在Redis的数据库中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键（键对象），另一个对象用作键值对的值（值对象）。
2. Redis中的每个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding属性和ptr属性：

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    // 指向底层实现数据结构的指针
    void *ptr;
} robj;
```

3. 对象的type属性记录了对象的类型

   ![image-20220118232828683](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118232828683.png)

4. 对于Redis数据库保存的键值对来说，键总是一个字符串对象，而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种

5. Type命令输出

   ![image-20220118233821034](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118233821034.png)

6. 对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

   ![image-20220118233843251](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118233843251.png)

7. 每种类型的对象都至少使用了两种不同的编码

   ![image-20220118233909701](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118233909701.png)

8. 通过encoding属性来设定对象所使用的编码，而不是为特定类型的对象关联一种固定的编码，极大地提升了Redis的灵活性和效率，因为Redis可以根据不同的使用场景来为一个对象设置不同的编码，从而优化对象在某一场景下的效率。

9. 可以通过`OBJECT ENCODING key`来查询

#### 8.2 字符串对象

1. 字符串对象的编码可以是int、raw或者embstr。

2. 如果一个字符串对象保存的是整数值，并且这个整数值可以用long类型来表示，那么字符串对象会将整数值保存在字符串对象结构的ptr属性里面（将void*转换成long），并将字符串对象的编码设置为int。

   ![image-20220118234120388](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118234120388.png)

3. 如果字符串对象保存的是一个字符串值，并且这个字符串值的长度大于32字节，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，并将对象的编码设置为raw。

   ![image-20220118234228583](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118234228583.png)

4. 如果字符串对象保存的是一个字符串值，并且这个字符串值的长度小于等于32字节，那么字符串对象将使用embstr编码的方式来保存这个字符串值。embstr编码是专门用于保存短字符串的一种优化编码方式。raw编码会调用两次内存分配函数来分别创建redisObject结构和sdshdr结构，而embstr编码则通过调用一次内存分配函数来分配一块连续的空间，空间中依次包含redisObject和sdshdr两个结构

   ![image-20220118234631114](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118234631114.png)
   
5. ![image-20220118235415087](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220118235415087.png)

6. 总结一下

   ![image-20220123222438137](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220123222438137.png)

7. 对于int编码的字符串对象来说，如果我们向对象执行了一些命令，使得这个对象保存的不再是整数值，而是一个字符串值，那么字符串对象的编码将从int变为raw。

8. **字符串命令的实现**

   1. 因为字符串键的值为字符串对象，所以用于字符串键的所有命令都是针对字符串对象来构建的，

      ![image-20220123222633435](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220123222633435.png)


#### 8.3 列表对象

1. 列表对象的编码可以是ziplist或者linkedlist。

2. ziplist编码的列表对象使用压缩列表作为底层实现，每个压缩列表节点（entry）保存了一个列表元素

   ![image-20220123230816406](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220123230816406.png)

3. linkedlist编码的列表对象使用双端链表作为底层实现，每个双端链表节点（node）都保存了一个字符串对象，而每个字符串对象都保存了一个列表元素。

   ![image-20220123230843720](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220123230843720.png)

4. linkedlist编码的列表对象在底层的双端链表结构中包含了多个字符串对象

5. 完整字符串对象表示：

   ![image-20220123231111927](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220123231111927.png)

6. 编码转换：

   1. 当列表对象可以同时满足以下两个条件时，列表对象使用ziplist编码：（两个条件的上限值是可以修改的）
      1. 列表对象保存的所有字符串元素的长度都小于64字节；
      2. 列表对象保存的元素数量小于512个；不能满足这两个条件的列表对象需要使用linkedlist编码。

7. 列表命令的实现

   ![image-20220123231230877](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220123231230877.png)

#### 8.4 哈希对象

1. 哈希对象的编码可以是ziplist或者hashtable。

2. ziplist编码的哈希对象**使用压缩列表作为底层实现**，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾

3. ziplist 

   ![image-20220124213849925](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124213849925.png)

4. 比如插入键值对

   ![image-20220124213918322](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124213918322.png)

5. hashtable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典键值对来保存：

   1. 字典的每个键都是一个字符串对象，对象中保存了键值对的键；
   2. 字典的每个值都是一个字符串对象，对象中保存了键值对的值

6. hashtable：

   ![image-20220124213954478](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124213954478.png)

7. 当哈希对象可以同时满足以下两个条件时，哈希对象使用ziplist编码，不能满足这两个条件的哈希对象需要使用hashtable编码。（这两个条件的上限值是可以修改的）

   1. 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；
   2. 哈希对象保存的键值对数量小于512个；

8. 哈希命令的实现：

   ![image-20220124214116367](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124214116367.png)

#### 8.5 集合对象

1. 集合对象的编码可以是intset或者hashtable。

2. intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里面。

3. intset：

   ![image-20220124223150876](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124223150876.png)

4. hashtable编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象包含了一个集合元素，而字典的值则全部被设置为NULL

   ![image-20220124223217011](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124223217011.png)

5. 集合对象可以同时满足以下两个条件时，对象使用intset编码（不能满足这两个条件的集合对象需要使用hashtable编码）

   1. 集合对象保存的所有元素都是整数值；
   2. 集合对象保存的元素数量不超过512个。（第二个条件的上限值是可以修改的）

6. 集合命令的实现

   ![image-20220124230815548](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124230815548.png)

#### 8.6 有序集合对象

1. 有序集合的编码可以是ziplist或者skiplist。     

2. ziplist编码的压缩列表对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的**成员**（**member**），而第二个元素则保存元素的**分值**（**score**）

3. ziplist：

   ![image-20220124230941641](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124230941641.png)

4. 压缩列表展示

   ![image-20220124231000340](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220124231000340.png)

5. skiplist编码的有序集合对象使用zset结构作为底层实现，**一个zset结构同时包含一个字典和一个跳跃表**：

   1. zset结构中的zsl跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素：**跳跃表节点的object属性保存了元素的成员，而跳跃表节点的score属性则保存了元素的分值。**

   2. zset结构中的dict字典为有序集合创建了一个从成员到分值的映射，**字典中的每个键值对都保存了一个集合元素：字典的键保存了元素的成员，而字典的值则保存了元素的分值。**

   3. ```c
      typedef struct zset {
          dict *dict;
          zskiplist *zsl;
      } zset;
      ```

   4. 通过这个字典，程序可以用O（1）复杂度查找给定成员的分值

   5. 有序集合每个元素的成员都是一个字符串对象，而每个元素的分值都是一个double类型的浮点数。

   6. 这两种数据结构都会通过指针来共享相同元素的成员和分值，所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值，也不会因此而浪费额外的内存。

6. 当有序集合对象可以同时满足以下两个条件时，对象使用ziplist编码：(不能满足以上两个条件的有序集合对象将使用skiplist编码)(两个条件的上限值是可以修改的)

   1. 有序集合保存的元素数量小于128个；
   2. 有序集合保存的所有元素成员的长度都小于64字节；

7. 有序集合命令的实现

   ![image-20220125000330912](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125000330912.png)

#### 8.7 类型检查与命令多态

1. Redis中用于操作键的命令基本上可以分为两种类型。
   1. 其中一种命令可以对任何类型的键执行，比如说DEL命令、EXPIRE命令、RENAME命令、TYPE命令、OBJECT命令等。
   2. 而另一种命令只能对特定类型的键执行，比如说：
      1. SET、GET、APPEND、STRLEN等命令只能对字符串键执行；
      2. HDEL、HSET、HGET、HLEN等命令只能对哈希键执行；
      3. RPUSH、LPOP、LINSERT、LLEN等命令只能对列表键执行；
      4. SADD、SPOP、SINTER、SCARD等命令只能对集合键执行；
      5. ZADD、ZCARD、ZRANK、ZSCORE等命令只能对有序集合键执行；
2. 在执行一个类型特定的命令之前，Redis会先检查输入键的类型是否正确，然后再决定是否执行给定的命令。
3. 类型特定命令所进行的类型检查是通过redisObject结构的type属性来实现的
4. ![image-20220125000529286](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125000529286.png)
5. **多态命令的实现**
   1. Redis除了会根据值对象的类型来判断键是否能够执行指定命令之外，还会根据值对象的编码方式，选择正确的命令实现代码来执行命令。
      1. 基于类型的多态——一个命令可以同时用于处理多种不同类型的键。DEL、EXPIRE、TYPE
      2. 基于编码的多态——一个命令可以同时用于处理多种不同编码。LLEN
   2. ![image-20220125000714404](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125000714404.png)

#### 8.8 内存回收

1. Redis在自己的对象系统中构建了一个**引用计数（reference counting）技术实现的内存回收机制**，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收。

2. 引用计数信息由redisObject结构的refcount属性记录：

   ```c
   typedef struct redisObject {
   ...
       int refcount;
   ...
   } robj;
   ```

3. ![image-20220125000851962](Redis%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.photo/image-20220125000851962.png)

#### 8.9 对象共享

1. 对象的引用计数属性还带有**对象共享**的作用。
2. 在Redis中，让多个键共享同一个值对象需要执行以下两个步骤：
   1. 将数据库键的值指针指向一个现有的值对象；
   2. 将被共享的值对象的引用计数增一。
3. 目前来说，Redis会在初始化服务器时，创建一万个字符串对象，这些对象包含了从0到9999的所有整数值，当服务器需要用到值为0到9999的字符串对象时，服务器就会使用这些共享对象，而不是新创建对象。
4. 创建共享字符串对象的数量可以通过修改redis.h/REDIS_SHARED_INTEGERS常量来修改。
5. 这些共享对象不单单只有字符串键可以使用，那些在数据结构中嵌套了字符串对象的对象（linkedlist编码的列表对象、hashtable编码的哈希对象、hashtable编码的集合对象，以及zset编码的有序集合对象）都可以使用这些共享对象
6. **受到CPU时间的限制，Redis只对包含整数值的字符串对象进行共享。**

#### 8.10 对象的空转时长

1. redisObject结构包含的最后一个属性为lru属性，该属性记录了对象最后一次被命令程序访问的时间：

   ```c
   typedef struct redisObject {
       unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                               * LFU data (least significant 8 bits frequency
                               * and most significant 16 bits access time). */
   } robj;
   ```

2. OBJECT IDLETIME命令可以打印出给定键的空转时长，这一空转时长就是通过将当前时间减去键的值对象的lru时间计算得出的。OBJECT IDLETIME命令的实现是特殊的，这个命令在访问键的值对象时，不会修改值对象的lru属性。

3. 键的空转时长还有另外一项作用：如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。

## 第二部分 [单机数据库](https://baike.baidu.com/item/单机数据库/1833858)的实现

### 第9章 数据库 

#### 9.1 服务器中的数据库
#### 9.2 切换数据库
#### 9.3 数据库键空间
#### 9.4 设置键的生存时间或过期时间
#### 9.5 过期键删除策略

#### 9.6 Redis的过期键删除策略

#### 9.7 AOF、[RDB](https://baike.baidu.com/item/RDB/1239436)和复制功能对过期键的处理

#### 9.8 数据库通知

### 第10章 RDB持久化

#### 10.1 RDB 文件的创建与载入

#### 10.2 自动间隔性保存

#### 10.3 RDB 文件结构

#### 10.4 分析RDB文件

### 第11章 AOF持久化

#### 11.1 AOF持久化的实现

#### 11.2 AOF文件的载入与数据还原

#### 11.3 AOF重写

### 第12章 事件

#### 12.1 文件事件

#### 12.2 时间事件

#### 12.3 事件的调度与执行

### 第13章 客户端

#### 13.1 客户端属性

#### 13.2 客户端的创建与关闭

### 第14章 服务器

#### 14.1 命令请求的执行过程

#### 14.2 serverCron函数

#### 14.3 初始化服务器

## 第三部分 多机数据库的实现

### 第15章 复制

#### 15.1 旧版复制功能的实现

#### 15.2 旧版复制功能的缺陷

#### 15.3 新版复制功能的实现

#### 15.4 部分重同步的实现

#### 15.5 PSYNC 命令的实现

#### 15.6 复制的实现

#### 15.7 心跳检测

### 第16章 Sentinel

#### 16.1 启动并初始化Sentinel

#### 16.2 获取主服务器信息

#### 16.3 获取从服务器信息

#### 16.4 向主服务器和从服务器发送信息

#### 16.5 接收来自主服务器和从服务器的频道信息

#### 16.6 检测主观下线状态

#### 16.7 检查客观下线状态

#### 16.8 选举领头Sentinel

#### 16.9 故障转移

#### 第17章 集群

### 17.1 节点

#### 17.2 槽指派

#### 17.3 在集群中执行命令

#### 17.4 重新分片

#### 17.5 ASK错误

#### 17.6 复制与故障转移

#### 17.7 消息

## 第四部分 独立功能的实现

### 第18章 发布与订阅

#### 18.1 频道的订阅与退订

#### 18.2 模式的订阅与退订

#### 18.3 发送消息

#### 18.4 查看订阅信息

### 第19章 事务

#### 19.1 事务的实现

#### 19.2 WATCH 命令的实现

#### 19.3 事务的ACID 性质

### 第20章 Lua脚本

#### 20.1 创建并修改Lua 环境

#### 20.2 Lua 环境协作组件

#### 20.3 EVAL命令的实现

#### 20.4 EVALSHA 命令的实现

#### 20.5 脚本管理命令的实现

#### 20.6 脚本复制

### 第21章 排序

#### 21.1 SORT <key> 命令的实现

#### 21.2 ALPHA 选项的实现

#### 21.3 ASC 选项和DESC 选项的实现

#### 21.4 BY选项的实现

#### 21.5 带有ALPHA 选项的BY 选项的实现

#### 21.6 LIMIT 选项的实现

#### 21.7 GET选项的实现

#### 21.8 STORE 选项的实现

#### 21.9 多个选项的执行顺序

### 第22章 [二进制](https://baike.baidu.com/item/二进制/361457)位数组

#### 22.1 位数组的表示

#### 22.2 GETBIT命令的实现

#### 22.3 SETBIT 命令的实现

#### 22.4 BITCOUNT 命令的实现

#### 22.5 BITOP 命令的实现

### 第23章 [慢查询](https://baike.baidu.com/item/慢查询/9200910)日志 378

#### 23.1 慢查询记录的保存

#### 23.2 慢查询日志的阅览和删除

#### 23.3 添加新日志

### 第24章 监视器

#### 24.1 成为监视器

#### 24.2 向监视器发送命令信息