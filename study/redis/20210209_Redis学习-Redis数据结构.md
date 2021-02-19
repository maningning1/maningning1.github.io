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

![1.1_SDS示例1](D:\study_note\maningning1.github.io\images\redis\1.1_SDS示例1.png)

![2.2_SDS示例2](D:\study_note\maningning1.github.io\images\redis\2.2_SDS示例2.png)

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

  ![1.3_SDS空间预分配](D:\study_note\maningning1.github.io\images\redis\1.3_SDS空间预分配.png)

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

![1.4_链表结构](D:\study_note\maningning1.github.io\images\redis\1.4_链表结构.png)

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

![1.5_哈希表结构](D:\study_note\maningning1.github.io\images\redis\1.5_哈希表结构.png)

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

![1.6_字典结构](D:\study_note\maningning1.github.io\images\redis\1.6_字典结构.png)

### 键冲突

Redis中哈希表键冲突和Java中HashMap相同，都是采用链地址法来解决，通过每个哈希表节点的next指针构成单向链表，**但是Redis是采用的头插法**（Java8之后HashMap是采用的尾插法，而Redis是因为没有指向链表表尾的指针，为了速度考虑使用头插法）。

### rehash

哈希表中保存的键值对增加，就会导致链表过长而使得查询速度变慢，键值对减少就会导致dictEntry的空槽过多浪费空间，所以需要rehash（重新散列）来维持哈希表的负载因子。

## 跳跃表

## 整数集合

## 压缩列表