应用场景：big sequential access.
提出弱一致性是可以接受的，并且使用弱一致性来获取更高的性能
使用单个master
故障自动恢复


master data
- filename =》 chunk handle/id
- chunk handle =》 list of chunkservers(数据副本) and chunk version

是一种分布式文件（存储）系统。不是副本复制方式。

# 概述

适合应用场景：顺序存取大文件
设计目标：大规模、性能（自动分片）、容错（复制）、一致性

# 模型架构

![](Pasted%20image%2020230716142603.png)
![](Pasted%20image%2020230716142952.png)
GFS集群包括一个master（有shadow master）和多个chunk server。client可以与master/chunk serever交互。

- master：**维护文件系统的 metadata**，它知道文件被分割为哪些 chunk、以及这些 chunk 的存储位置；它还负责 chunk 的迁移、重新平衡(rebalancing)和垃圾回收；此外，master 通过心跳与 chunkserver 通信，向其传递指令，并收集状态；GFS只有一个master
- chunk server：存储 chunk，**client 和 chunkserver 不会缓存 chunk 数据，防止数据出现不一致**；
- client：首先向 master 询问文件 metadata，然后根据 metadata 中的位置信息去对应的 chunkserver 获取数据；
- chunk：**存储在 GFS 中的文件分为多个 chunk**，chunk 大小为 64M，**每个 chunk 在创建时 master 会分配一个不可变、全局唯一的 64 位标识符(`chunk handle`)**；默认情况下，**一个 chunk 有 3 个副本**，分别在不同的 chunkserver 上；

master在内存中存储三种metadata。标记nv的需要写入到磁盘。标记v的master会在启动后查询chunkserver集群：

- namespace(即：目录层次结构)和文件名；(nv)
- 文件名 -> `array of chunk handles` 的映射；(nv)
- `chunk handles` -> 版本号(nv)、list of chunkservers(v)、primary(v)、租约(v)

## 租约

如果每次写文件都请求 master，那么 master 则会成为性能瓶颈，master 找到拥有该 chunk 的 chunkserver，并给其中一个 chunkserver 授予**租约**，拥有租约的 chunkserver 称为 `Primary`，其他叫做 `Secondary`，之后：

- master 会增加版本号，并将版本号写入磁盘，然后 master 会向 `Primary` 和`Secondary` 副本对应的服务器发送消息并告诉它们，谁是 `Primary`，谁是 `Secondary`，最新的版本号是什么；
- 在租约有效期内，对该 chunk 的写操作都由 `Primary` 负责；
- 租约的有效期一般为 60 秒，租约到期后 master 可以自由地授予租约；
- master 可能会在租约到期前撤销租约(例如：重命名文件时)；
- 在写 chunk 时，`Primary` 也可以请求延长租约有效期，直至整个写完 chunk；


# 读写

## 读

![](Pasted%20image%2020230716143245.png)- client 将 文件名+offset 转为文件名+ `chunk index`，向 master 发起请求；
- master 在内存中的 metadata 查询对应 chunk 所在的 `chunk handle` + `chunk locations` 并返回给 client；
- client 将 master 返回给它的信息缓存起来，用文件名 + `chunk index` 作为 key；(**注意：client 只缓存 metadata，不缓存 chunk 数据**)
- client 会选择网络上最近的 chunkserver 通信(Google 的数据中心中，IP 地址是连续的，所以可以从 IP 地址差异判断网络位置的远近)，并通过 `chunk handle` + `chunk locations` 来读取数据；

跨chunk的读请求，会被分解成两个读请求。

## 写

![](Pasted%20image%2020230716143457.png)
1. client 向 master 询问 `Primary` 和 `Secondary`。如果没有 chunkserver 持有租约（有租约的是primary），master 选择一个授予租约；
2. master 返回 `Primary` 和 `Secondary` 的信息，client 缓存这些信息，只有当 `Primary` 不可达或者**租约过期**才再次联系 master；
3. client 将追加的记录发送到**每一个 chunkserver(不仅仅是****`Primary`)**，chunkserver 先将数据写到 LRU 缓存中(不是硬盘！)；
4. 一旦 client 确认每个 chunkserver 都收到数据，client 向 `Primary` 发送写请求，`Primary` 可能会收到多个连续的写请求，会先将这些操作的顺序写入本地；
5. `Primary` 做完写请求后，将写请求和顺序转发给所有的 `Secondary`，让他们以同样的顺序写数据；
6. `Secondary` 完成后应答 `Primary`；
7. `Primary` 应答 client 成功或失败。如果出现失败，client 会重试，但在重试整个写之前，会先重复步骤 3-7；
这里可能会产生不一致，写的位置都是一样的，但是有的可能没有追加成功，此时追加成功的不会撤销。

# 一致性

**GFS 是宽松的一致性模型(relaxed consistency model)，可以理解是弱一致性的，它并不保证一个 chunk 的所有副本是相同的**。如果一个写失败，client 可能会重试：

- 对于写：可能有部分副本成功，而另一部分失败，副本就会不一致。
- 对于 `record append`：也会重试，**但是不是在原来的 offset 上重试**，而是在失败的记录后面重试，这样 `record append` 留下的不一致是永久的不一致，并且会让副本包含重复的数据。

**对于 metadata**：metadata 都是由 master 来处理的，读写操作通过锁保护，可以保证一致性。

## 实现强一致性

GFS 并不是强一致性的，如果这里要转变成强一致性的设计，几乎要重新设计系统，需要考虑：

- 可能需要让 `Primary` 重复探测请求；
- 如果 `Primary` 要求 `Secondary` 执行一个操作，`Secondary` 必须执行而不是返回一个错误；
- 在 `Primary` 确认所有的 `Secondary` 都追加成功之前，`Secondary` 不能将数据返回给读请求；
- 可能有一组操作由 `Primary` 发送给 `Secondary`，`Primary` 在确认所有的 `Secondary` 收到了请求之前就崩溃了。当 `Primary` 崩溃了，一个 `Secondary` 会接任成为新的 `Primary`；

# 快照

GFS 通过 snapshot 来创建一个文件或者目录树的备份，它可以用于备份文件或者创建 checkpoint（用于恢复）。GFS 使用写时复制（copy-on-write)来写快照。

当 master 收到 snapshot 操作请求后：

- 撤掉即将做快照的 chunk 的租约，准备 snapshot（相当于暂停了所有写操作）；
- master 将操作记录写入磁盘；
- master 将源文件和目录树的 metadata 进行复制，新创建的快照文件指向与源文件相同的 chunk；

## 引用计数

引用计数用来实现 copy-on-write 生成快照。当 GFS 创建一个快照时，它并不立即复制 chunk，而是增加 GFS 中 chunk 的引用计数，表示这个 chunk 被快照引用了，等到客户端修改这个 chunk 时，才需要在 chunkserver 中拷贝 chunk 的数据生成新的 chunk，后续的修改操作落到新生成的 chunk 上。


# 局限性

只有一个master。

- 耗尽内存存储metadata
- 负载过大
- 弱一致性
- master故障不会自动切换。

# 衍生

![](Pasted%20image%2020230716160445.png)
