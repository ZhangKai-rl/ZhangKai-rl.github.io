# 概述

实现：对应的开源实现是HDFS。
是什么？分布式文件存储系统，是分布式基石。
弱一致性

# 文件保存

- 大文件
大文件分成chunk = 64MB
- 超大文件
使用主从结构master保存元数据，索引指向其余保存的节点

# 架构

![](Pasted%20image%2020230704232018.png)
如图所示，GFS 集群包括：一个 master 和多个 chunkserver，并且若干 client 会与之交互。

主要架构特性：

- **chunk**：存储在 GFS 中的文件分为多个 chunk，chunk 大小为 64M，每个 chunk 在创建时 master 会分配一个不可变、全局唯一的 64 位标识符(`chunk handle`)；默认情况下，一个 chunk 有 3 个副本，分别在不同的 chunkserver 上；
- **master**：维护文件系统的 metadata，它知道文件被分割为哪些 chunk、以及这些 chunk 的存储位置；它还负责 chunk 的迁移、重新平衡(rebalancing)和垃圾回收；此外，**master 通过心跳与 chunkserver 通信**，向其传递指令，并收集状态；
- **client**：首先向 master 询问文件 metadata，然后根据 metadata 中的位置信息去对应的 chunkserver 获取数据；
- **chunkserver**：存储 chunk，**client 和 chunkserver 不会缓存 chunk 数据，防止数据出现不一致**；

## master

![](Pasted%20image%2020230704232238.png)
为了简化设计，GFS 只有一个 master 进行全局管理。

master 在内存中存储 3 种 metadata，如下。**标记 nv(non-volatile, 非易失) 的数据需要在写入的同时存到磁盘**，标记 v 的数据 master 会在启动后查询 chunkserver 集群：
- namespace(即：目录层次结构)和文件名；(nv)
- 文件名 -> `array of chunk handles` 的映射；(nv)
- `chunk handles` -> 版本号(nv)、list of chunkservers(v)、primary(v)、租约(v)

# 租约

如果每次写文件都请求 master，那么 master 则会成为性能瓶颈，master 找到拥有该 chunk 的 chunkserver，并给其中一个 chunkserver 授予**租约**，拥有租约的 chunkserver 称为 `Primary`，其他叫做 `Secondary`，之后：

- master 会增加版本号，并将版本号写入磁盘，然后 master 会向 `Primary` 和`Secondary` 副本对应的服务器发送消息并告诉它们，谁是 `Primary`，谁是 `Secondary`，最新的版本号是什么；
- 在租约有效期内，对该 chunk 的写操作都由 `Primary` 负责；
- 租约的有效期一般为 60 秒，租约到期后 master 可以自由地授予租约；
- master 可能会在租约到期前撤销租约(例如：重命名文件时)；
- 在写 chunk 时，`Primary` 也可以请求延长租约有效期，直至整个写完 chunk；

# 读写文件

## 读文件

![](Pasted%20image%2020230704232440.png)
- client 将 文件名+offset 转为文件名+ `chunk index`，向 master 发起请求；
- master 在内存中的 metadata 查询对应 chunk 所在的 `chunk handle` + `chunk locations` 并返回给 client；
- **client 将 master 返回给它的信息缓存起来**，用文件名 + `chunk index` 作为 key；(**注意：client 只缓存 metadata，不缓存 chunk 数据**)
- client 会选择网络上最近的 chunkserver （不论primary/secondary？）通信(Google 的数据中心中，IP 地址是连续的，所以可以从 IP 地址差异判断网络位置的远近)，并通过 `chunk handle` + `chunk locations` 来读取数据；

## 写文件

![](Pasted%20image%2020230704232458.png)
如图，写文件可分为 7 步：
1. client 向 master 询问 `Primary` 和 `Secondary`。如果没有 chunkserver 持有租约，master 选择一个授予租约；
2. master 返回 `Primary` 和 `Secondary` 的信息，client 缓存这些信息，只有当 `Primary` 不可达或者**租约过期**才再次联系 master；
3. client 将追加的记录（WAL?）发送到**每一个 chunkserver(不仅仅是****`Primary`)**，chunkserver 先将数据写到 LRU 缓存中(不是硬盘！)；
4. 一旦 client 确认每个 chunkserver 都收到数据，client 向 `Primary` 发送写请求，`Primary` 可能会收到多个连续的写请求，会先将这些操作的顺序写入本地；
5. `Primary` 做完写请求后，将写请求和顺序转发给所有的 `Secondary`，让他们以同样的顺序写数据；
6. `Secondary` 完成后应答 `Primary`；
7. `Primary` 应答 client 成功或失败。如果出现失败，client 会重试，但在重试整个写之前，会先重复步骤 3-7；

# 一致性

弱一致性
**GFS 是宽松的一致性模型(relaxed consistency model)，可以理解是弱一致性的，它并不保证一个 chunk 的所有副本是相同的**。如果一个写失败，client 可能会重试：

- 对于写：可能有部分副本成功，而另一部分失败，副本就会不一致。
- 对于 `record append`：也会重试，但是不是在原来的 offset 上重试，而是在失败的记录后面重试，这样 `record append` 留下的不一致是永久的不一致，并且会让副本包含重复的数据。
![](Pasted%20image%2020230704232950.png)
如图，先解释图上 `defined` 和 `consistent` 两个概念：

- `defined`：一个文件区域在经过一系列操作之后，client 可以看到数据变更写入的所有数据；
- `consistent`：所有 client 不论从哪个副本中读取同一份文件，得到的结果都是相同的；

**对于 metadata**：metadata 都是由 master 来处理的，读写操作通过锁保护，可以保证一致性。
**对于文件数据**：
- 在没有并发的情况下，写入不会互相干扰，那么则是 `defined`；
- 在并发的情况下，成功的写入是 `consistent` 但不是 `defined`；
- 顺序写和并发写 `record append` 能够保证是 `defined`，但是在 `defined` 的区域之间会夹杂着一些不一致的区域；
- **如果出现写失败，副本之间会不一致**；

## 转换成强一致性

GFS 并不是强一致性的，如果这里要转变成强一致性的设计，几乎要重新设计系统，需要考虑：

- 可能需要让 `Primary` 重复探测请求；
- 如果 `Primary` 要求 `Secondary` 执行一个操作，`Secondary` 必须执行而不是返回一个错误；
- 在 `Primary` 确认所有的 `Secondary` 都追加成功之前，`Secondary` 不能将数据返回给读请求；
- 可能有一组操作由 `Primary` 发送给 `Secondary`，`Primary` 在确认所有的 `Secondary` 收到了请求之前就崩溃了。当 `Primary` 崩溃了，一个 `Secondary` 会接任成为新的 `Primary`；

## 为什么使用弱一致性

为什么 Google 最初选择弱一致性呢？教授在课堂上给出一种解释。

> Robert教授：如果你通过搜索引擎做搜索，20000 个搜索结果中丢失了一条或者搜索结果排序是错误的，没有人会注意到这些。这类系统对于错误的接受能力好过类似于银行这样的系统。当然并不意味着所有的网站数据都可以是错误的。如果你通过广告向别人收费，你最好还是保证相应的数字是对的。

# 容错

- **快恢复**：master 和 chunkserver 都设计成在几秒钟内恢复状态和重启；
- **chunk 副本**：如前面提到的，chunk 复制到多台机器上；
- **master 副本**：master 也会被复制来保证可用性，**称为 shadow-master；**

# 快照

GFS 通过 snapshot 来创建一个文件或者目录树的备份，它可以用于备份文件或者创建 checkpoint（用于恢复）。GFS 使用写时复制（copy-on-write)来写快照。

当 master 收到 snapshot 操作请求后：

- 撤掉即将做快照的 chunk 的租约，准备 snapshot（相当于暂停了所有写操作）；
- master 将操作记录写入磁盘；
- master 将源文件和目录树的 metadata 进行复制，新创建的快照文件指向与源文件相同的 chunk；

# 局限性

**GFS 最严重的局限性就在于它只有一个 master 节点**