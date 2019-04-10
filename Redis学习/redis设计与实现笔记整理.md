## 对象类型和编码

Redis使用对象来表示数据库中的键和值，每次我们在Redis数据库中创建一个键值对的时候，至少会创建两个对象，一个作为键，一个作为值。



Redis中每个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性，encoding属性和pre属性：

```c
typedef struct redisObject{
    //类型
    unsigned type:4;
    //编码
    unsigned enconding:4;
    //指向底层实现数据结构的指针
    void *ptr;
    //引用计数
    int refCount;
    //空转时间
    unsigned lru;，
}robj
```



### 1.类型

​									表1. 类型常量

| 类型常量     | 对象的名称   |
| ------------ | ------------ |
| REDIS_STRING | 字符串对象   |
| REDIS_LIST   | 列表对象     |
| REDIS_HASH   | 哈希对象     |
| REDIS_SET    | 集合对象     |
| REDIS_ZSET   | 有序集合对象 |

通过`TYPE KEY`可以查看数据库键对应的值的对象类型



 ### 2.编码和底层实现

通过`OBJECT ENCODING key`命令可以查看数据库键的值对象的编码

​									表2.对象的编码常量

| 对象所使用的底层数据结构          | 编码常量                  | OBJECT ENCODING命令输出 |
| --------------------------------- | ------------------------- | ----------------------- |
| 整数                              | REDIS_ENCODING            | int                     |
| embstr编码的简单动态字符串（SDS） | REDIS_ENCODING_EMBSTR     | embstr                  |
| 简单动态字符串                    | REDIS_ENCODING_RAW        | raw                     |
| 字典                              | REDIS_ENCODING_HT         | hashtable               |
| 双端链表                          | REDIS_ENCODING_LINKEDLIST | linkedlist              |
| 压缩列表                          | REDIS_ENCODING_ZIPLIST    | ziplist                 |
| 整数集合                          | REDIS_ENCODING_INTSET     | intset                  |
| 跳表                              | REDIS_ENCONDING_SKIPLIST  | skiplist                |

​								表3.不同类型和编码的对象

| 类型   | 编码                |
| ------ | ------------------- |
| string | int、embstr、raw    |
| list   | ziplist、linkedlist |
| hash   | ziplist、hashtable  |
| set    | intset、hashtable   |
| zset   | ziplist、skiplist   |



###3.数据结构分析

####3.1 简单动态字符串SDS

Redis没有直接使用C语言的字符串表示，而是自己构建了一种名为简单动态字符串（SDS）的抽象类型，作为Redis的默认字符串表示。

SDS的定义：

```c
struct sdshdr{
    //记录buf数组已经使用的字节长度
    int len;
    //记录buf数组中未使用的字节数量
    int free；
    //字节数组，用于保存字符串
    char buf[];
};
```

为什么不用C语言提供的字符串？

| C字符串                              | SDS                                                          |
| ------------------------------------ | ------------------------------------------------------------ |
| 获取字符串长度时间复杂度O(n)         | 获取字符串长度的复杂度为O（1）                               |
| API不是安全的，可能会造成缓冲区溢出  | API通过free来判断剩余空间，不够就扩容，不会造成缓冲区溢出    |
| 修改字符串长度需要频繁执行内存重分配 | 通过空间预分配和惰性空间释放，减少了内存重分配               |
| 只保存文本数据，不是二进制安全的     | 以处理二进制的方法来处理SDS存放在buff数组中的数据，不会对数据进行任何的限制、过滤等，数据写入是什么样的，输出就是什么样的，二进制安全的 |
|                                      | 因为buff最后一个元素是’/0‘，可以使用一部分<string.h>库中的函数 |

#### 3.2 链表linkedlist

Redis链表的实现是一个无环的双端队列，而且带有头指针和尾指针和链表长度

```c
typedef struct listNode{
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    
    //节点值
    void *value;
}listNode;

typedef struct list{
    //表头节点
    listNode *head;
    //表尾节点
    listNode *tail;
    //链表所包含的节点数量
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    
    //节点值释放函数
    void (*free)(void *ptr);
    
    //节点值对比函数
    int (*match)(void *ptr,void *key);
    
}list;
```



#### 3.3 字典

Redis的字典的底层使用了哈希表，哈希表的实现和Java的HashMap很相似，也是使用的链地址法来处理冲突

```c

//哈希表
typedef struct dictht{
    //哈希表数组
    dictEntry **table;
    //哈希表大小
	unsigned long size;
    //哈希表大小掩码，用于计算索引值，为size-1
    unsigned long sizemask;
    //该哈希表已有节点的数量
    unsigned long used;
}dictht;


typedef struct dictEntry{
    //键
    void *key;
    //值
    union{
        void *val;
        uint64_t u64;
        int64_t s64;
    }v;
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
}dictEntry;

//字典
typedef struct dict{
    //类型特定的函数，保存了一簇用于操作特定类型键值对的函数
    dictType *type;
    //私有数据，保存了需要传给那些类型特定函数的可选参数
    void *privdata;
    //哈希表，这里两个哈希表，一个是平时使用的，一个是rehash时用的
    dicht ht[2];
    
    //rehash索引
    //当rehash不在进行时，值为-1
    int trehashindx;
}dict;
```

**哈希算法：**

```c
//使用字典设置的hash函数计算key的哈希值
hash = dict->type->hashFunction(key);
//根据情况不同，x可以是0或1
index = hash&dict->ht[x].sizemask;
```

**rehash：**
步骤和hashMap很像，大体就是新开辟一个散列桶，把旧桶的数据rehash到新桶，然后旧桶释放，使用新桶，如下：

1）为字典的ht[1]分配空间，如果是扩容，ht[1]为第一个大于等于ht[0].used*2的2的n次幂；如果缩容，那么ht[1]的大小为第一次大于等于ht[0].used的2的n次幂。

2）将ht[0]中的所有键值对rehash到ht[1]

3）rehash完成后将ht[0]释放，将ht[1]设置为ht[0],并在ht[1]创建一个空白哈希表，为下一次rehash做准备。



**resize时机：**
无非就是负载因子到达阈值了，进行扩容或者缩容，负载因子=ht[0].used/ht[0].size;

当满足下面条件是，自动扩容：

1）服务器目前没有执行BGSAVE或BGREWRITEAOF命令，并且负载因子大于1

2）服务器目前正在执行BGSAVE或BGREWRITEAOF命令，并且负载因子大于5

通过BGSAVE或BGREWRITEAOF是否正在执行，负载因子阈值不同，是因为在进行这些命令的时候Redis会创建子进程，而大多数操作系统使用写时复制的技术来优化子进程的使用效率，所以在子进程存在期间，服务器会提高扩容的负载因子阈值，从而尽可能的避免在子进程存在期间进行哈希表扩展操作，这可以避免不必要的内存写入操作，最大限度地节约内存。



**渐进式rehash：**

和hashMap的rehash不一样的一点是，Redis的rehash动作不是一次性、集中式地完成的，而是分多次、渐进式地完成的。

具体做法如下：
1）为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表

2）在字典中维持一个索引计算器变量rehashidx，并将它的值设为0，表示rehash正式开始

3）在rehash期间，每次对字典的增删查改，操作这两个哈希表，程序除了执行指定的操作外，还会将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash完成后，rehasidx的值加1。

4）随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值都会被rehash到ht[1]。这是rehashidx的值变成-1，rehash完成。

渐进式rehash的好处是采取了分治的方式，避免了集中rehash带来的庞大数据量



#### 3.4 跳表skiplist

跳表支持平均O（logN）、最坏O（N）的复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

**跳表由zskiplistNode和zskiplist两个结构定义，它是通过幂次定律来随机生成一个1-32的值来表示每个节点的层数，每个节点有level个指针，每一层指向和它同一层的下一次节点，通过这些层来加速访问其他节点。跳表中还有一个头节点，不存储实际信息，用来作为遍历的起点，头节点的层数等于其他节点中最大的层数。当我们要在跳表中找一个数值的时候，从头节点的最高层开始，先下一个节点需找，如果下一个节点比要找的值大，就向下一层移动，否则都向下一个节点移动，直到找到为止。查找的时候平均时间复杂度是O（logn）**

```c
//跳表节点
typedef struct zskiplistNode{
    //后退指针
    struct zskiplistNode *backward;
    //分值
    double score;
    //成员对象,指向一个字符串对象，对象中存着一个SDS
    robj *obj;
        
    //层
    struct zskiplistLevel{
        //前进指针
        struct zskiplistNode *forword;
        
        //跨度，记录到下一个节点跨越的节点数，
        //这个和跳表的实现没什么关系，不过通过跨度和我们能直接得到所找节点的排名
        unsigned int span;
    }level[];
}zskiplistNode;

//跳表
typedef struct zskiplist{
    //表头和表尾节点
    struct skiplistNode *header,*tail;
    //表中节点的数量
    unsigned long length;
    //表中层数最大的节点的层数
    int level;
}zskiplist;

```

#### 3.5 整数集合intset

当set的元素只包含整数而且数量不多时，使用intset来作为set的底层

```c
typedef struct intset{
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组,各元素从小到大排序，不存在重复元素
    int8_t contents[];
}intset;
```

contents中的元素编码由encoding决定，等于而且所有元素中编码所占空间最大的，如果元素有一个是INTSET_ENC_INT64的话，那么contents中所有元素的编码都会变成INTSET_ENC_INT64。

**升级：**

当我们要将一个新元素添加到整数集合中，而且新元素的类型比整数集合现有的所有元素都大，那么需要对intset进行升级

步骤如下：

1)根据新元素的类型，扩展整数集合底层数组空间的大小，并为新元素重新分配空间

2)将旧元素的类型扩展到和新元素一样，然后放到数组中正确的位置上去，最后把新元素也放上去。

**升级的好处：**

1）提高灵活性，自动适应元素的类型

2）节省内存空间，不用一开始就使用64的空间去保存元素，直到遇到类型更大的我们才升级

**降级：**

intset不支持降级。



#### 3.6 压缩链表 ziplist

当一个列表键只包含少量列表项，并且每个列表项要么就是小整数，要么就是长度比较短的字符串，Redis就会使用ziplist来做为list、hash、zset底层实现。

压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的的顺序型数据结构。一个压缩列表可以包含任意多个节点，每个节点保存一个字节数组或者一个整数值



**压缩列表的结构：**

| zlbytes | zltail | zllen | entry1 | entry2 | ...  | entryN | zlend |
| ------- | ------ | ----- | ------ | ------ | ---- | ------ | ----- |
| 4字节   | 4字节  | 2字节 | 不定   |        |      |        | 1字节 |

zlbytes: 记录整个压缩列表占用的内存字节数

zltail ：记录压缩列表表尾节点到压缩列表起始地址的偏移，这样无需遍历整个压缩列表就可以确定表尾的地址

zllen：记录压缩列表节点数量

entryX ：压缩列表的各个节点

zlend ： 特殊值‘ 0xFF’用于标记压缩列表的未端



**压缩列表节点：**

| previous_entry_length                        | encoding | content |
| -------------------------------------------- | -------- | ------- |
| 前一个节点的长度，通过它可以实现从后往前遍历 | 编码     | 内容    |

当前一股节点的长度小于254字节，previous_entry_length  就是一个字节，大于等于254字节就使用5字节来保存previous_entry_length  。

**连锁更新：**

如果压缩列表中有多个介于250-253之间的节点，记录这些节点的previous_entry_length的长度为1字节，如果我们插入一个大于等于254字节的新节点，那么后面的节点都大于254字节了，这时候会导致后面的节点一个接一个的连锁更新。

但是这种情况不常见，而且节点数量也不多，所以不用担心会影响性能。



##服务器