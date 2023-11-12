b站讲解视频

# 概述

论文：《Bigtable: A Distribted Storage System for Structured Data》---2006 OSDI
是什么？分布式结构化数据存储系统(KV Store)，**NoSQL（即不支持SQL）。**
实现产品：HBase、Cassandra（一半）
bigtable在GFS和Chubby的基础上首次于工程中实现了可用的LSM树结构。

# 数据结构

KV
- bigtable的key：行关键字（row）+列关键字（column）+时间戳（time）
- bigtabe的value：为解析的byte数组（比如html格式的网页、转成二进制的图片），类似关系型数据库的blob类型

行关键字可以理解为主键，列关键字可以理解为列名。

## 举例

google web index
![](Pasted%20image%2020230704230440.png)
![](Pasted%20image%2020230704230618.png)
列族

![](Pasted%20image%2020230704230820.png)

# bigtable API

不重要，是特殊设计的API
不支持SQL，因此要提供API。
功能：
1. 建表、删表等DDL
2. 对单行数据的增删查功能，不支持修改操作（只支持单行事务，即跨行后无法保证ACID。只是个KV Store，不是分布式事务模型（Spanner、oceanbas、tidb））
3. 范围扫描

# bigtable的架构和高可用

**重要**
融合了GFS和Chubby两个技术底座，首次搭建了LSM树结构
![](Pasted%20image%2020230704231408.png)

## tablet

一个tablet内部是一堆KV对。
将key按字典序排列，然后按范围分片，成为一个**tablet**，物理上对应GFS中的一个文件，有全局唯一的文件名，逻辑上是一个LSM树（存储的是实际数据：KV对）。
![](Pasted%20image%2020230704231659.png)
LSM树契合GFS的追加写操作。

## key->tablet文件名

这里是由Tablet Server完成的
设置一个三层的、类似b+树的结构来存储**tablet位置信息**。
![](Pasted%20image%2020230704233942.png)


# 查询Key

![](Pasted%20image%2020230704234303.png)
图下半部分的GFS，是目标tablet内部的LSMtree。
为提高速度，将缓存放在client。