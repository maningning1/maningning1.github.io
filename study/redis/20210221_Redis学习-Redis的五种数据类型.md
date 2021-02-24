# Redis学习-Redis的五种数据类型

Redis对外暴露5中数据类型，分别是string、list、hash、set和zset，每种数据类型下对应多种数据结构的封装，Redis会根据不同的使用场景为对象选取不同的数据结构实现，从而优化对象在某一场景下的效率。

## 编码

在Redis中，使用`OBJECT ENCODING`命令可以查看对象的相应编码，在Redis中，编码对应的底层数据结构关系如下表所示：

```shell
127.0.0.1:6379> set msg "hello"
OK
127.0.0.1:6379> OBJECT ENCODING msg
"embstr"
```

| OBJECT ENCODING命令输出 | 底层数据结构                      |
| ----------------------- | --------------------------------- |
| "int"                   | 整数                              |
| "embstr"                | embstr编码的简单动态字符串（SDS） |
| "raw"                   | 简单动态字符串                    |
| "hashtable"             | 字典                              |
| "linkedlist"            | 双端链表                          |
| "ziplist"               | 压缩列表                          |
| "intset"                | 整数集合                          |
| "skiplist"              | 跳跃表和字典                      |



## 字符串对象

