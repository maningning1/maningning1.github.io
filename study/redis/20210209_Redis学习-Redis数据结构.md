# Redis学习-Redis数据结构

## 动态字符串

Redis底层使用SDS（动态字符串）代替C中的String对象，只有在打印日志等无须对字符串值进行修改的地方才会使用C字符串。

### 结构

SDS的结构如下：

```c
struct sdshdr {
    int len;
    int free;
    char buf[];
}
```

+ len：记录buf数组中已使用字节的数量，等于SDS所保存字符串的长度；
+ free：记录buf数组中未使用的字节数量；
+ buf：该值为一个字节数组，用于保存字符串；

SDS遵循了C字符串以空字符结尾的惯例，所以在buf数组中结尾保存了空字符'\0'，且不计入len属性中，如下图所示：

![1.1_SDS示例1](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.1_SDS%E7%A4%BA%E4%BE%8B1.png?raw=true)

![2.2_SDS示例2](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.2_SDS%E7%A4%BA%E4%BE%8B2.png?raw=true)

### 特点

**1.获取字符串长度**

C字符串并不记录自身的长度信息，所以要获取C字符串的长度就必须要遍历整个字符串，所以该操作的时间复杂度为O(N)。而SDS结构内部len属性记录了SDS的长度，所以时间复杂度仅为O(1)。

**2.杜绝缓冲区溢出**

在C字符串中，如果没有对字符串分配足够的空间，在进行拼接操作时就会造成缓冲区溢出。在SDS中进行拼接操作前，会先检查SDS空间是否满足空间要求，如果不满足则会先将SDS空间扩展为执行拼接所需空间大小（下面再介绍空间分配），这样就不会出现缓冲区溢出的情况了。

**3.空间分配策略**

对于C字符串，在每次拼接或截断操作时，都必须重新分配内存防止缓冲区溢出或内存泄漏，但是对于Redis来说，如果频繁重新分配内存会对性能造成影响，所以SDS实现了空间预分配和惰性空间释放这两种优化策略。

+ **空间预分配**

  当SDS**空间不足**时，就会进行扩展，API会对SDS分配额外的空间，防止频繁分配内存操作。

  + 当SDS扩展后的长度小于1MB时，程序就会分配和len相同大小的未使用空间。比如修改后SDS的len变为了10字节，那么程序也会分配10字节的未使用空间，故free属性也变为10；
  + 当SDS扩展后长度大于等于1MB时，程序会分配1MB的未使用空间；

  ![1.3_SDS空间预分配](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.3_SDS%E7%A9%BA%E9%97%B4%E9%A2%84%E5%88%86%E9%85%8D.png?raw=true)

+ **惰性空间释放**

  SDS缩短字符串时，并不会立即重新分配内存回收多出的字节，而是使用free属性记录下来，只有显示调用相应API才会对未使用的空间释放。

**4.二进制安全**

因为C字符串在读入'\0'就认为是字符串的末尾，导致C字符串不能保存二进制数据；而SDS使用len属性的值来判断字符串是否结束，所以其是二进制安全的。

**5.部分兼容C字符串函数**

因为SDS中buf遵循C字符串以空字符结尾的惯例，所以保存文本数据的SDS可以重用一部分<string.h>库定义的函数。

## 链表

### 结构

Redis中的链表是双向链表，采用list来持有链表进行操作，采用listNode结构来标识每个链表节点。

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode;
```

```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表节点数量
    unsigned long len;
    // 节点值复制函数
    void (*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    void (*match)(void *ptr, void *key);
} list;
```

![1.4_链表结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.4_%E9%93%BE%E8%A1%A8%E7%BB%93%E6%9E%84.png?raw=true)

+ listNode中包含prev和next指针，所以为双向链表；
+ list中包含head指针和tail指针，所以获取表头节点和表尾节点的复杂度为O(1)；
+ list中采用len记录list的长度，与SDS相同，空间换时间；
+ dup函数用于复制链表节点所保存的值；
+ free函数用于是否链表节点所保存的值；
+ match函数用于对比链表节点所保存的值和输入值是否相等；

## 字典

### 结构

#### 哈希表

Redis中字典使用哈希表作为底层实现，一个哈希表中有多个哈希表节点，哈希表与哈希表节点结构如下所示：

```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算哈希表节点索引，其值为size-1
    unsigned long sizemask;
    // 哈希表已有节点数量
    unsigned long used;
} dictht;
```

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    // 指向下一个哈希表节点
    struct dictEntry *next;
} dictEntry;
```

![1.5_哈希表结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.5_%E5%93%88%E5%B8%8C%E8%A1%A8%E7%BB%93%E6%9E%84.png?raw=true)

#### 字典

字典结构如下所示，type与privdata属性针对不同类型键值对，为创建多态字典而设置的；ht属性是一个包含两个项的数组，每个项是一个dictht哈希表，字典使用ht[0]，而ht[1]在rehash时会使用。

```c
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void privdata;
    // 哈希表
    dictht ht[2];
    // rehash索引，如果没有进行rehash则值为-1
    int rehashidx;
}
```

![1.6_字典结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.6_%E5%AD%97%E5%85%B8%E7%BB%93%E6%9E%84.png?raw=true)

### 键冲突

Redis中哈希表键冲突和Java中HashMap相同，都是采用链地址法来解决，通过每个哈希表节点的next指针构成单向链表，**但是Redis是采用的头插法**（Java8之后HashMap是采用的尾插法，而Redis是因为没有指向链表表尾的指针，为了速度考虑使用头插法）。

### rehash

哈希表中保存的键值对增加，就会导致链表过长而使得查询速度变慢，键值对减少就会导致dictEntry的空槽过多浪费空间，所以需要rehash（重新散列）来维持哈希表的**负载因子（哈希表保存的节点数量/哈希表大小）**。rehash的步骤为：

+ 1.为字典的ht[1]哈希表分配空间，如果是扩展操作，ht[1]的大小为第一个大于等于ht[0].used*2的2^n；如果是收缩操作，ht[1]的大小为第一个大于等于ht[0].used的2^n（used属性是保存的哈希表节点个数）；
+ 2.将保存在ht[0]中的所有键值对重新计算索引然后放置到ht[1]上；
+ 释放ht[0]，并将ht[1]设置为ht[0]，并在ht[1]上新创建一个空白哈希表；

### 渐进式rehash

rehash动作不是一次性完成的，而是分多次渐进式地完成的，当开始rehash时，会将字典中rehashidx值设置为0，此时字典的操作就会在ht[0]和ht[1]上同时进行，直到ht[0]上所有键值对都rehash至ht[1]后，程序将rehashidx设置为-1，表示rehash操作完成。

## 跳跃表

### 简介

跳跃表又称跳表，它通过在每个节点中维持多个指向其他节点的指针来达到快速访问的目的，其平均O(logN)、最坏O(N)的复杂度可以和平衡树媲美，并且其实现比平衡树更加简单。

跳表的实现是根据有序链表的基础发展起来的，对于一个有序链表，如果要查找某个节点，时间复杂度为O(n)，假如我们在每两个节点间新增指针连接成一个新的链表，节点就为原来的一半，也会减少查询时间。

![1.7跳跃表结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.7%E8%B7%B3%E8%B7%83%E8%A1%A8%E7%BB%93%E6%9E%84.png?raw=true)

如上图所示，查询操作就可以从最上层链表开始查找，时间复杂度就可以降低到O(log n)。

对于插入操作，为了防止跳表退化为链表，插入操作会每个节点随机出一个层数(level)，这样插入操作的性能就会明显优于平衡树。随机层数计算的伪代码如下，所以对于跳表主要关注下一层是否有指针的概率p和运行的最大层数maxLevel。

```java
int randomLevel() {
	level = 1;
    // random()返回一个[0...1)的随机数
    while (random() < p && level < MaxLevel)
        level++;
    return level;
}   
```

### 结构

Redis中跳表的概率p取1/4，maxLevel取32层，结构如下所示：

```c
typedef struct zskiplistNode {
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分数值
    double score;
    // 成员对象
    robj *obj;
} zskiplistNode;
```

```c
typedef struct zskiplist {
    // 表头、表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点数量
    unsigned long length;
    // 层数最大节点的层数
    int level;
} zskiplist;
```

其是一个链表，使用zsiplist记录整个链表的信息，表头指向一个32层的链表节点，其他的每个链表节点会记录各个节点的对象，分值，各层的信息等，其中level[]中保存每一层的前进指针及指针跨度，结构如下图所示：

![1.8_Redis中的跳跃表结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.8_Redis%E4%B8%AD%E7%9A%84%E8%B7%B3%E8%B7%83%E8%A1%A8%E7%BB%93%E6%9E%84.png?raw=true)

## 整数集合

### 结构

整数集合是用于保存整数的数据结构，其结构如下所示：

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

length保存元素数量，contents按从小到达顺序保存集合中的元素，虽然contents属性声明为int8_t，但是其真正类型取决于encoding属性的值，这样可以节约内存。

+ encoding为INTSET_ENC_INT16，contents就是一个int16_t类型的数组（-32768~32767）；
+ encoding为INTSET_ENC_INT32，contents就是一个int32_t类型的数组（-2147483648~2147483647）；
+ encoding为INTSET_ENC_INT64，contents就是一个int64_t类型的数组（-9223372036854775808~9223372036854775807）；

### 升级与降级

+ **升级**

  如果我们添加新元素到整数集合中，且新元素类型比当前类型要大时，就需要升级，就要将encoding属性的值更改，并调整原理元素的位置（因为升级后每个元素占用内存增大）。

+ **降级**

  整数集合不支持降级操作，所以一旦升级后就会一直保持升级后的编码。

## 压缩列表

### 结构

压缩列表数由一系列特殊编码的连续内存块组成的顺序性数据结构，其包含多个节点，每个节点可以保存一个字节数组或一个整数值。

![1.9_压缩列表](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.9_%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8.png?raw=true)

+ zlbytes：记录整个压缩列表占用的内存字节数；
+ zltail：记录压缩列表尾结点距其实地址字节数，可以通过该偏移量直接定位尾结点地址；
+ zllen：节点数量，当值小于UINT16_MAX（65535）时，取该值；当该值等于UINT16_MAX时必须遍历才能求出节点数量；
+ zlend：特殊值0xFF，标记压缩列表末端；

压缩列表节点可以用于保存一个字节数组或者一个整数值，结构如下所示：

![1.10_压缩列表节点](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/1.10_%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8%E8%8A%82%E7%82%B9.png?raw=true)

+ previous_entry_length：记录压缩列表中前一个节点的长度，根据该值可以得到前一节点的地址（相当于链表中的向前指针），这样程序就可以根据该值一直向前节点回溯；

  + 如果前一节点长度小于254字节，则该属性长度为1字节，前一节点长度就保存在这个字节中；
  + 如果前一节点长度大于等于254字节，则该属性长度为5字节，第一个字节设置为0xFE，后4个字节用于保存前一节点的长度；

+ encoding：记录节点content中保存的数据类型及长度；

  + 该值一字节长，且以11开头的是指整数编码，对应关系如下表所示；

    | 编码     | 编码长度 | content属性保存的值 |
    | -------- | -------- | ------------------- |
    | 11000000 | 1字节    | int16_t             |
    | 11010000 | 1字节    | int32_t             |
    | 11100000 | 1字节    | int64_t             |
    | 11110000 | 1字节    | 24位有符号整数      |
    | 11111110 | 1字节    | 8位有符号整数       |

  + 一字节、两字节、五字节长，且以00/01/10开头的是字节数组编码，数组长度由编码除去最高两位之后的其他位记录；

+ content：保存节点的值；

### 连锁更新问题

因为每个entry中保存的是**变长编码**，假设一个压缩列表中，其包含e1、e2、e3、e4…..，e1节点的大小为253字节，那么e2.prevrawlen的大小为1字节，如果此时在e2与e1之间插入了一个新节点e_new，e_new编码后的整体长度（包含e1的长度）为**254字节**，此时e2.prevrawlen就需要扩充为**5字节**；如果e2的整体长度变化又引起了e3.prevrawlen的存储长度变化，那么e3也需要扩…….如此递归直到表尾节点或者某一个节点的prevrawlen本身长度可以容纳前一个节点的变化。**其中每一次扩充都需要进行空间再分配操作**。删除节点亦是如此，只要引起了操作节点之后的节点的prevrawlen的变化，都可能引起连锁更新。

## quicklist

### 结构

quicklist是一个由ziplist组成的双向链表，每个节点使用ziplist来保存数据，本质来说quicklist是保存着ziplist的双向链表（linkedlist）。



![1.11_quicklist结构](D:\study_note\maningning1.github.io\images\redis\1.11_quicklist结构.png)

```c
typedef struct quicklistNode {
    struct quicklistNode *prev; //上一个node节点
    struct quicklistNode *next; //下一个node
    unsigned char *zl;            //保存的数据 压缩前ziplist 压缩后压缩的数据
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

linkedlist和ziplist有如下特性，而quicklist则是他们的一个折中方案，结合了他们的优点：

+ linkedlist：插入复杂度低，但其内存开销大，且地址不连续，容易产生内存碎片；
+ ziplist：存储在一段连续的内存上，所以存储效率高，但是修改、插入、删除操作就会频繁申请和释放内存，所以ziplist很长时会转换为linkedlist。

### 参数

在quicklist中，相同的存储项可以包含在不同的结构中（比如存储12个数据项，可以每个quicklist包含3个节点，每个ziplist包含4个数据项；也可以是每个quicklist包含4个节点，每个ziplist包含3个数据项），所以在quicklist中有如下特点：

+ 每个quicklist节点的ziplist越短，则内存碎片越多，这样就可能会退化为一个linkedlist；
+ 每个quicklist节点的ziplist越长，则分配大块连续内存空间的难度就越大，这样就可能退化成一个ziplist；

所以，在quicklist中redis提供了`list-max-ziplist-size`和`list-compress-depth`来根据情况调整。

+ `list-max-ziplist-size`
  + 当取正值时代限定每个quicklist节点上ziplist的长度，当其为4时表示每个quicklistNode节点上最多只能有4个ziplist；
  + 当取负数时只能取-1~-5这五个值：
    + -5：每个quicklist节点上的ziplist大小不能超过64Kb；
    + -4：每个quicklist节点上的ziplist大小不能超过32Kb；
    + -3：每个quicklist节点上的ziplist大小不能超过16Kb；
    + -2：每个quicklist节点上的ziplist大小不能超过8Kb；
    + -1：每个quicklist节点上的ziplist大小不能超过4Kb；

+ `list-compress-depth`

  当列表很长时，中间数据被访问的频率就会比较低，该参数可以将中间数据节点进行压缩。

  + 0：不压缩，redis的默认值；

  + 1：表示quicklist两端各有一个节点不压缩；

  + 2：表示quicklist两端各有两个节点不压缩；

    ...

  + n：表示quicklist两端各有n个节点不压缩；