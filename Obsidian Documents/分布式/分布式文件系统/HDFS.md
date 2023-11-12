# 概述

java语言编写
强一致性
hadoop核心组成，块级分布式存储服务。
HDFS通过统一的命名空间目录树来定位文件。

# 特点

- 典型master/slave架构
- 物理分块存储（block机制），默认为128MB
- 命名空间：HDFS支持传统的层次型文件组织结构，由namenode维护
- namenode元数据管理：把目录结构和文件分块位置信息叫做元数据。NameNode的元数据记录每一个文件所对应的block信息（block的ID以及所在DataNode节点的信息）
- datanode数据存储：文件的各个block具体存储管理由DataNode节点承担。一个block会有多个DataNode存储，DataNode会定时的向NameNode汇报自己持有的block信息。
- 副本机制：默认为3。高可用性
- 一次写入，多次读出：HDFS是设计成适应一次写入、多次读出的场景。且不支持文件的随机修改。（支持文件追加，不支持随机修改）。正因为如此，HDFS适合做大数据分析的底层存储服务，并不适用做网盘等应用（修改不方便、延迟高、网络开销大、成本太高）
- 适合存储大文件，对小文件支持不友好。

# 架构

主从架构：namenode/datanode。使用RPC通信
![](Pasted%20image%2020230705132429.png)

**NameNode HDFS集群的管理者，Master，存放元数据**：
- **管理文件系统命名空间，以目录树的形式维护HDFS的系统目录和文件**
- **保存元数据**：文件目录名及层级关系；文件目录所有者及其权限；每个文件块的名及文件由哪些块组成
- 管理datanode：datanode需要向namenode汇报心跳
- 维护副本策略
- 记录文件块（block）的映射信息。该文件切成了几个块，每个块放在那里
- 负责处理客户端的读写请求
- 高可用状态同步：有备用namenode（secondary namenode）；主从Namenode并不是通过强一致的协议保证状态一致的，而是通过第三方的文件共享存储系统，主Namenode将EditLog日志写入共享存储系统，从Namenode读取这些日志并执行修改操作。

**DataNode：NameNode下达命令，DataNode执行实际的操作，Slave节点**
- 保存实际的块数据和块校验
- 根据namenode指示进行创建、删除、复制等操作。
- 负责数据块的读写
- 每3s向namenode发送心跳

 **Client客户端**
- **client负责将文件切分成block**
- 与NameNode交互，获取文件的位置信息等**文件元数据**
- 与DataNode进行交互，读取或写入数据
- Client可以使用一些命令来管理HDFS或访问HDFS

## NameNode

### 元数据

- edits（编辑日志文件）：保存了自最近检查点之后的所有文件更新操作
- fsimage（元数据检查点镜像文件）：保存了文件系统中所有的目录和文件信息。如：某个目录下有哪些子目录和文件；以及文件名、文件副本数、文件由哪些blocks组成等。
active namenode由一份最新的元数据（edits + fsimage）
standby namende在检查单定期将内存中的元数据保存到fsimage文件中

## SecondaryNameNode

namenode在更新内存中的元信息之前都会先将操作写入edits文件，在namenode重启的过程中，edits和fsimage合并一起，此过程会影响Hadoop重启速度。secondarynamenode就是为了解决这个问题!
其作用就是定期的合并edits和fsimage文件
![](Pasted%20image%2020230705174619.png)

# 问题

## 单点问题

只有一个namenode，因此会有单点问题。
解决：
- active和standby NameNode（两个nn）。两个nn为了数据同步，会通过一组journalnode的独立进程进行相互通信。
- namenode集群（多组acive和standby）+zookeeper提升可靠性。故障自动切换namenode

# Shell命令

分为用户命令和管理员命令

# 读写文件

## 读流程

![](Pasted%20image%2020230705134600.png)
- **流程说明**：

- 客户端通过Distribute FileSystem向NameNode请求下载文件
- NameNode通过查询元数据，找到数据块所在DataNode地址。block id + datanode地址
- 挑选一台DataNode（就近原则、然后随机）服务器，请求读取数据
- DataNode传输数据给客户端，（从磁盘中读取输入流，以packet为单位来做校验）
- 客户端以packet为单位接收，先在本地缓存，然后写入目标文件

## 写流程

![](Pasted%20image%2020230705135243.png)
**流程说明**:

- 客户端通过Distribute FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否存在，父级目录是否存在。
- NameNode返回给客户端是否可以上传
- 客户端请求第一个block上传到哪些DataNode上（副本）
- NameNode返回N个DataNode节点，分别为dn1, dn2, dn3......dnn
- 客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求后继续调用dn2，然后dn2调用dn3，将通信管道建立完成。
- dn1、dn2、dn3逐级应答客户端。
- 客户端开始向第一个DataNode上传block（先从磁盘读取存放到本地缓存），以packet为单位，**dn1收到一个packet就会传给dn2, 然后dn2传给dn3。dn1每传一个block会放入一个确认队列等待确认。**
- 当一个block传输完毕后，客户端再次请求NameNode上传第二个block的服务器。（重复执行3-7步骤）

namenode在更新内存中的元信息之前都会先将操作写入edits文件。


# 优缺点

## 缺点

很多小文件时，namenode压力大。