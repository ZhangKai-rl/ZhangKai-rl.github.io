参考笔记：[FaRM论文笔记 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/597272185)

# 概览

## 论文

No compromises: distributed transactions with consistency, availability, and performance-----2015

## 简介

高性能事务，FaRM在TATP基准上，使用90台机器，获取1.4亿个事务/秒。作为对比，Spanner能处理10～100个事务/秒。
Spanner是全球范围内进行同步复制的事务系统，而FaRM的一切都在同一个数据中心内运行。

**FaRM提供了严格的可串行化，类似于Spanner提供的外部一致性。FaRM整体目标是获取高性能。**

FaRM = Fast + Remote + Memory

- transactions+replication+sharding的实现
- 实现读写事务的乐观并发控制OOC
- FaRM 直接写入RAM不需要写入磁盘。RAM不能持久化，因此我们使用了UPS

​ Spanner目前仍是活跃使用中的系统，而FaRM是微软研究的产品。

## 高性能策略

- 单数据中心(One data center)：系统只部署在单个数据中心内。spanner多个数据中心
- 数据分片(shard)：如果不同事务访问不同分片，那么互不影响，可完全并行。
- 非易失性DRAM(non-volatile DRAM)：采用非易失性DRAM硬件，避免不得不写入稳定存储设备时的写入性能瓶颈。所以在他们的设计中，你不必写入关键路径到固态硬盘或磁盘。这消除了存储访问成本（IO），接着需要解决CPU瓶颈和网络瓶颈。
- 内核旁路技术(kernel bypass)：这避免了操作系统与网卡交互。
- 远程直接数据存取(RDMA)：他们使用具有RDMA特殊功能的网卡，这允许网卡从远程服务器读取内存，而不必中断远程服务器。这为我们提供了对远程服务器或远程内存的低延迟网络访问。
- 乐观并发控制(OOC, Optimistic Concurrency Control)：**为了充分利用以上这些提速手段（RDMA），他们采用乐观并发控制**。对比前面经常谈论的悲观并发控制方案其需要在事务中访问对象时获得锁，确保到达提交点时拥有所有访问对象的锁，然后再进行提交；而使用乐观并发控制不需要锁，尤其在FaRM中，不需要在读事务中获取锁，当你进行commit时，你需要验证读取最近的对象，是则提交，不是则中止且可能伴随重试。FaRM使用乐观并发控制的原因，主要是被RDMA的使用所需动的。

# farm setup

- 高速网络：数据中心中，90台机器通过高速数据中心网络连接，内部是一个交换网络。
    
- 数据分片：得不到不同机器上。根据分片的级别，被称为**区域(region)**，region大小2GB。**region分布在内存DRAM中，而不是在磁盘上**。所以数据库的所有数据集必须和部署的机器的DRAM相适应。如果数据集大于所有机器的内存总量，则必须配置更多的机器和DRAM。
    
- 主备复制：如果宕机DRAM数据会丢失，所以他们也有主备复制(P/B replication)方案。这里通过配置管理器(configuration manager, CM)配合zookeeper，维护region到primary、backups机器的映射。
    
- 不间断电源UPS：由于整个系统部署在单个数据中心，数据中心断电就会导致所有DRAM丢失数据，所以DRAM配置在有不间断电源(UPS)的机器上。如果发生断电，DRAM通过UPS即使将数据存储到SSD上，或者只是刷新内存中的内容(region、transaction、log等)，之后恢复电源再从SSD加载DRAM存储的内容。这里仅在故障恢复时使用SSD。
    

​ 在区域region中有一些对象object，可以把region想象成一个2GB的字节对象数组。

- 对象拥有唯一的标识符oid，由区域编号和区域内的访问偏移量构成`<region #, offset>`
	region全局唯一，每台机器有自己的地址空间
    
- 对象有元数据信息，对象头部包含一个64bit的数字，底63bit表示版本号，顶1bit表示锁。这个头部元信息在乐观并发控制中起重要作用

解决了IO瓶颈，和电源故障。
接下来要降低CPU使用率，CPU瓶颈和网络瓶颈

# 内核旁路



