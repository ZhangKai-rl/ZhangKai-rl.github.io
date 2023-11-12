# 预习

1997论文。
背景为网络文件系统。在一组用户间共享文件。

三个论文重要想法：

- 缓存一致性协议
- 分布式锁
- 分布式崩溃恢复

# 网络文件系统

 ![](Pasted%20image%2020230716164558.png)
  传统的网络文件系统（涉及负载均衡和分片），对外提供open、create等文件操作函数，服务端内部实现复杂，而client端除了函数调用和缓存以外，基本不做其他工作。
 Frangipani的file server运行在客户端（称作workstation）。这里所有的client共享一个虚拟磁盘(virtual disk)，这个虚拟磁盘内部使用Petal实现，由数个机器构成，机器复制磁盘块(disk blocks)，内部通过Paxos共识算法保证操作按正确顺序应用等。
 
 这样设计易于扩展，克服了扩展时文件服务器会成为瓶颈的问题。

# 设计目标

- caching：缓存，不是所有数据都保留在Petal(Frangipani文件服务器（共享磁盘）部分被称为Petal)中。**client采用回写式缓存(write back-cache)取代直写式(write-through)**。即操作发生在client本地的cache中，未来某时刻被传输到Petal。缓存在本地的file server中
    
- strong consistency：强一致性。即希望client1写file1时，其他client2或者workstation能够看到file1的变动。
    
- perference：高性能。

# Frangipani实现挑战

​ 假设WS1工作站1执行read f的操作，之后通过本地cache对f通过vi进行操作。这时需要考虑几个可能发生的场景如何处理：

1. 缓存一致性。WS2 cat f
2. 原子性。WS1创建d/f，WS2创建d/g。保证不会影响对方。
3. 崩溃恢复。WS1文件操作是崩溃。
接下来解决这些问题

# Frangipani---缓存一致性

使用lock server（容错的分布式锁，可以由zookeepe提供，论文中用paxos实现）。
lock server里面有个全局表，保存inode和拥有其锁的工作站。每个WS也要维护一张表，保存inode和锁的状态。锁的状态有：busy（在操作）、idle（粘性锁）。sticky lock是一种优化，防止多次通信。

规则：缓存一个文件，需要先获得分布式锁。

缓存中的数据每隔30s会写回petal

## Frangipani---协议

WS和LS之间通信有四种信息：请求锁、授予锁、撤销锁(revoke,ls发给ws，表示想要收回锁)、释放锁（busy -> idle）。

锁流程：

1. WS1向LS请求锁(requesting a lock)，要求访问文件f
2. LS查询自己维护的file inode => owner表，发现f没有被任何WS上锁
3. LS在table中记录f对应的owner为WS1，响应WS1授予锁(granting a lock)
4. WS1在本地table中修改f文件对应的状态为busy，写操作执行完后，将f文件的锁状态改成idle。这里lock有租约期限，WS1需要定期向LS续约。
5. WS2想问访问f，向LS发送请求锁(requesting a lock)
6. LS看table，发现f对应的owner为WS1（即使状态为idle，拥有者依然为WS1），于是向WS1发送撤销锁(revoking a lock)
7. WS1确定已经不对文件修改后，向Petal同步自己对f的修改数据
8. **WS1完成对f的修改同步后，向LS发送释放锁(releasing a lock)，同时删除自己本地对f锁的记录**
9. LS发现WS1释放锁后，重新修改table中f的owner为WS2
10. LS向WS发送授予锁(granting a lock)

先同步修改（写回cache）到petal，然后再释放锁。

# Frangipani---原子性（使用锁）

使用锁来实现**原子文件系统操作**。

​ 假设WS1在目录d下创建f目录，那么根据论文所描述，会获取目录d的锁，然后获取文件f的inode锁，大致伪代码如下：

```pseudocode
acquire("d") // 获取目录d的锁
create("f" ,....) // 创建文件f
acquire("f") // 获取文件f的inode锁
    allocate inode // 分配f文件对应的inode
    write inode // 写inode信息
    update directory ("f", ...) // 关联f对应的inode到d目录下
release("f") // 释放f对应的inode锁. busy -> idle
```
有获取多把锁的场景，因此可能会死锁。
Frangipani规则：锁进行排序，按序获取锁（可能按照inode）。

# Frangipani---崩溃恢复（WAL）

遵循预写式日志（write-ahead loggin）协议。
可以说petal就是为了WAL协议设计的。

这里有一件事情需要注意，**==Log只包含了对于元数据的修改==，比如说文件系统中的目录、inode、bitmap的分配。Log本身不会包含需要写入文件的数据，所以==日志并不包含用户更新的文件数据==，它只包含了故障之后可以用来恢复文件系统结构的必要信息**。
文件数据不会通过日志，而是直接给petal。
因为文件数据没有记录到log，所以**需要原子写文件**。

一开始Log存在于工作站内存中，向Petal中写Log的时机是revoke锁的时候。
当工作站从锁服务器收到了一个Revoke消息（此时会向petal同步修改），要自己释放某个锁，它需要执行好几个步骤。

1. 首先，工作站需要将内存中还没有写入到Petal的Log条目写入到Petal中。
2. 之后，再将被Revoke的Lock所保护的数据写入到Petal。
3. 最后，向锁服务器发送Release消息。

![](Pasted%20image%2020230716220854.png)
每台机器磁盘由log和fs两部分组成。

client的工作站更新时有一下操作：
1. 更新日志
2. 安装更新：log记录完成后，更新文件系统中的文件，进行读写操作。

log entry：序列号sn、更新数组（需要更新的块号inode、版本号、对应的更新）。
使用校验和来确保log的完整性。

因而log只是保证了崩溃时文件系统结构一致。

## 原子写文件（WAL中没有文件数据）

​ **对于需要原子写文件的场景，通常人们通过先将所有数据写入一个临时文件，然后做一个原子重命名（atomic rename），使得临时文件编程最终需要的文件**。同理，如果Frangipani需要原子写文件，也会采用这种方式。

## 崩溃场景分析

1. 写log前就崩溃了：数据直接丢失
2. log写到petal后崩溃
	- 每把锁都有租期（因为有可能是网络分区）。LS会等到租期过期后收回锁，recovery daemon会读取log然后应用log操作。因为其他WS可能会应用log操作，所以log最终要存在petal
3. 写log到petal的工程中崩溃
	- 校验和起作用。recovery daemon会停止在检验和不通过的记录之前。

# 日志和版本

​ 一个场景整体操作时序如下：

1. WS1在log中记录`delete ('d/f')`，删除d目录下的f文件
2. WS2在log中记录`create('d/f')`
3. WS1崩溃
4. WS3观察到WS1崩溃，为WS1启动一个recovery demon，执行WS1的log记录`delete('d/f')`

​ 原本WS2能在Petal中创建d目录下的f文件，但是WS3却有可能因为恢复执行WS1的log的缘故，将f文件删除。**这里Frangipani通过日志版本号避免了这个问题**。

**recovery daemon只重放高于version number的日志。**

# 总结

知识点：
- 缓存一致性协议(cache coherence)
- 分布式锁(distributed locking)
- 分布式恢复(distributed recovery)