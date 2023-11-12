
分为上层的zookeeper配置服务和下层的ZAB（其思想和作用都类似于raft）

实现非强一致性使用ZAB协议。

高性能：
	- 客户端异步
	- 非强一致性，可以从多个复制读
通用协调服务

追踪master，追踪配置信息，进行配置管理。从而可以专注于app开发。

# 是什么

是一个**分布式协调服务**。
功能：
- 统一命名服务
- 配置管理
- 成员管理
- Leader 选举
- 协调分布式事务
- 分布式锁
ZooKeeper 并没有直接提供这些服务，而是提供 API 供开发者实现自己需要的服务。
ZooKeeper 实际上运行在 Zab 之上。
raft和ZAB在同一层次.

# 特点

1. 层级命名空间：命名方式类似文件系统，多叉树。每个节点是个znode，主要包括:
	- data：记录用户数据，类型为字节数组，通过多副本保证存储可靠性
	- type：znode类型。分为：持久化、临时、顺序增量三种类型。（组合后为四种）
	- version：znode数据版本号，每次数据更新版本号+1
	- children：znode的子节点。除了临时znode，都可以有子节点
	- ACL：znode访问控制列表。
2. watcher：发布订阅机制。单次触发
3. session：是客户端与Zookeeper服务端之间通信通道，同一个Session中的消息是有序的。
4. ==**读多写一**==。有自己的一致性定义。写线性一致，单个client线性一致

# 复制状态机RSM

节点叫做znode。
zookeeper是一个RSM。

raft=》ZAB，serviced=》ZooKeeper。 
​ 类似Raft流程，Zookeeper对外服务时：

1. Client访问Zookeeper，create一个Znode
2. Zookeeper调用ZAB库(类似Raft库)，生成操作log，ZAB负责和Zookeeper集群的其他机器完成类似Raft的工作，日志同步、心跳保活等(整体思想和Raft类似，但实现方式不同)
    - ZAB保证日志顺序
    - 能避免网络分区、脑裂问题？怎么避免的：和raft一样使用大多数原则。
    - 同样要求操作都是具有确定性的(换言之不会有非确定性的操作在ZAB之间传递)
3. ZAB完成日志同步等工作后，回应Zookeeper
4. Zookeeper处理逻辑，响应Client的create请求

​ **在Zookeeper中，维护着由znode组成的tree树结构。**

# 高吞吐量

1. **异步**
	- 操作都是异步的，Client可以一次向Zookeeper提交多个操作。比如Client一次提交多个put(批量操作)，但是实际上leader只向持久存储写入一次(而不是写入多次)，即**Zookeeper批量整合多次写操作，只进行一次磁盘写入操作。**
2. **read by any server**
	- 其他zookeeper完成read操作时，不需要和leader有任何网络通信。所以有可能读到落后副本
	- 因此不是强一致性

如果要实现强一致性，需要每次读leader，或者其他方法。

lab3 需要是线性一致性的。

# zookeeper一致性定义

**ZooKeeper 有两个基本的一致性保证：线性写和先进先出(FIFO)的客户端请求**

1. **写是线性一致性的。只能由leader处理**。Leader 保证写操作的顺序，并且该顺序在所有 Follower 上保持一致。注意这里用了串行化(`serializable`)而不是线性一致性(`linearizability`)。
2. **单个客户端的所有请求是线性一致的。**
3. 同一个客户端所有操作（rw）以FIFO client顺序。
	<u>即如果client1先于client2请求，那么client2的处理一定在client1之后，client2的请求能观测到client1的操作结果。</u>
	- no reads from past
	- 客户端下一个读，必须读到的不能在上个读之前，cannot back in time
	- **按client请求进行写操作(write in client order)**,所有client
	- **读操作能观测到同一个client的最后一次写操作(read obverse last write from same client)**
	- **读操作会观察到日志的某些前缀(read will observe some prefix of the log)**
		就是可以观察到旧数据
	- **不能读取旧前缀的数据(no read from from the past)**
		<u>这意味着如果你第一次看到了前缀prefix 1，然后你发出读取前缀prefix 1。然后你执行第二次读取，第二次读取必须至少看到prefix 1或者更大的前缀数据(比如prefix 2)，但是绝不能看到比precix 1更小的前缀。</u>

# zookeeper一致性实现

论文没有明确说
zk client中有session
log的index为zxid。zxid是最后一个事务的标记。
写入操作的log commited后才会给client返回zxid。
当leader提交时会将zxid返回给client，放在session中。所以与session关联的是上次写入的zxid。读的时候带有zxid，至少要大于zxid才能返回读结果。

**弥补线性一致性的方法**：如果读最新的数据，先发送sync再发读请求。

实现过程：
![](Pasted%20image%2020230622234116.png)

# 帮助编程的规则

# zookeeper watch

类似于分布式观察者模式。
 ZooKeeper 实现的方式是通过客服端和服务端分别创建有观察者的信息列表。客户端调用 getData、exist 等接口时，首先将对应的 Watch 事件放到本地的 ZKWatchManager 中进行管理。服务端在接收到客户端的请求后根据请求类型判断是否含有 Watch 事件，并将对应事件放到 WatchManager 中进行管理。
 在事件触发的时候服务端通过节点的路径信息查询相应的 Watch 事件通知给客户端，客户端在接收到通知后，首先查询本地的 ZKWatchManager 获得对应的 Watch 信息处理回调操作。这种设计不但实现了一个分布式环境下的观察者模式，而且通过将客户端和服务端各自处理 Watch 事件所需要的额外信息分别保存在两端，**减少彼此通信的内容**。大大提升了服务的处理性能。
 **客户端的 Watcher 机制是一次性的，触发后就会被删除。**

# zookeeper API

使用Zookeeper的一个主要原因是，它可以是一个VMware FT所需要的Test-and-Set服务（详见4.7）的实现。Test-and-Set服务在发生主备切换时是必须存在的，但是在VMware FT论文中对它的描述却又像个谜一样，

保证了例如test and set类似的源于功能。

**version是原子性的关键**

Zookeeper的API某种程度上来说像是一个文件系统。它有一个层级化的目录结构，有一个根目录（root），之后每个应用程序有自己的子目录。比如说应用程序1将自己的文件保存在APP1目录下，应用程序2将自己的文件保存在APP2目录下，这些目录又可以包含文件和其他的目录。
这么设计的一个原因刚刚也说过，Zookeeper被设计成要被许多可能完全不相关的服务共享使用。所以我们需要一个命名系统来区分不同服务的信息，这样这些信息才不会弄混。
![](Pasted%20image%2020230623011107.png)
![](Pasted%20image%2020230623011119.png)
getData返回值和版本号。
zookeeper鼓励无锁编程。

# 模型

![](Pasted%20image%2020230715235843.png)
如图，ZooKeeper 主要由以下组件构成：

- **请求处理器(Request Processor)**：将收到的请求转为**幂等的**事务，根据版本信息，生成包含新数据、版本号和更新的时间戳的 `setDataTXN`
- **原子广播（Atomic Broadcast）**：使用 zab 协议达成共识。
- **多副本数据库（Replicated Database）**：将内存状态存储为模糊快照(fuzzy snapshot)，用来恢复状态。做快照时不锁定当前状态。

# 读写操作

**ZooKeeper 允许所有节点处理读请求，但写操作仍然只发送给 Leader**

## 写操作

任意一个Zookeeper实例均可以接受客户端的写请求，但需要进一步转发给Leader协调完成分布式写。Zookeeper采用了简化版的Paxos协议，也称之为ZAB协议，只要多数Zookeeper实例写成功，就认为本次写是成功的。

## 读操作

任意节点都可以处理读操作
缺点是可能返回过时数据--弱一致性。

# 应用

leader选举、分布式锁、分布式队列、GFS的
可应用在HDFS、HBase

- VMware-FT 中的 `test-and-set` 服务；
- GFS 中的 master 可以使用 ZooKeeper 来扮演，甚至可以提高性能，因为所有副本都能提供读服务；
- 在 MapReduce 中用来管理成员信息，谁是当前的 master、谁是 worker、worker 列表、什么工作分配给什么 worker 等等；

## leader选举

基于Zookeeper实现Leader选举的基本思想是，**让各个参与竞选的实例同时在Zookeeper上创建指定的znode，谁创建成功则谁竞选成功，并将自己的信息（host、ip、port等）写入该znode数据域，之后其他竞选者向该znode注册watcher，便于当前Leader出现故障时，第一时间再次参与竞选**，如下图所示：
![](Pasted%20image%2020230705152610.png)
基于Zookeeper的Leader选举流程如下：

（1）各实例启动成功后，尝试在Zookeeper上创建ephemeral类型的znode节点/current/leader，假设实例B创建成功，则将自己的信息写入该znode，并将自己的角色标准为Leader，开始执行Leader相关的初始化工作；
（2）除B之外的实例得知创建znode失败，则向/current/leader注册watcher，并将自己角色标注为Follower，开始执行Follower相关的初始化工作；
（3）系统正常运行，直到实例B因故障推出，此时znode节点/current/leader被Zookeeper删除，其他Follower收到节点被删除的事情，重新转入步骤1，开始新一轮的Leader选举。

## 分布式锁

利用zookeeper可以创建临时有序号节点的特性来实现分布式锁。

# 总结

- 弱一致性。精心设计的API
- 适合做配置服务。区分master，primary，leader。
- 高性能。

之后也会有弱一致性来提供高性能。



