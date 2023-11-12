Hadoop框架下的数据库
参考文章：
- [我终于看懂了HBase，太不容易了... - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/145551967)


# 概述

**Bigtable开源实现**，基于HDFS存储引擎，提供了高并发的随机写和实时查询功能。
HBase 是一个开源的、分布式的、版本化的 **NoSQL** 数据库（也即非关系型数据库）。
利用HDFS提供分布式数据存储。
也以表的形式组织数据，表也由行和列组成。HBase有列族概念，将一列或者多列组织在一起，HBase每一个列必须属于某个列族。**存储结构化和半结构化的数据**。同一列可以不同类型。
缺少很多RDBMS（关系型数据库管理系统）特性，比如列类型，辅助索引，触发器，和高级查询语言等。

# 特点

- 强读写一致，不是“最终一致性”的数据存储，这使得它非常适合高速的计算聚合。CP
- 自动分片，通过Region分散在集群中，当行数增长的时候，Region也会自动的切分和再分配
- 自动的故障转移
- Hadoop/HDFS集成
- 丰富的“简洁，高效”API，Thrift/REST API，Java API
- 块缓存，布隆过滤器，可以高效的列查询优化
- MVCC。当修改值时实际为添加一条记录。读时读时间戳最新的即可。
- 本质上为key-value数据库。Key由RowKey(行键)+ColumnFamily（列族）+Column Qualifier（列修饰符）+TimeStamp（时间戳--版本）+KeyType（类型：DELETE）组成，而Value就是实际上的值。

# 数据模型

与数据库的Database和Table的概念类似，HBase将这两项称之为namespace和table，一个namespace能够包含多个table，HBase内置了两个默认的namespace：
1. hbase：系统内建表，包括了namespace和meta表；
2. default：用户建表时未指定namespace的表都创建在这里。

HBase表有一系列行构成，每行数据都有一个rowkey，以及若干个column family构成，每个column family可以包含无限多的列。因此，可以将HBase看成一个有序多维映射表，**也可以看作是一个key/value存储系统，rowkey作为key，其他部分是value。**

- rowkey：HBase表中的数据都是以rowkey作为标识的，rowkey类似于关系型数据库中的“主键”，每行数据都有一个rowkey，唯一标识该行，是定位数据的索引。同一张表内，rowkey是全局有序的。rowkey没有数据类型，以字节数组（byte[]）的形式保存。
- **column family**：HBase表中的数据是按照column family组织的，每行数据拥有相同的column family，column family属于schema的一部分，**定义表结构时就需要指定好**。column family是访问控制的基本单元，**同一个column family的数据再物理上会存储在同一个文件内。**
- column qualifier：**column family内部列标识，HBase每列数据可以通过family：qualifier来做唯一标识**。Qualifier不属于schema的一部分，可以动态制定，且每行数据可以有不同的schema。column qualifier也是没有数据类型的，以字节数组（byte[]）的形式保存。
- cell：通过rowkey，column family + column qualifier可以唯一定位一个cell，内部保存了多个版本的数值，默认情况下，每个数值的版本号是写入时间戳。
- timestamp：**cell内部数据是多版本的**，默认将写入时间戳作为版本号，用户可以根据自己的业务需求设置版本号，timestamp的数据类型为long类型。每个column family保留最大版本数可以单独配置，默认是3，如果读取数据时未指定版 本号，HBase只会返回最新版本的数值。如果一个cell内部数据超过最大版本数，旧的版本数据会被自动删除。
![](Pasted%20image%2020230705160638.png)
物理存储方式：
![](Pasted%20image%2020230705160730.png)
![](Pasted%20image%2020230705160817.png)
**CF理解** : 一个表的不同的column family就是有相同row key的不同的表，只是在api层包装成了一张逻辑表(可能还多一些metadata，比如partition range是一样的，方便做单row的查询，方便在多个column family做join)


# 列簇式存储引擎

不是列式存储引擎。
==同一列簇中的数据会单独存储，但列簇内数据是行式存储的。==

# 架构

HBase**按照rowkey**将数据划分成为多个固定大小的有序分区，**每个分区被称为一个“region”**，这些region会被均衡地存在在不同的节点上。如果HBase是构建在HDFS之上的，那么所有的region均会以文件的形式保存到HDFS上，以保证这些数据的高可靠存储。

HBase采用了经典的master/slave架构，**与HDFS不同的是，它的master与slave不直接互连，而是通过引入Zookeeper来让两类服务解耦，这使得Master变得完全无状态**，避免了master宕机导致的整个集群不可用。
![](Pasted%20image%2020230705161322.png)
- HMaster：HMaster可以存在多个，主HMaster由Zookeeper动态选举产生，当主HMaster出现故障后，系统可由Zookeeper动态选举出新的HMaster来接管。**HMaster主要负责协调RegionServer的均衡性和元信息的管理，为用户提供增删改查的操作。**
- RegionServer：RegionServer负责单个Region的存储和管理，例如文件的切分等，并与Client进行交互，处理读写请求。
- Zookeeper：Zookeeper内部存储着有关HBase的重要元信息和状态信息，担任着Master与RegionServer之间的服务协调角色，具体包括：集群中只有一个Master、存储Region的寻址入口、实时监控RegionServer的上线和下线信息、存储HBase的schema和table元数据等功能。
- Client：提供HBase的访问接口，并与RegionServer交互读写数据，**维护cache**以加快对HBase的访问。

## HRegionServer

某个实际存储数据的服务器。

![](Pasted%20image%2020230731223253.png)
**HBase一张表的数据会分到多台机器上的**。那HBase是怎么切割一张表的数据的呢？用的就是**RowKey**来切分，其实就是表的**横向**切割。
HLOG即为WAL。
一个HRegionServer有多个Hregion
![](Pasted%20image%2020230731223343.png)
HRegion下面有多个**Store**。
HRegion上有两个很重要的属性：`start-key`和`end-key`。
一个HBase表首先要定义**列族**，然后列是在列族之下的，列可以随意添加。
**一个列族的数据是存储**在一起的，所以==一个列族的数据是存储在一个Store里边的。==
我们可以认为HBase是基于列族存储的（毕竟物理存储，一个列族是存储到同一个Store里的）
![](Pasted%20image%2020230731223509.png)
HBase在写数据的时候，会先写到`Mem Store`，当`MemStore`超过一定阈值，就会将内存中的数据刷写到硬盘上，形成**StoreFile**，而`StoreFile`底层是以`HFile`的格式保存，`HFile`是HBase中`KeyValue`数据的存储格式。

## HMaster

**读写请求都没有经过HMaster**

HMaster会处理 HRegion 的分配或转移。如果我们HRegion的数据量太大的话，HMaster会对拆分后的Region**重新分配RegionServer**。（如果发现失效的HRegion，也会将失效的HRegion分配到正常的HRegionServer中）

HMaster会处理元数据的变更和监控RegionServer的状态。

# 读写

## 读

client请求到Zookeeper，然后Zookeeper返回HRegionServer地址给client，client得到Zookeeper返回的地址去请求HRegionServer，HRegionServer读写数据后返回给client。
![](Pasted%20image%2020230731223150.png)