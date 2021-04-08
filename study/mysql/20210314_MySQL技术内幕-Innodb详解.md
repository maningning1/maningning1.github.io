# MySQL技术内幕-Innodb详解

## 概述

## Innodb架构

### 内存

Innodb存储引擎中内存结构如下所示，其中主要包括缓冲池、重做日志缓冲及额外内存池等。

![1.04_Innodb内存结构](D:\study_note\maningning1.github.io\images\mysql\1.04_Innodb内存结构.png)

#### 缓冲池

Innodb存储引擎是基于磁盘存储的，并将其中记录按照页（MySQL中默认大小为16Kb）的方式进行管理，由于CPU速度与磁盘速度间的鸿沟，于是基于缓冲池技术来提高数据库的整体性能。

缓冲池就是缓存表数据与索引数据，把磁盘上的数据加载到缓冲池中，避免每次访问都进行磁盘IO，起到加速访问的作用。

缓冲池中保存的数据页类型有索引页、数据页、undo页、插入缓冲、自适应哈希索引、锁信息、数据字典信息等，如下所示：

![1.01_缓冲池结构](D:\study_note\maningning1.github.io\images\mysql\1.01_缓冲池结构.png)

而缓存容量是有限的，所以必须采用淘汰算法来管理内存，在Innodb中也是采用LRU算法对缓冲池进行管理，不过也根据数据库系统的一些特点进行了更新。

##### 传统LRU算法

LRU算法，即最近最常使用（Least Recently Used），Redis就使用该算法对内存进行管理，其算法会“淘汰”使用频率较低的数据，其原理如下所示，采用一个链表记录LRU中的数据，当链表中包含插入的数据时，会将该数据所对应的链表节点插入到LRU链表的头部；当链表中不包含插入的数据时，会将该数据所对应的链表节点插入到LRU链表的头部，如果链表中节点个数达到LRU所设置的最大值，则会淘汰最末端的节点。

![1.02_传统的LRU算法](D:\study_note\maningning1.github.io\images\mysql\1.02_传统的LRU算法.png)

##### 预读失效

磁盘读写并不是按区读取，而是按页读取（Linux系统中默认内存的页大小为4Kb），这样就可以预加载一部分数据，这就叫预读。而MySQL并没有从预读的页中读取数据，就称为预读失效。

为了防止预读失效，主要可以从以下两点优化：

+ 让预读失败的页停留在缓冲池LRU里的时间尽可能短；
+ 让真正被读取的页放在缓冲池头部；

##### 缓冲池污染

缓冲池污染是指当某个SQL语句需要扫描大量数据时，可能会导致将整个缓冲池的页都替换出去，导致大量热数据被换出，造成MySQL性能急剧下降。

##### 改进的LRU算法

为了防止预读失效及缓冲池污染问题，Innodb采用的LRU算法针对传统的LRU算法进行了一些改进：

+ 在LRU链表中加入了midpoint位置，该位置在LRU长度的5/8处，将LRU划分为了new和old两个区域，新插入的页只加入到old的头部；

  ![1.03_改进的LRU算法](D:\study_note\maningning1.github.io\images\mysql\1.03_LRU算法的分代.png)

+ 在缓冲池中加入一个“停留时间窗口”的机制，插入old的页只有满足**被访问**且**停留时间**大于设置时间，才会被放入new区的头部；

##### 相关参数

```shell
mysql> show variables like '%innodb_buffer_pool_size%';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
1 row in set (0.00 sec)

mysql> show variables like '%innodb_old_blocks_pct%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
1 row in set (0.00 sec)

mysql> show variables like '%innodb_old_blocks_time%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
1 row in set (0.00 sec)
```

+ innodb_buffer_pool_size：配置缓冲池的大小；
+ innodb_old_blocks_pct：old占整个LRU链表的长度，默认为37；
+ innodb_old_blocks_time：old代的停留时间窗口，默认为1000，单位为毫秒；

#### 重做日志缓冲

#### 额外内存池



## Innodb关键特性

### 写缓冲（change buffer）

## Innodb存储结构

在Innodb中，所有数据都被逻辑的存放在一个空间中，称为表空间，表空间又由段、区、页组成，如下所示。

![1.05_Innodb逻辑存储结构](D:\study_note\maningning1.github.io\images\mysql\1.05_Innodb逻辑存储结构.png)

### 表空间

表空间是Innodb存储引擎逻辑结构的最高层，所有数据都存放在表空间ibdata1中。

### 段

### 区

### 页

### 行

## Innodb数据页结构

Innodb数据页存放的是表中行的实际数据，其主要包括以下7个部分组成：

+ File Header（文件头）
+ Page Header（页头）
+ Infimun和Supremum Records
+ User Records（用户记录，即行记录）
+ Free Space（空闲空间）
+ Page Directory（页目录）
+ File Trailer（文件结尾信息）

![1.06_Innodb数据页结构](D:\study_note\maningning1.github.io\images\mysql\1.06_Innodb数据页结构.png)

### File Header

File Header用来记录页的一些头信息，由以下8个部分组成，共占用38字节。

| 名称                             | 大小（字节） | 说明                                                         |
| -------------------------------- | ------------ | ------------------------------------------------------------ |
| FIL_PAGE_SPACE_OR_CHKSUM         | 4            | MySQL4.0.14之前版本，该值为0；之后版本该值代表页的checksum值。 |
| FIL_PAGE_OFFSET                  | 4            | 表空间的页的偏移值，用于定位某页的位置。                     |
| FIL_PAGE_PREV                    | 4            | 当前页的上一页，B+ Tree是双向链表。                          |
| FIL_PAGE_NEXT                    | 4            | 当前页的下一页，B+ Tree是双向链表。                          |
| FIL_PAGE_LSN                     | 8            | Log Sequence Number，代表该页最后被修改的日志序列位置。      |
| FIL_PAGE_TYPE                    | 2            | Innodb存储引擎页的类型，详情见下表所示。                     |
| FIL_PAGE_FILE_FLUSH_LSN          | 8            | 该值仅在系统表空间的一个页中定义，代表文件至少被更新到了该LSN值。对于独立表空间，该值为0。 |
| FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID | 4            | 表示页属于哪个表空间                                         |

FIL_PAGE_TYPE常见类型如下表所示，记住0x45BF，该值代表了存放的是数据页，即实际行记录的存储空间。

![1.07_Innodb存储引擎中页的类型](D:\study_note\maningning1.github.io\images\mysql\1.07_Innodb存储引擎中页的类型.png)

### Page Header

Page Header用来记录数据页的状态信息，由14个部分组成，共占用56个字节，其具体结构如下所示：

| 名称              | 大小（字节） | 说明                                                   |
| ----------------- | ------------ | ------------------------------------------------------ |
| PAGE_N_DIR_SLOTS  | 2            | 在页目录中槽（Slot）的个数。                           |
| PAGE_HEAP_TOP     | 2            | 堆中第一个记录的指针，记录在页中是根据堆的形式存放的。 |
| PAGE_N_HEAP       | 2            | 堆中的记录数。                                         |
| PAGE_FREE         | 2            | 指向可重用空间的首指针。                               |
| PAGE_GARBAGE      | 2            | 已删除记录的字节数。                                   |
| PAGE_LAST_INSERT  | 2            | 最后插入记录的位置。                                   |
| PAGE_DIRECTION    | 2            | 最后插入的方向。                                       |
| PAGE_N_DIRECTION  | 2            | 一个方向连续插入记录的数量。                           |
| PAGE_N_RECS       | 2            | 该页中记录的数量。                                     |
| PAGE_MAX_TRX_ID   | 8            | 修改当前页的最大事务ID。                               |
| PAGE_LEVEL        | 2            | 当前页在索引树中的位置。                               |
| PAGE_INDEX_ID     | 8            | 索引ID，表示当前页属于哪个索引。                       |
| PAGE_BTR_SEG_LEAF | 10           | B+树数据页非叶节点所在段的segment header。             |
| PAGE_BTR_SEG_TOP  | 10           | B+树数据页所在段的segment header。                     |

### Infimum和Supremum Record

在Innodb的数据页中有两个虚拟的行记录，用来限定记录的边界，这就是Infimum和Supremum Record。Infimum记录是比该页中任何主键值都要小的值，Supremum是比任何可能大的值还要大的值，其结构如下所示：

![1.08_Infimum和Supremum](D:\study_note\maningning1.github.io\images\mysql\1.08_Infimum和Supremum.png)

### User Record和Free Space

User Record是指实际存储行记录的内容；Free Space是指空闲空间，它们都是使用链表来保存的。

### Page Directory

Page Directory中存放了记录的相对位置。

### File Trailer

用于检测页是否已经完整地写入磁盘。