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

