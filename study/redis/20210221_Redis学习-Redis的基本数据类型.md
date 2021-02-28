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

![2.01_raw编码的内存结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.01_raw%E7%BC%96%E7%A0%81%E7%9A%84%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84.png?raw=true)

embstr编码的字符串对象结构如下所示：

![2.02_embstr编码的内存结构](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.02_embstr%E7%BC%96%E7%A0%81%E7%9A%84%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84.png?raw=true)

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

![2.03_字符串命令](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.03_%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%91%BD%E4%BB%A4.png?raw=true)

## 列表对象

### 编码

在redis3.2之前，列表对象的编码可以是ziplist、linkedlist；而在redis3.2之后，列表对象的底层全部都由quicklist实现。所以本文先介绍在redis3.2之前列表对象的基本属性。

当创建一个列表对象后，

```shell
127.0.0.1:6379> rpush list_obj 1 "three" 5
(integer) 3
```

ziplist编码的列表对象结构如下所示：

![2.04_ziplist编码的列表对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.04_ziplist%E7%BC%96%E7%A0%81%E7%9A%84%E5%88%97%E8%A1%A8%E5%AF%B9%E8%B1%A1.png?raw=true)

linkedlist编码的列表对象结构如下所示：

![2.05_linkedlist编码的列表对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.05_linkedlist%E7%BC%96%E7%A0%81%E7%9A%84%E5%88%97%E8%A1%A8%E5%AF%B9%E8%B1%A1.png?raw=true)

其中，在linkedlist编码中每个链表的节点都是保存了一个字符串对象，字符串对象也是redis五种基本类型对象中唯一一个可以被其他对象嵌套的对象。

### 编码转换

当列表对象同时满足以下两个条件时，使用ziplist编码：

+ 列表对象报错的所有字符串元素长度都小于64字节；
+ 列表对象报错的元素数量小于512个；

**注意**：可以通过`list-max-ziplist-value`和`list-max-ziplist-entries`参数修改。

### 列表命令

![2.06_列表命令](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.06_%E5%88%97%E8%A1%A8%E5%91%BD%E4%BB%A4.png?raw=true)

## 哈希对象

### 编码

哈希对象的编码可以是ziplist或者hashtable，当创建一个哈希对象后

```shell
127.0.0.1:6379> hmset hash_obj name "Tom" age 25 career "Programmer"
OK
```

ziplist结构如下所示：

![2.07_ziplist编码的哈希对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.07_ziplist%E7%BC%96%E7%A0%81%E7%9A%84%E5%93%88%E5%B8%8C%E5%AF%B9%E8%B1%A1.png?raw=true)

hashtable结构如下所示：

![2.08_hashtable编码的哈希对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.08_hashtable%E7%BC%96%E7%A0%81%E7%9A%84%E5%93%88%E5%B8%8C%E5%AF%B9%E8%B1%A1.png?raw=true)

### 编码转换

当哈希对象同时满足以下两个条件时，哈希对象使用ziplist编码：

+ 哈希对象报错的所有键值对的键和值长度都小于64字节；
+ 哈希对象报错的键值对数量小于512个；

**注意**：可以通过`hash-max-ziplist-value`和`hash-max-ziplist-entries`参数修改。

```shell
127.0.0.1:6379> hmset hash_obj name "Tom" age 25 career "Programmer"
OK
127.0.0.1:6379> object encoding hash_obj
"ziplist"
127.0.0.1:6379> hset hash_obj name xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
(integer) 0
127.0.0.1:6379> object encoding hash_obj
"hashtable"
```

### 哈希命令

![2.09_哈希命令](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.09_%E5%93%88%E5%B8%8C%E5%91%BD%E4%BB%A4.png?raw=true)

## 集合对象

### 编码

集合对象的编码可以是intset或者hashtable，如下所示：

```shell
127.0.0.1:6379> sadd set_obj "apple" "banana" "cherry"
(integer) 3
127.0.0.1:6379> object encoding set_obj
"hashtable"
127.0.0.1:6379> sadd numbers 1 3 5
(integer) 3
127.0.0.1:6379> object encoding numbers
"intset"
```

intset编码结构如下所示：

![2.10_intset编码的集合对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.10_intset%E7%BC%96%E7%A0%81%E7%9A%84%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png?raw=true)

hashtable编码结构如下所示：

![2.11_hashtable编码的集合对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.11_hashtable%E7%BC%96%E7%A0%81%E7%9A%84%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png?raw=true)

### 编码转换

当集合对象同时满足以下两个条件时，采用intset编码：

+ 集合对象保存的所有元素都是整数值；
+ 集合对象保存元素数量不超过512个；

**注意**：可以通过`set-max-intset-value`参数修改第二个参数。

### 集合命令

![2.12_集合命令](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.12_%E9%9B%86%E5%90%88%E5%91%BD%E4%BB%A4.png?raw=true)

## 有序集合对象

### 编码

有序集合对象的编码可以是ziplist或者skiplist。

```shell
127.0.0.1:6379> zadd price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3
```

如果使用ziplist编码作为底层实现，则集合中每个元素使用紧挨在一起的压缩列表节点来保存，第一个节点保存元素成员，第二个保存元素分值，并且压缩列表内集合元素会按照分值从小到大进行排序，结构如下所示：

![2.13_ziplist编码的有序集合对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.13_ziplist%E7%BC%96%E7%A0%81%E7%9A%84%E6%9C%89%E5%BA%8F%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png?raw=true)

而如果使用skiplist编码作为底层实现，在跳表中也是按照分值从小到大保存了所有集合元素，跳表节点的object属性保存元素成员，跳表节点score属性保存元素的分值；除此之外，结构中还使用了dict字典来保存有序集合成员到分值的映射，这样可以用O(1)的复杂度查找给定成员的分值，而且两种数据结构都会通过指针共享相同元素的成员和分值，不会浪费额外的内存，其结构如下所示（图中成员和分值是共享的数据）：

![2.14_skiplist编码的有序集合对象](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.14_skiplist%E7%BC%96%E7%A0%81%E7%9A%84%E6%9C%89%E5%BA%8F%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png?raw=true)

### 编码转换

当有序集合对象同时满足以下两个条件时，采用ziplist编码：

+ 有序集合对象保存的元素数量小于128个；
+ 有序集合报错的所有元素成员长度小于64字节；

**注意**：可以通过`zset-max-ziplist-value`和`zset-max-ziplist-entries`参数修改。

### 有序集合命令

![2.15_有序集合命令](https://github.com/maningning1/maningning1.github.io/blob/main/images/redis/2.15_%E6%9C%89%E5%BA%8F%E9%9B%86%E5%90%88%E5%91%BD%E4%BB%A4.png?raw=true)