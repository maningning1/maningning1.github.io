# Redis学习-Redis的基本数据类型

Redis对外暴露5中数据类型，分别是string、list、hash、set和zset，每种数据类型下对应多种数据结构的封装，Redis会根据不同的使用场景为对象选取不同的数据结构实现，从而优化对象在某一场景下的效率。

## 对象

Redis使用对象来表示数据库中的键和值，每当我们新创建一个键值对时，至少会创建两个对象，一个用于键，一个用于值，如下所示，键创建了一个包含字符串值“msg”的对象，值创建了一个包含字符串值“hello”的对象。

```shell
127.0.0.1:6379> set msg "hello"
OK
```

Redis中对象由redisObject表示，其中键总是一个字符串对象，而值可以是字符串、列表哈希、集合、有序集合等。

```c
typedef struct redisObject {
    // 对象的类型，字符串/列表/集合/哈希表
    unsigned type:4;
    // 未使用的两个位
    unsigned notused:2; /* Not used */
    // 编码的方式，底层数据结构
    unsigned encoding:4;
    // 当内存紧张，淘汰数据的时候用到
    unsigned lru:22; /* lru time (relative to server.lruclock) */
    // 引用计数
    int refcount;
    // 数据指针
    void *ptr;
}
```

### 数据类型

redisObject的type决定了对象的类型，在shell中可以通过`TYPE`命令查看，如下所示：

```c
127.0.0.1:6379> set msg "hello"
OK
127.0.0.1:6379> TYPE msg
string
```

| 对象         | type属性值   | TYPE命令输出 |
| ------------ | ------------ | ------------ |
| 字符串对象   | REDIS_STRING | string       |
| 列表对象     | REDIS_LIST   | list         |
| 哈希对象     | REDIS_HASH   | hash         |
| 集合对象     | REDIS_SET    | set          |
| 有序集合对象 | REDIS_ZSET   | zset         |

### 编码

在Redis中，使用`OBJECT ENCODING`命令可以查看对象的相应编码，在Redis中，编码对应的底层数据结构关系如下表所示：

```shell
127.0.0.1:6379> set msg "hello"
OK
127.0.0.1:6379> OBJECT ENCODING msg
"embstr"
```

| OBJECT ENCODING命令输出 | 编码                      | 底层数据结构                      |
| ----------------------- | ------------------------- | --------------------------------- |
| "int"                   | REDIS_ENCODING_INT        | 整数                              |
| "embstr"                | REDIS_ENCODING_EMBSTR     | embstr编码的简单动态字符串（SDS） |
| "raw"                   | REDIS_ENCODING_RAW        | 简单动态字符串                    |
| "hashtable"             | REDIS_ENCODING_HT         | 字典                              |
| "linkedlist"            | REDIS_ENCODING_LINKEDLIST | 双端链表                          |
| "ziplist"               | REDIS_ENCODING_ZIPLIST    | 压缩列表                          |
| "intset"                | REDIS_ENCODING_INTSET     | 整数集合                          |
| "skiplist"              | REDIS_ENCODING_SKIPLIST   | 跳跃表和字典                      |

## 字符串对象

### 编码

字符串对象的编码可以是int、raw、embstr。

```shell
127.0.0.1:6379> set str_obj 1
OK
127.0.0.1:6379> OBJECT ENCODING str_obj
"int"
127.0.0.1:6379> set str_obj "hello"
OK
127.0.0.1:6379> OBJECT ENCODING str_obj
"embstr"
127.0.0.1:6379> set str_obj "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
OK
127.0.0.1:6379> OBJECT ENCODING str_obj
"raw"
```

+ int：当字符串对象中保存的都是整数值时，采用int编码保存；
+ embstr：其是一种保存短字符串的一种优化编码，当对象中字符串长度小于44字节（redis3.2版本前为39字节）时，采用embstr保存；
+ raw：当对象中字符串长度大于44字节（redis3.2版本前为39字节）时，采用raw保存；

### raw与embstr的区别

raw编码的字符串对象结构如下所示：

![2.01_raw编码的内存结构](D:\study_note\maningning1.github.io\images\redis\2.01_raw编码的内存结构.png)

embstr编码的字符串对象结构如下所示：

![2.02_embstr编码的内存结构](D:\study_note\maningning1.github.io\images\redis\2.02_embstr编码的内存结构.png)

embstr编码的字符串对象采用一块连续的内存空间保存redisObject对象和sdshdr对象，故只需要分配一次内存空间，而raw编码的字符串对象需要调用两次内存分配函数。但是embstr每次重新分配空间时整个redisObject和sds都需要重新分配，故redis的embstr实现为只读。

### 编码转换

+ 当int编码保存的值不再是整数，或者大小超过long的范围，则会自动转换为raw；
+ redis中对应浮点数类型也是作为字符串保存，在进行计算时将其转换为浮点数类型；
+ 因为embstr是只读的，所以对其进行修改后，都会转换为raw对象，无论是否达到44字节；

```shell
127.0.0.1:6379> set pi 3.14
OK
127.0.0.1:6379> OBJECT ENCODING pi
"embstr"
127.0.0.1:6379> set num 100
OK
127.0.0.1:6379> OBJECT ENCODING num
"int"
127.0.0.1:6379> append num " hello"
(integer) 9
127.0.0.1:6379> get num
"100 hello"
127.0.0.1:6379> OBJECT ENCODING num
"raw"
127.0.0.1:6379> set msg "hello"
OK
127.0.0.1:6379> OBJECT ENCODING msg
"embstr"
127.0.0.1:6379> append msg " redis"
(integer) 11
127.0.0.1:6379> OBJECT ENCODING msg
"raw"
```

### 字符串命令

![2.03_字符串命令](D:\study_note\maningning1.github.io\images\redis\2.03_字符串命令.png)

## 列表对象

### 编码

在redis3.2之前，列表对象的编码可以是ziplist、linkedlist；而在redis3.2之后，列表对象的底层全部都由quicklist实现。所以本文先介绍在redis3.2之前列表对象的基本属性。

当创建一个列表对象后，

```shell
127.0.0.1:6379> rpush list_obj 1 "three" 5
(integer) 3
```

ziplist编码的列表对象结构如下所示：

![2.04_ziplist编码的列表对象](D:\study_note\maningning1.github.io\images\redis\2.04_ziplist编码的列表对象.png)

linkedlist编码的列表对象结构如下所示：

![2.05_linkedlist编码的列表对象](D:\study_note\maningning1.github.io\images\redis\2.05_linkedlist编码的列表对象.png)

其中，在linkedlist编码中每个链表的节点都是保存了一个字符串对象，字符串对象也是redis五种基本类型对象中唯一一个可以被其他对象嵌套的对象。

### 编码转换

当列表对象同时满足以下两个条件时，使用ziplist编码：

+ 列表对象报错的所有字符串元素长度都小于64字节；
+ 列表对象报错的元素数量小于512个；

**注意**：可以通过`list-max-ziplist-value`和`list-max-ziplist-entries`参数修改。

### 列表命令

![2.06_列表命令](D:\study_note\maningning1.github.io\images\redis\2.06_列表命令.png)

## 哈希对象

### 编码

哈希对象的编码可以是ziplist或者hashtable