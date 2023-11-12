使用的源码是github上一个人注释的。
c++编写的单机数据库
使用LSM

# LSM

![](Pasted%20image%2020230625153453.png)
当一个数据写入时，首先记录预写日志，然后将数据插入到内存中一个名为 **MemTable** 的数据结构中。当 MemTable 的大小到达阈值后，就会转换为 Immutable MemTable 。**MemTable 和 Immutable MemTable 的内容大致相同**，但是 Immutable MemTable 无法再发生写入。因此，**在 MemTable 转换为 Immutable MemTable 的同时，也会开启一个新的 MemTable 供新数据写入。**
新老 SSTable 之间可能存在数据重叠。
合并策略为level compaction，合并过程中会进行数据去重、布局优化等操作。
非C/S模式，而是类似于库。

## 特点

将随机写转化成了顺序写，从而大大提高了写速度。
缺点：牺牲部分读性能，可能存在写放大问题
- 读放大（Read Amplification）。LSM-Tree 的读操作需要从新到旧（从上到下）一层一层查找，直到找到想要的数据。这个过程可能需要不止一次 I/O。特别是 range query 的情况，影响很明显。
- 空间放大（Space Amplification）。因为所有的写入都是顺序写（append-only）的，不是 in-place update ，所以过期数据不会马上被清理掉。

RocksDB 和 LevelDB 通过后台的 compaction 来减少读放大（减少 SST 文件数量）和空间放大（清理过期数据），但也因此带来了写放大（Write Amplification）的问题。

- 写放大。实际写入 HDD/SSD 的数据大小和程序要求写入数据大小之比。正常情况下，HDD/SSD 观察到的写入数据多于上层程序写入的数据。

# 对外接口

**对外写接口有两种：put、delete。**
在这两种操作中，数据库都会创建一个`WriteBatch`对象来进行对应操作。一个Batch中的操作是原子的，是leveldb执行一次操作的最小单位

# WriteBatch

关键成员：`std::string rep_;`

在 WriteBatch 中，所有的数据在被编码后存放在了一个名为 _rep__ 的 string 中。rep_ 在初始化时会被 resize 成 12 字节大小。这 12 个字节的前 8 个字节（即 64 个 bit）存放了操作的 sequence number，后 4 个字节（32 个 bit)存放了该 Batch 中操作的数量。sequence number 本质上是一个计数器，每个操作都对应了一个 sequence number 。该计数器在 leveldb 内部维护，每进行一次操作就累加一次。由于在 leveldb 中，一次更新或者一次删除，采用的是 append 的方式，而非直接更新原数据。因此对同样一个 key，会有多个版本的数据记录，而最大的 sequence number 对应的数据记录就是最新的。
rep_12个字节之后的为编码后的操作

![](Pasted%20image%2020230627155611.png)

合并写步骤：

在这个函数中`Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates)`，LevelDB 对多个并发写进行了优化，它会将同一时间内的“小写入”合并为一个“大写入”来执行，减少日志文件的小写入次数，增加整体的写入性能。
![](Pasted%20image%2020230627173748.png)
# 字节序

使用的是小端字节序。
大端序在解码和编码时，要比小端序麻烦一些，性能也要差一些，所以实现的时候推荐使用小端序，LevelDB使用的就是小端序实现.
# status

错误码和错误信息描述
![](Pasted%20image%2020230623171641.png)
# 编码

**levelDB使用小端字节序存储。**
level DB把key、value编码到一个单值里面。
如何把一对key、value编码到一个单值中呢？很自然的一个想法是用冒号做分割，存储成key:value这种格式。但是如果key或者value里面有冒号怎么办？总不能规定不允许使用冒号吧？所以使用特殊符号做区分是不够通用的。
一种常见的方法是用size-value的方式。

分为变长和定长编码。
又分为32位和64位。
## 变长编码

两个API:`EncodeVarint32`和`EncodeVarint64`。
编码目的：节约存储空间。
结果：int -> varint
LevelDB的解决方案是：把size域每个字节的bit0（最高位）设置为标记位，bit1~bit7存储size的值。如果bit0为0，表示size域到当前字节结束。下一个字节就是数据域了，如果bite0为1，表示下一个字节还是size域。
## 定长编码

定长编码的作用是转换成小端字节序。
两个API:`EncodeFixed32`和`EncodeFixed64`。
### 定长解码

解码函数`DecodeFixed32`和`DecodeFixed64`是编码的逆过程。
# MemTable

有两种：memtable和immutable memtable。
存在于内存中使用的数据结构是skiplist

只有一个memtable，满了后若没有来得及持久化，则会引起系统停顿。
常驻内存。使用跳表实现。
所有level DB写操作只能操作memtable，且无删除操作。
在memtable中是根据InternalKey进行排序的。
_MemTable_ 是一个有**引用计数**的类型，当其引用计数为 0 时（除了刚创建时）就会从内存中删除。

![](Pasted%20image%2020230803170638.png)
MemTable 的 KeyComparator[10] 负责从 memkey 中提取出 internalkey，最终排序逻辑是使用 InternalKeyComparator [11]进行比较，排序规则如下：
## 写操作

即add函数。
一个entry如下所示：
![](Pasted%20image%2020230629232911.png)
将kv，及本次操作的sn编码成上图entry，然后插入`table_`。
## 读操作

`MemTable::Get`函数
[LevelDB 完全解析（8）：读操作之 Get (qq.com)](https://mp.weixin.qq.com/s?__biz=MzI0NjA1MTU5Ng==&mid=2247483846&idx=1&sn=df505dab6729473c8276753c6efba050&chksm=e94467a5de33eeb3e429ee62bee9a79fa952130e96ed4e6cdc35a7a8d2add48103084698b710&scene=178&cur_album_id=1342947967103352833#rd)
# SSTable
## 层次

第一层是L0层，由immutable memtable 经过minor compaction生成。
**不同文件的key有重叠**，L0层根据**文件个数**触发major compaction

由compaction生成：
- Minor Compaction：一个 MemTable 直接 dump 成 level-0 的一个 SSTable。
    
- Major Compaction：多个 SSTable 进行合并、重整，生成 1～多个 SSTable。

1. level-0的每个 SSTable 的 key 范围可能相交，每一个 SSTable 都需要判断
2. 非 level-0 的每一层内，SSTable 的 key 范围是不相交的——SSTable 是根据 key 范围有序排序的，可以通过二分查找优化查找效率。

是数据落盘之后的文件。在 LevelDB 中，SSTable 是以 _.sst_ 文件的形式存在的。
sorted string table。有序
由compaction生成（后台线程进行压缩）：
- minor compaction：一个 Immutable MemTable 直接 dump 成 level-0 的一个 SSTable。
- major compaction：多个 SSTable 进行合并、重整，生成 1～多个 SSTable。

Manifest文件：记录每个SST文件的key的区间。
## 总体结构

![](Pasted%20image%2020230630192003.png)
![](Pasted%20image%2020230630005148.png)

图中所示为 SSTable 文件的整体结构。
文件的前部分为数据部分（DataBlock），文件的结尾部分为数据的索引。
索引部分，又分为了 Filter Block 、 Meta Index Block 、 Index Block 以及 Footer 。

### 数据部分

数据部分由许多个 DataBlock 组成，
**在 LevelDB 中，一个 DataBlock 的大小默认设置为 4 KB 。**
每个 DataBlock 由三个部分组成：数据部分（Data）、压缩类型（CompressionType）以及校验和（CRC）。
![](Pasted%20image%2020230630010805.png)
- Data 中存储的即是数据库的数据内容； 
- CompressionType 中记录了对原始数据使用的压缩方式，在 LevelDB 中默认的压缩方式是 Snappy ； 
- CRC 部分则记录了对 Data 以及 CompressionType 进行 CRC32 计算得到的结果，用来保证数据的正确性。
![](Pasted%20image%2020230630010830.png)
**Leveldb很巧妙了利用了有序数组相邻Key可能有相同的Prefix的特点来减少存储数据量。---共享key**。每个Entry只记录自己的Key与前一个Entry Key的不同部分。
![](Pasted%20image%2020230630192413.png)
==每一个entry是一组kv对==。
_DataBlock_ 中键值的共享在每一个 Entry 中进行，每 16 个键（默认值）保存在一个 Entry 中，并且为每个 Entry 在文件中维护了一个 Restart Point ，用于指向这个 Entry 。
LevelDB 将 block 的一个 key-value 称为一条 entry。每条 entry 的格式如下：![图片](https://mmbiz.qpic.cn/mmbiz_png/9637P74IBOGE3fDb9Yib2lx8qlDQiaCR05D6uQpUc3abicbhuKQTu77qKQl8k1pnLKnkMG6sKo2jpL0kptPY4qUdQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
- shared_bytes：和前一个 key 相同的前缀长度。
- unshared_bytes：和前一个 key不同的后缀部分的长度。
- value_length：value 数据的长度。
- key_delta：和前一个 key不同的后缀部分。
- value：value 数据。

tailer部分包含所有的重启点（restart point）

#### DataBlock

block中的每条entry格式为：
```c++
<SharedKeyLength> <NonSharedKeyLength> <ValueLength> <NonSharedKeyContent> <ValueContent>
```

### 索引部分

## 写流程

## 读流程

_DataBlock_ 中键值的共享在每一个 Entry 中进行，每 16 个键（默认值）保存在一个 Entry 中，并且为每个 Entry 在文件中维护了一个 Restart Point ，用于指向这个 Entry 。这样，在读取一个 Key 时，只需要找到这个 Key 所在的 Entry ，然后从这个 Entry 的第一个 Key 开始读即可。
共享值的压缩方法只会对 Key 进行，对于 Value 则是不做任何处理直接编码进 buffer 中。

入口： _table_cache.cc_ 中的 `FindTable` 函数中。

# 数据结构

## 跳表

### 目的

内存中的**memtable的底层数据结构(索引)。**
既是索引也是存储数据的结构

使用数组查询性能高，但是读写性能差O(N)，而普通链表相反，查询为O(N).
AVL/红黑树过于复杂，重新平衡操作难；且不适合范围查询。

跳表类似于一种二分查找的有序链表。

### 名词

一个拥有 n 个指针的链表节点称为 n 级节点（高度为n）。
在插入时，一个新的节点的级别通过随机数进行生成。

### levelDB实现

#### 数据格式

==这个格式很重要==
```cpp
SkipList Node := InternalKey + ValueString   memrecord

InternalKey := KeyString + SequenceNum + Type

Type := kDelete or kValue

ValueString := ValueLength + Value
KeyString := UserKeyLength + UserKey
```

#### 外部接口

总共只有两个：`Insert` 和 `Contains` 。因此，SkipList 实际上并不支持删除操作。
一个 Node 一旦被插入了 SkipList 中，那么其就无法再被移除了。并且，在插入时要求跳表中不存在与待插入键相同键的节点。注意，这里的键并不是用户在插入时给出的原始键，**而是包含了 sequence number 的 internalKey** ，理论上每个 internalKey 都是独一无二的。

#### 成员变量

SkipList 内部的数据部分包含了六个对象：随机生成节点层级时的最大值 _kMaxHeight_ 、用于比较不同键大小的比较器 _compare__ 、用于管理内存的 _arena__ 、跳表的头节点 _head__ 、当前跳表中所有节点的最高层级 _max_height__ 以及随机数生成器 _rnd__ 。

## 布隆过滤器

### 论文

1970年
[Space/time trade-offs in hash coding with allowable errors](Space/time trade-offs in hash coding with allowable errors)
### 目的

减小读放大现象。可以快速判断某个key一定不在某个sstable中

### 数据结构

底层使用一个bit array，初始所有位都为0.

插入时对于一个元素，使用k个hash函数进行散列，然后将k个hash在bit array中的对应位置置1。
查找时同样操作。如果k个位置均为1，则该元素有可能存在；有0肯定不存在。

### 时空优势

**不存储数据项本身，而是存储数据项的几个哈希值**，并且用高效紧凑的位数组来存，避免了指针的额外开销。使得 Bloom Filter 的大小与数据项本身大小（如字符串的长短）无关。
因此也无需对数据项本身进行压缩。

牺牲一些准确性，换取时空高效性。

### 参数取舍

bloom filter有一定的误判率，跟以下参数有关：
1. 哈希函数的个数 k
2. 底层位数组的长度 m
3. 数据集大小 n

论文中调参结论：当 `k = ln2 * (m/n)` 时，Bloom Filter 获取最优的准确率。m/n 即 bits per key（集合中每个 key 平均分到的 bit 数）。

### level DB实现

并未真正使用k个hash函数，而是使用double-hashing进行了一个优化。
使用一个hash来达到k个hash的效果


## LRU cache

经典解法是使用一个哈希表（unordered_map）和一个双向链表，哈希表解决索引问题，双向链表维护访问顺序。

### 目的

用以部分缓存SSTable。两种cache

### level DB

依据上述接口，可捋出 LevelDB 缓存相关需求：
1. 多线程支持
2. 性能需求
3. 数据条目的生命周期管理

使用句柄指向缓存中的kv条目。
只有在所有持有该条目句柄都释放时，该条目所占空间才会真正被释放。
![](Pasted%20image%2020230624201426.png)


有两张表HashTable和LRUcache

使用两个双向链表保存数据：
- in-use：不可驱逐，无序。
- lru：空闲的，可以被驱逐的，有序按时间戳

### 数据结构

![](Pasted%20image%2020230624202652.png)
![](Pasted%20image%2020230624202703.png)

#### LRUHandle----基本数据单元

`LRUHandle` 是双向链表和哈希表的基本构成单位，同时也是数据条目缓存和操作的基本单元.
作为节点

#### HandleTable

LRUCache的索引
```cpp
LRUHandle** list_;  // 指向一个桶，桶中是一个链表
```

#### LRUCache

缓存数据结构，使用LRU驱逐策略

#### ShardedLRUCache

ShardedLRUCache由一组LRUCache组成，每个是一个分片，也是加锁的粒度，减少加锁开销，从而增加并发度。

取32位key hash做分片路由

# Arena内存池

简化内存池，只申请不释放。只有当整个 Arena 对象销毁的时候才会将之前申请的内存释放掉。

## 目的

LevelDB中需要频繁申请和释放内存，如果直接使用系统的new/delete或者malloc/free接口申请和释放内存，会产生大量内存碎片，进而拖累系统的性能表现。

## 架构

![](Pasted%20image%2020230804003151.png)
![](Pasted%20image%2020230625014850.png)
**alloc_ptr_** 指向当前可分配的内存块空闲空间的头部。 
**alloc_bytes_remaining_** 存储当前内存空空闲空间的大小。
**blocks_** 是一个数组，保存了所有已分配内存块的指针。
**memory_usage_** 保存Arena占用的总内存大小。

## 核心思想

**Arena从系统申请内存时是以固定大小（block_size) 整块申请的。**
LevelDB向Arena申请内存时，Arena按照LevelDB申请的内存大小(request_size)从已申请的内存块中分配，如果内存块中剩下的内存少于request_size，就不能直接分配了。此时分两种情况：如果requet_size 大于block_size的四分之一，Arena会申请（new）一个request_size的内存返回给LevelDB。反之就申请一个block_size的内存块，然后从新的内存块中分配内存给LevelDB。
这是四分之一策略。减小内存碎片。
## 释放

Arena申请的所有内存都是在Arena析构时一起释放的。**LevelDB中每个MemTable对象都有一个Arena成员变量**，MemTable是LevelDB在内存中的数据缓存结构，每个MemTable写满后就会落盘（写入磁盘），然后被销毁。Arena也会跟着一起析构。
# DB

是直接对外的API类。
class DBImpl : public DB继承自DB类，是主要的level DB实现类。
# 日志及WAL

重放日志过程

对应源文件：
![](Pasted%20image%2020230803163910.png)

![](Pasted%20image%2020230803164315.png)

存于log文件中，一个log文件包含多个block，一个block保护你多条record。log文件划分为固定大小（32KB）的block。如果block末尾不足7B，不足以放下record头，那么进行padding（填0x00）
![](Pasted%20image%2020230630191645.png)
![](Pasted%20image%2020230629205002.png)

日志被存储的最小单位是 **Chunk（record）**，一个 Chunk 包含了四个部分：校验和 Checksum（4 个字节）、数据长度 Length （2 个字节），Chunk 类型 Type（1 个字节）以及数据内容，前三个部分被称为 ChunkHeader 。
校验和是由数据类型以及数据内容计算出来的，使用的算法是 CRC 校验算法。
**Chunk /record类型一共有 4 种：Full、First、Midlle、Last** 。之所以需要有四种 Chunk 类型，是因为一条日志有可能因为大小太大而被存在多个 Chunk 当中。假如一条日志被存放在了一个 Chunk 当中，那么这个 Chunk 的类型就是 Full ；假如一条日志被存放在了多个 Chunk 当中，那么第一个 Chunk 的类型为 First ，最后一个 Chunk 的类型为 Last ，First 和 Last 之间包含了零个或多个 Middle 类型的 Chunk ，这些所有的 Chunk 中的数据组合起来就是日志的原始数据。

**kHeaderSize = 7 byte；即日志头（chunkheader）为7byte。Header is checksum (4 bytes), length (2 bytes), type (1 byte).**
日志头固定为7B，日志value（即用户data）为变长的。

## 日志格式

```cpp
Block := Record * N   blocksize = 32KB
Record := Header + Content
Header := Checksum + Length + Type  Header size = 7B
Type := Full or First or Midder or Last
```
leveldb没有进行日志的压缩

# env

考虑到移植以及灵活性，**LevelDB将系统相关的处理（文件/进程/时间）抽象成Evn**，用户可以自己实现相应的接口，作为option的一部分传入，默认使用自带的实现。

env.h中声明了:
虚基类env，在env_posix.cc中，派生类PosixEnv继承自env类，是LevelDB的默认实现。
虚基类WritableFile、SequentialFile、RandomAccessFile，分别是文件的写抽象类，顺序读抽象类和随机读抽象类
类Logger，log文件的写入接口，log文件是防止系统异常终止造成数据丢失，是memtable在磁盘的备份
类FileLock，为文件上锁
WriteStringToFile、ReadFileToString、Log三个全局函数，封装了上述接口

# 版本控制

level使用**增量式**存储。
Leveldb每次新生成sstable文件，或者删除sstable文件，都会从一个版本升级成另外一个版本。即sstable是leveldb的最小操作单元，有原子性。
**LeveDB用**Version**表示一个版本的元信息。**
每次文件信息修改都会生成新version。
**Version中主要包括一个FileMetaData指针的二维数组，分层记录了所有的SST文件信息。**

**FileMetaData**数据结构用来维护一个文件的元信息，包括文件大小，文件编号，最大最小值，引用计数等，其中引用计数记录了被不同的Version引用的个数，保证被引用中的文件不会被删除。除此之外，Version中还记录了触发Compaction相关的状态信息，这些信息会在读写请求或Compaction过程中被更新。
**VersionSet**是一个Version构成的双向链表，这些Version按时间顺序先后产生，记录了当时的元信息，链表头指向当前最新的Version，同时维护了每个Version的引用计数。使得LevelDB可以在一个稳定的快照视图上访问文件。VersionSet中除了Version的双向链表外还会记录一些如LogNumber，Sequence，下一个SST文件编号的状态信息。
![](Pasted%20image%2020230630224051.png)
versionset中的所有version都指向同一个FileMetaData二维数组。
相邻Version之间的不同仅仅是一些文件被删除另一些文件被删除。也就是说将文件变动应用在旧的Version上可以得到新的Version，这也就是Version产生的方式。
LevelDB用**VersionEdit**来表示这种相邻Version的差值。
![](Pasted%20image%2020230630224226.png)
为了避免进程崩溃或机器宕机导致的数据丢失，LevelDB需要将元信息数据持久化到磁盘，承担这个任务的就是**Manifest**文件。可以看出每当有新的Version产生都需要更新Manifest，很自然的发现这个新增数据正好对应于VersionEdit内容，**也就是说Manifest文件记录的是一组VersionEdit值**，在Manifest中的一次增量内容称作一个Block，其内容如下：
```cpp
Manifest Block := N * Item
Item := [kComparator] comparator
		or [kLogNumber] 64位log_number
		or [kPrevLogNumber] 64位pre_log_number
		or [kNextFileNumber] 64位next_file_number_
		or [kLastSequence] 64位last_sequence_
		or [kCompactPointer] 32位level + 变长的key
		or [kDeletedFile] 32位level + 64位文件号
		or [kNewFile] 32位level + 64位 文件号 + 64位文件长度 + smallest key + largest key
```
恢复元信息的过程也变成了依次应用VersionEdit的过程，这个过程中有大量的中间Version产生.LevelDB引入VersionSet::Builder来避免这种中间变量，方法是先将所有的VersoinEdit内容整理到VersionBuilder中，然后一次应用产生最终的Version，这种实现上的优化如下图所示：
![](Pasted%20image%2020230630230307.png)

**重要类**：Version、FileMetaData、VersionSet、VersionEdit、Manifest和Version::Builder

## manifest文件

相当于log
Manifest 文件保存了整个 LevelDB 实例**元数据**，比如：每一层有哪些 SSTable、属于哪个层级、文件名称、最小主键和最大主键。
格式上，Manifest 文件其实就是一个 **[log 文件](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/Ry7FRClFz8XRut_fSYlX2g)**，一个 log record 就是一个 **[VersionEdit](https://link.zhihu.com/?target=https%3A//github.com/google/leveldb/blob/1.22/db/version_edit.h%23L29)**。
leveldb元数据包括：
```cpp
std::string comparator_;比较器的名称，这个在创建 LevelDB 的时候就确定了，以后都不能修改。
uint64_t log_number_;最小的有效 log number。小于 log_numbers_ 的 log 文件都可以删除。
uint64_t prev_log_number_;已经废弃，代码保留是为了兼容旧版本的 LevelDB。
uint64_t next_file_number_;下一个文件的编号 。
SequenceNumber last_sequence_;SSTable 中的最大的 sequence number。

std::vector<std::pair<int, InternalKey> > compact_pointers_;记录每一层要进行下一次 compaction 的起始 key。
DeletedFileSet deleted_files_;可以删除的 SSTable（level-no -> file-no）。
std::vector<std::pair<int, FileMetaData> > new_files_;新增的 SSTable。
```

## current文件

由于每次启动leveldb都会新建一个manifest文件，当前文件记录了**当前的清单文件名**，即当前的Manifest，而Manifest可能有多个。
该文件内容只有一个：当前使用的manifest文件的文件名
## recover恢复

1. 利用Current文件读取最近使用的manifest文件；
2. 创建一个空的version，并利用manifest文件中的session record依次作apply操作，还原出一个最新的version，注意**manifest的第一条session record是一个version的快照，后续的session record记录的都是增量的变化**；
3. 将非current文件指向的其他过期的manifest文件删除；
4. 将新建的version作为当前数据库的version；
## MVCC

leveldb使用MVCC避免写冲突。
快照，使用sequence number来表示一个版本。
# compaction

分为minor compaction和major compaction：
- Minor Compaction基本代码路径是：DBImpl::CompactMemTable=> DBImpl::WriteLevel0Table => BuildTable。
###  Major Compaction

1. 每次 compaction 结束，更新 manifest 之后，都会调用 VersionSet::Finalize[4] 计算下一次要进行 major compaction 的 level。
    
2. 每次 major compaction 开始时，调用 VersionSet::PickCompaction[5] 计算需要进行 compaction 的SSTable。
    
3. 如果选中的 level-n 的 SSTable 和 level-n+1 的SSTable 的 key 范围没有重叠，可以直接将 level-n 的 SSTable “移动”到 level-n+1，只需要修改 Manifest。
    
4. 否则，调用 DBImpl::DoCompactionWork[6] 对 level-n 和 level-n+1 的 SSTable 进行多路归并。
### compaction的作用

- **数据持久化**（minor compaction）
- **提高读写的效率**：leveldb适合读少写多的场景；读操作比较复杂，先在内存中进行logn的查询，然后查询L0，因为L0的key有overlap，因此最差情况下可能要遍历L0层所有文件。
- **平衡读写差异**：解决用户写入速度始终大于major compaction速度的问题。此时会导致0层文件数量持续上升。使用以下两个规定来解决：
	- 当0层文件数量超过`SlowdownTrigger`时，写入的速度主要减慢；
	- 当0层文件数量超过`PauseTrigger`时，写入暂停，直至Major Compaction完成；
- **清理过期数据**
### compaction触发时机

LevelDB中会有后台线程来执行Compaction的操作，将上层文件与下层文件归并生成新的下层文件。
Version中记录的各层的文件信息来帮助决定进行Compaction的时机：
- **容量触发Compaction**：相关变量compaction_level_、compaction_score_。记录了当前Version最需要进行Compaction的Level，以及其需要进行Compaction的紧迫程度。score大于1被认为是需要马上执行的。我们知道每次文件信息的改变都会生成新的Version，所以每个Version对应的这两个值初始化后不会再改变。**level0层compaction_score_与文件数相关，其他level的则与当前层的文件总大小相关**。同时Version中会记录每层上次Compaction结束后的最大Key值compact_pointer_，下一次触发自动Compaction会从这个Key开始。容量触发的优先级高于下面将要提到的Seek触发。
- **Seek触发Compaction**：Version中会记录file_to_compact_和file_to_compact_level_，这两个值会在Get操作每次尝试从文件中查找时更新。
- **文件个数触发Compaction** ： L0层文件个数过多会触发compaction。
- **文件总大小触发**：L1-LN该层文件总大小超过阈值就会触发。
- **某层无效文件过多**：seek compaction文件无效查找次数过多，将这些无效查询的文件进行错峰合并（时机和文件）

>防止迭代compaction

- source层的文件个数只有一个；
- source层文件与source+1层文件没有重叠；
- source层文件与source+2层的文件重叠部分不超过10个文件；
当满足这几个条件时，可以将souce层的该文件直接移至source+1层。
### compaction过程

1. 寻找合适的输入文件：其实输入文件为上次合并文件的最大key的首个文件
2. 根据key重叠情况扩大输入文件集合：level n层中找和起始输入文件key有重叠的文件。再在level n+1中找key有重叠的文件构成两层的输入文件集合。
3. 多路合并：输出到level n+1
4. 积分计算：进行计分挑选出下个需要合并的层数
### compaction文件选取

选取level n+1中于level n中key有重叠的文件
level n文件选取：
### 构造compaction

首先通过Version来构造所有本次Compaction所需要的信息，记录在**Compaction对象**中，包括发生Compaction的level，所有参与的level和level+1层的文件信息，level+2层的文件信息等。

### version持久化

Compaction过程会造成文件的增加和删除，这就需要生成新的Version，上面提到的Compaction对象包含本次Compaction所对应的VersionEdit，Compaction结束后这个VersionEdit会被用来构造新的VersionSet中的Version。同时为了数据安全，这个VersionEdit会被Append写入到Manifest中。

### Compaction 的问题

Compaction 会对 LevelDB 的性能和稳定性带来一定影响：

1. 消耗 CPU：对 SSTable 进行解析、解压、压缩。
    
2. 消耗 I/O：大量的 SSTable 批量读写，十几倍甚至几十倍的写放大会消耗不少 I/O，同时缩短 SSD 的寿命（SSD 的写入次数是有限的）。
    
3. 缓存失效：删除旧 SSTable，生成新 SSTable。新 SSTable 的首次请求无法命中缓存，可能引发系统性能抖动。


常见的做法是，控制 compaction 的速度（比如 RocksDB 的 Rate Limiter[7]），让 compaction 的过程尽可能平缓，不要引起 CPU、I/O、缓存失效的毛刺。这种做法带来一个问题：compaction 的速度应该控制在多少？Compaction 的速度如果太快，会影响系统性能；Compaction 的速度如果太慢，会阻塞写请求。这个速度和具体的硬件能力、工作负载高度相关，往往只能设置一个“经验值”，比较难通用。同时这种做法只能在一定程度上减少系统毛刺、抖动，Compaction 带来的写放大依然是那么大。

优化：
1. Dostoevsky: Better Space-Time Trade-Offs for LSM-Tree Based Key-Value Stores via Adaptive Removal of Superfluous Merging - 这篇论文将各种 compaction 方式和影响介绍得非常清楚。
    
2. WiscKey: Separating Keys from Values in SSD-conscious Storage - 通过将键值分离，大大减少了写放大。


# 写流程

来自博客[LevelDB 源代码阅读（一）：写流程 - 马克刘的博客 (thumarklau.github.io)](https://thumarklau.github.io/2021/07/08/leveldb-source-code-reading-1/)
使用LSM树存储数据
![](Pasted%20image%2020230629185136.png)
写入时先WAL，然后写memtable。

用户调用`leveldb::Status s = db->Put(leveldb::WriteOptions(), key, value);`
进入`DB::Put`函数，首先创建合并写对象`writebatch`，并将kv编码
一个操作的kv对，被编码为下图所示：
![](Pasted%20image%2020230629185519.png)
之后加入此writebatch对象的rep_（？）：
![](Pasted%20image%2020230629185556.png)
之后进入`DBImpl::Write`函数。这是一个多线程函数。`DBImpl`维持着一个写操作的队列`writers_`。首先进行写之前的空间检查`MakeRoomForWrite`. 位于队首的线程将这个队列中的多个写操作合成一个大的writebatch。先WAL： `log_->AddRecord`，再写入memtable：`WriteBatchInternal::InsertInto`。最后改变写队列中，这堆写的状态。

`WriteBatchInternal::InsertInto`：
里面调用`MemTable::Add`函数。
![](Pasted%20image%2020230629203727.png)
最终调用`table_.Insert(buf);`。将此编码后的kv（buf）插入到条表中。
## 写优化

batch写和合并写。
![](Pasted%20image%2020231107161656.png)
# Iterator

![](Pasted%20image%2020231107163038.png)
levelDB使用了大量的iterator。
>使用原因

leveldb不同组件具体存储格式是不同的，如果每一个层次的数据遍历都需要详细的关心全部数据存储格式，无疑将使得整个过程变得无比的冗余复杂。
Iterator在各个层次上，向上层实现提供了：
	**无须了解下层存储细节的情况下，通过统一接口对下层数据进行遍历的能力。**
## 分类

- 基本Iterator：最原子的Iterator，针对相应的数据结构实现Iterator接口；
- 组合Iterator：通过各种方式将多个基本Iterator组合起来，向上层提供一致的Iterator接口。
- 功能Iterator：某种或多种组合Iterator的联合使用，附加一些必要的信息，实现某个过程中的遍历操作。
### 基本Iterator

三种；别针对Memtable，Block以及Version中的文件索引格式，实现了最原子的Iterator。
1. memtableIterator：遍历跳表
2. block::iter：针对SST block存储格式的iterator
3. version::levelFileNumIterator：Level1层之上的文件由于相互之间没有交集且有序，可以利用文件信息中的最大最小Key来进行二分查找。LevelFileNumIterator就是利用这个特点实现的对文件元信息进行遍历的Iterator。其中每个项记录了当前文件最大key到文件元信息的映射关系。这里的文件元信息包含文件号及文件长度。
### 组合Iterator

由多个基本iterator或者组合iterator组成。
- **TwoLevelIterator**
- **MergingIterator**
### 功能Iterator
# cache

有两种cache：block cache和 table cache.
在 LevelDB 中，block cache 和 table cache 都是基于 ShardedLRUCache[1] 实现的。
# 读流程

![](Pasted%20image%2020230803184110.png)

LevelDB 通过 leveldb::DB::Get接口对外提供点查询的能力，具体的实现是 leveldb::DBImpl::Get。

首先会生成internal key（用于memtable） = user key + sequence number。只能查询这个sequence之前的写入。
level0文件由于是immutable dump转储产生，会相互重叠，因此要每个文件查找。
其他层次归并（压缩）保证了有序且不重叠，使用二分查找。

 leveldb::DBImpl::Get 的实现：

1. 获取互斥锁[9]。
    
2. 获取本次读操作的 Sequence Number[10]：如果 ReadOptions 参数的 snapshot 不为空，则使用这个 snapshot 的 sequence number；否则，默认使用 LastSequence（LastSequence 会在每次写操作后更新）。
    
3. MemTable， Immutable Memtable 和 Current Version（SSTable） 增加引用计数[11]，避免在读取过程中被后台线程进行 compaction 时“垃圾回收”了。
    
4. 释放互斥锁[12] 。所以，下面 5 和 6 两步是没有持有锁的，不同线程的请求可以并发执行。
    
5. 构造 LookupKey[13] 。
    
6. 执行查找：
    
7. 从 MemTable 查找[14] 。
    
8. 从 Immutable Memtable 查找[15]。
    
9. 从 SSTable 查找[16]。
    
10. 获取互斥锁[17]
    
11. 更新 SSTable 的读统计信息[18]，根据统计结果决定是否调度后台 Compaction[19]。=> 极少遇到有读触发 compaction 的场景，这一步的似乎意义不大。
    
12. MemTable, Immutable Memtable 和 Current Version 减少引用计数，返回结果[20]。
    
主要逻辑在于读 MemTable（包括 Immutable MemTable） 和读 SSTable。
## 快照

所谓快照，其实就是代表了数据库在某一个时刻的状态，**LevelDB 中数据读取就是通过 Snapshot 机制来实现的。**
level DB的快照通过一个名为`sequence number`的整型数来实现。当数据被写入后，此次操作对应的数据就不可改变。每次有写操作时sn就+1.
==LevelDB 维护了一个全局的 sequence number ，每次有写操作发生时这个 sequence number 就会加一。==
在写操作发生时，LevelDB 会为这个写操作分配一个 sequence number，其值就是当前系统中 sequence number 的值。

# compaction

单线程合并文件

# 几种key

![](Pasted%20image%2020230803175620.png)
user key
internal key = user key + sequence number + value type用于在跳表中查找
lookup key = internal key + internal key length

## lookup key

在`dbformat.h`中定义
![](Pasted%20image%2020230803180633.png)
1. klength 的类型是 varint32，存储 userkey + tag 的长度[22]。
    
2. userkey 就是 Get 接口传入的 key 参数。
    
3. **tag 是 7 字节 sequence number 和 1 字节 value type**。
    
4. 一个 varint32 最多需要 5 个字节，tag 需要 8 字节。所以一个 LookupKey 最多需要分配 usize + 13 字节内存（usize 是 userkey 的 size）。
# 参考

[庖丁解LevelDB之Iterator | CatKang的博客](http://catkang.github.io/2017/02/12/leveldb-iterator.html)
# 问题

>levelDB为什么设计成多层结构？

为了分摊写放大。
[(23 封私信 / 73 条消息) leveldb为什么要设计为多层结构呢？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/396452321)