# 概览

预习：Spanner: Google’s Globally-Distributed Database----2012论文
复杂的论文。
这节课将spanner论文-一个具体的分布式事务系统的设计实现。比较重要的部分。

## Spanner

现在仍然在使用。
Spanner是谷歌公司研发的、可扩展的、多版本、全球分布式、同步复制**数据库**。它支持外部一致性的分布式事务。
ACID、可串行化、失败原子性。

- Wide-area transation：支持广域的事务，即跨越多个国家/地区的分布式事务
    
    - read-write trasaction using 2-phase commit, Paxos：读写事务使用两阶段提交实现，协议的参与者都是Paxos组(2PL+2PC+Paxos)
    - Read-only transaction：**只读事务可以在任意数据中心运行**，大约比读写事务快10倍（Spanner针对只读事务做优化）。只读事务优化：
        - snapshot isolation：使用快照隔离技术，使得只读事务能更快
        - synchronized clocks：实现依赖**同步时钟**，这些时钟基本是完全同步的，事务方案必须处理不同机器的一些时间漂移或错误容限（具体的实现称为**TrueTime API**）
- Wide-used：Spanner是一项云服务，你可以作为Google的客户使用。比如Gmail的部分功能可能有依赖Spanner。

# 高层组织架构

在不同地方有数据中心，每个数据中心运行着kv server。
将数据进行分片到各地kv server中，每个分片在kv server有着复制，构成paxos组。
![](Pasted%20image%2020230718142706.png)

- multiple shards：spanner进行分片，获取更高的并行度。如果事务涉及不同的分片，事务之间互不依赖，可并行处理。
- paxos per shard：每个shard分片都有对应的Paxos组，用于replication复制。paxos/raft基于大多数原则，因而速度慢的机器不会对整体性能产生太大影响。也增加了容错性，容许部分服务器宕机。
	- 1. 容错
	- 2. 大多数原则降低通信成本。
- Replica close to clients：Replica通常部署在靠近client用户群体（这里client主要指公司内部的后端服务，因为Spanner整体而言是对内的基础架构服务）的地理位置。目标是能让client智能地访问最近的Replica。**通常只读事务可以由本地Replica执行，而不需要与其他数据中心进行任何通信**。

# spanner实现难点

1. Read of local replica yield lastet write：**从本地Relica读取，但要求看到最新的写入**
	- 事实上，**Spanner追求的是比线性强一致性更强的性质（外部一致性）**。（zookeeper也允许读任何节点，但是并不保证能读到其他client的最新写入，只保证读取当前session下最新的写入，即弱一致性）
2. 支持跨分片事务（分布式事务原子性实现）
3. 事务必须可串行化（只读和读写事务）。对于读写事务(read-write tx)，Spanner使用2PL和2PC。

# 读写事务

使用悲观锁2PC实现。单机事务可以使用1PC.对于Single Split与Multiple Split事务，Spanner分别采用了1PC与2PC

**读写事务只能发生在leader**

Spanner的client不是用户，而是比如说gmail的服务器这种。

## 读写事务举例

跨分片转账事务：

每个分片服务器都有锁表（还有事务管理器，只有paxos组中的leader有），表明记录被那个客户端持有锁。锁表不是复制的，只存在于paxos组的leader中，因而如果leader宕机事务需要重新开始（prepare之前），因为锁表丢失了。锁表不是复制的，使得读操作很快。

- **在执行只读事务时，直接读数据分片的复制的Paxos组的leader(即事务如果目前还只有read操作，还没有write操作时，不需要事务协调者参与)**
- **当事务涉及写操作后（要涉及2PC），当前事务需要经过事务协调者Paxos组leader，其协调和数据分片Paxos组leader之间的写操作**
- **lock table仅在Paxos组的leader中存放记录，不在Paxos组内复制(因为不需要复制锁表，加快了读操作)**
    - **只读事务下分片Paxos组的leader在事务过程中宕机，则事务中止，需要重新开始(因为锁信息丢失了，只有分片Paxos组leader记录锁信息)**
    - **事务中后续涉及写操作，升级为读写事务，prepare阶段前崩溃，锁信息会丢失（因为还只在分片Paxos组的leader的锁表记录中），prepare阶段后，某些记录被加锁的状态会通过log同步到Paxos组其他成员，锁记录不会丢失**。
- **2PC的事务协调者也是Paxos组，增强容错性(降低事务协调者宕机后，导致参与者必须等待协调者恢复后才能commit事务的事故概率)**


**2PC：第一步是获取锁、写日志，第二步是apply变更（安装日志）、释放锁**

![](Pasted%20image%2020230718150022.png)

|Client|事务协调者(Paxos组)|Sa(A数据分片Paxos组)|Sb(B数据分片Paxos组)|
|---|---|---|---|
|事务tid，向Sa的Paxos组leader发起数据分片A的读请求，向Sb的Paxos组leader发起数据分片B的读请求||||
|||Sa根据2PL，leader对X加锁，Owner是Client|Sb根据2PL，leader对Y加锁，Owner是Client|
|Client向事务协调者发起X=X-1，Y=Y+1的操作||||
||收到X和Y更新的操作，通过2PC流程，分别发送X更新操作和Y更新操作到分片A和分片B的Paxos组leader|||
|||根据2PC流程，leader发现X已经有锁，将读锁升级为写锁，根据WAL进行事务操作的log|根据2PC流程，leader发现Y已经有锁，将读锁升级为写锁，根据WAL进行事务操作的log|
||发起prepare请求，确认Sa和Sb是否可以准备事务提交，同样需要Paxos组log、复制|||
|||检查已拥有锁，log记录事务状态和2PC状态，lock持有状态（不是锁表）等信息，以及**在Paxos组内同步**log，确认整个Paxos组可以准备提交，leader回应OK|检查已拥有锁，log记录事务状态和2PC状态，lock持有状态等信息，以及**在Paxos组内同步**log，确认整个Paxos组可以准备提交，leader回应OK|
||发起commit(tid)请求，要求提交事务，同样需要Paxos组log、复制|||
|||**Paxos组**实际install log，执行log记录的事务操作，leader释放锁，leaer回应OK|**Paxos组**实际install log，执行log记录的事务操作，leader释放锁，leader回应OK|

是对上周的升级（2PC+2PL），都加入了paxos组来保证了容错和可用性(2PC +2PL+PAXOS)。
**最大的区别就是事物协调者(coordinator)和参与者(participant)都是Paxos组**。

# 只读事务

论文提到只读事务基本上比读写事务快10倍，读写事务在数百毫秒的量级，而只读事务在5～10毫秒的量级。

**spanner只读优化：**

- **Fast, Read from local shards：通过==只读只从本地服务器（最近）读取数据==，实现低延迟(无需额外和其他分片通信，实现一致性/串行化是个难点)** 。问题：本地的不是领导则么办？怎么确定最新？
- **no locks：不需要锁(读写事务需要锁，会互相阻塞；而只读事务不会阻塞读写事务)**
- **no 2PC：不需要两阶段提交(同时意味着不需要广域wide-area通信)**

## 只读事务正确性

- **可串行化**
	- 即几个读写事务和只读事务在一起运行时，能够按照结果对事务得到一个排序。只读事务要么读取到某个读写事务的所有写入结果，要么其写入结果一个都看不到，但必须保证只读事务不能看到读写事务的中间结果。
- 更强：==**外部一致性(external consistency)**==，是事务级别的属性，是强一致性。
	- **如果事务1在事务2之前commit，那么事务2一定能够读取到事务1的变更。**。
	- ​ **这里外部一致性可以理解为，在可串行化基础上再加上实时性的要求。实际上外部一致性贴近于线性一致性(linearizability)**。线性一致性是针对单独读写而不是事务。
		​ _补充说明：线性一致性要求如果操作2在操作1结束后才开始，那么操作2必须排在操作1之后。详情见“7.7 线性一致性/强一致性(Linearizability/strong consistency)”_

​![](Pasted%20image%2020230718151242.png)

## 只读事务实现

使用快照隔离，乐观

### 错误示例

always read lastet committed value，总是读最后提交的数据。
![](Pasted%20image%2020230718152954.png)

|事务|时间戳10|时间戳20|时间戳30|
|---|---|---|---|
|T1|set X=1, Y =1, Commit|||
|T2||Set X=2, Y=2, Commit||
|T3|Read X (now X=1)||Read Y (now Y=2), Commit|

​ 显然，如果总是读最后提交的数据，是不正确的，T3观察到了来自不同事务的写入，而不是得到一致的结果。（不符合可串行化，理论上T3要么X和Y都读取到1，要么都读取到2，要么都读取到X和Y的原始值假设为0）。

### 快照隔离（实现无锁读）

​ **Spanner采用快照隔离(Snapshot Isolation)方案实现只读事务的正确性(Correctness)**。

先讨论快照隔离概念，不涉及spanner广域。

​ 快照隔离具备以下特征：

- ==**为事务分配时间戳(Assign TS to TX)**==
    - 读写事务：在**开始提交**时分配时间戳（由主副本使用true time生成）（R/W：commit）
    - 只读事务：在**事务开始**时分配时间戳（R/O：start）
- **按照时间戳顺序执行所有的事务(Execute in TX order)**
- **每个副本保存多个键的值和对应的时间戳(Each replica stores data with a timestamp)**。多版本，每次更新保存一个版本。例如在一个replica中，可以说请给我时间戳10的x的值，或者给我时间戳20的x的值。**有时这被称为多版本数据库(multi-version database)或多版本存储(multi-version storage)**。对于每次更新，保存数据项的一个版本，这样你可以回滚(so you can go back in time)。


#### 举例：

|事务(开始的时间戳)|时间戳10|时间戳20|时间戳30|
|---|---|---|---|
|T1 (10)|set X=1, Y =1, Commit|||
|T2 (20)||Set X=2, Y=2, Commit||
|T3 (15)|Read X (X=1)||Read Y (Y=1), Commit|

​ 这里T3需要读取时间戳15之前的最新提交值，这里时间戳15之前的最新值，只有T1事务提交的写结果，所以T3读取的X和Y都是1。因为所有的事务按照全局时间戳执行，所以保证了我们想要的线性一致性或串行化。

每个replica给数据维护了version表，可以猜想结构大致如下：

|数据（假设X和Y在同一个replica中）|版本（假设用时间戳表示）|值|
|---|---|---|
|X|@10|1|
|X|@20|2|
|Y|@10|1|
|Y|@20|2|
​ 所以当读请求到达replica时，replica可以根据事务请求的时间戳，查询维护的数据记录表，决定该返回什么版本的数据。

旧数据问题：
​ **这里Spanner还依赖=="safe time"机制==，保证读操作从本地replica读数据时，能读取到事务时间戳之前的最新commit数据。**

- **Paxos按照时间戳顺序发送所有的写入操作(Paxos sends write in timestamp order)**
- **读取X数据的T时间戳版本数据之前，需等待T时间戳之后或相等的任意数据写入(Before Read X @15, wait for Write >= @15)**
    - **这里等待写入，同时还需要等待在2PC中已经prepare但是还没有commit的事务(Also wait for TX that have prepared but not committed)**

# Spanner核心--时钟偏移(Clock drift)问题

​ Spanner的实现和时间戳强相关，要求不同机器的时钟必须准确。不同的参与者在时间戳顺序上需要达成一致。在系统中的任何地方，时间戳都必须是相同的。
​ **实际上，时钟和时间戳只对只读事务非常重要(matters only for R/O txns)**。

场景分析：

1. 时间戳偏大(TS too large)
	- 比如只读事务T3在实际时间戳15执行，机器却认为在时间戳18执行，那么只读事务需要等待等待更长时间，即更大的时间戳之后的write出现后，才能得到read数据。所以时间戳偏大，只是**单纯影响性能**。
2. 时间戳偏小(TS too small)
	- 比如只读事务T3在实际时间戳15执行，机器的时钟错误导致认为T3在时间戳9执行，那么T3看不到T1(假设在时间戳10完成commit)的写入，这**打破了外部一致性(external consistency)**。

## 时钟同步

每个数据中心都有一些time master，大多数time master都配备了GPS，剩下的少数配置了原子钟。所有DB会每30秒去向这些服务器校准一次自己的时间，以保证自己的时间也与真实绝对时间几乎完全相同，这就保证了每台DB server的时间与真实绝对时间的误差都在几毫秒内。

1. spanne使用**原子钟(atomic clocks)**物理设备
    
    Spanner在机器上使用原子钟，以获取更精准的时间
    
2. **与全球时间同步(synchronize with global time)** ，GPS

	Spanner让时钟和全球时间同步，确保所有机器的时钟在全球时间上一致。使用GPS全球定位系统广播时间，作为同步不同原子钟的一种方式，然后保持它们同步运行。

![](Pasted%20image%2020230718172543.png)

## 时钟偏移解决方法

​解决时钟偏移(Clock drift)的方案，即不使用真实时间的时间戳，或者不仅仅是使用纯粹的时间戳，而是**使用时间间隔**。

True Time方案：
当机器向time master询问时间时，并不会返回一个时间值，而是返回一直TT区间的值，这个区间由一个earliest time和latest time组成，精准的时间必然处在这个TT区间内。

只读型事务被分配偏小的时间戳导致外部一致性被破坏的问题，通过两条规则来解决了，分别为Start Rule和Commit Wait Rule。

Start Rule：为事务分配时间戳就是返回的TT区间中的latest time来赋值，用TT.now().latest赋值，**这确保事务被赋值的时间戳是比真实的时间要大一些**。对于只读型事务而言，时间戳应该在开始的时候就赋予；对于读写型事务而言，时间戳应该在提交的时候再赋予。

Commit Wait Rule：这个规则只针对于读写型事务，由于事务被分配的时间戳为TT区间中的latest，实际是要大于真实时间的，后续则需要等待真实时间大于这个时间戳后才能提交该读写型事务。这确保了读写型事务被提交的那个时间的数值是比被分配的时间戳要大。

Spanner的TrueTime是利用GPS时钟和原子钟实现的, 可以提供非常精确的时钟, 误差在1ms~7ms之间.  
==TT.now()返回一个区间[earlist, latest], 保证abs time(绝对时间)是在区间以内.  ==
TT.after(t)判断一个时间是否已经成为过去时, 是对TT.now()的简单封装:


修正规则：

- **当前时间规则(start rule: now.lastest)：当前时间，取时间区间的lastest值**。保证一定在真实时间之后。
    
    - 读写事务：在**开始提交**时分配当前时间`now.lastest`作为时间戳（R/W：commit）
    - 只读事务：在**事务开始**时分配当前时间`now.lastest`作为时间戳（R/O：start）

- **提交等待规则(Commit wait rule)**
    
    我们会在事务中推迟(delay)提交。如果(读写)事务在start commit时获得某个时间戳，延迟提交直到时间刚比`now.earliest`大的时刻。
    
场景举例：

|事务(实际开始时间)|实际时间戳9|实际时间戳10|实际时间戳12|
|---|---|---|---|
|T1 (7)|set X=1, Commit (这里先不关心T1的细节，假设实际时间戳为9时完成提交)|||
|T2 (8)||Set X=2, 在实际时间戳9时获取当前时间，返回时间区间假设[8, 10]，那么需要延迟等待到时间戳10时提交Commit||
|T3 (11)|||Read X (X=2), Commit|

- 这里先不考虑T1读写事务的细节，这里假设T1在实际时间戳9完成提交。
- 而T2在实际时间戳8时开始读写事务，假设在实际时间戳9时获取当前时间，返回时间戳区间`[8,10]`，于是需要等到时间戳10时执行Commit。
- T3为只读事务，假设在实际时间戳9时获取当前时间，返回时间戳区间`[8,11]`，于是等到实际时间戳11时执行Read操作，于是读到最新的X数据为2（能看到T2的提交结果），符合外部一致性(External Consistency)。


# 总结

- 读写事务(R/W tx)是全局有序的(可串行化+外部一致性)，因为使用2PL+2PC
- 只读事务(R/O tx)只读本地replica，通过快照隔离(snapshot isolation)确保读正确性
    - 快照隔离，保证可串行化
    - 按时间戳顺序执行只读操作，保证外部一致性
        - 时间依赖完全同步的时钟，当前时间now采用时间间隔/区间表示`[earliest, lastest]`

​ 通过以上技术/手段，只读事务非常快，而读写事务平均100ms。读写事务慢的原因是跨国家/地区进行分布式事务。