bustub中加入事务支持。
# Task1 lock manager

类似于bufferpoll manager，整个系统有一个全局的LM。它维持一个关于活跃事务当前持有的锁的内部数据结构。
要实现表级和元组级LM并且支持三种隔离级别：`READ_UNCOMMITED`, `READ_COMMITTED`, and `REPEATABLE_READ`.
## 所需文件
### 修改

concurrency/lock_manager.cpp
include/concurrency/lock_manager.h
## 看源码

transaction.h and lock_manager.h

`TransactionManager`用来追踪系统中的所有事务。
### 上锁注意事项
表锁支持所有锁类型，行锁不支持意向锁。如果尝试在行上加意向锁，那么将此事务设为ABORT状态，并且抛出异常`TransactionAbortException(ATTEMPTED_INTENTION_LOCK_ON_ROW)`
#### **隔离级别和锁类型**

读未提交，不需要S/IS/SIX。任何尝试操作抛出异常`TransactionAbortException (LOCK_SHARED_ON_READ_UNCOMMITTED)`

- 可重复读。事务在GROWINT阶段可以上任意类型锁，SHRINKING阶段不可以加任何锁。
- 读已提交。事务在GROWINT阶段可以上任意类型锁，SHRINKING阶段可以加S, IS锁。
- 读未提交。事务只能在GROWING阶段可以上X, IX锁。不允许任何时候加S, IS, SIX锁。(不允许加S锁)。

==实现 `SERIALIZABLE` 需要在 `REPEATABLE_READS` 的基础上加上 index lock（或者谓词锁），解决幻读问题。==
#### 锁升级

只允许以下升级：
![](Pasted%20image%2020230528222908.png)
其余的事务必须abort并抛出异常`TransactionAbortException (INCOMPATIBLE_UPGRADE)`.

一个事务多个锁对同一资源进行升级，要将事务abort，并抛出异常。
### 解锁注意事项

必须要有锁才能解锁，否则抛异常。
解锁表，必须保证该事务在行上没有锁，否则抛异常。
解锁资源锁后，要进行授予锁。
#### 事务状态更新

**只有解锁S 或者 X 锁才更新事务状态。**

- 可重复读：解锁S/X，设txn状态为SHRINKING.
- 读已提交：解锁X，设为SHRINKING。解锁S无影响。
- 读未提交：解锁X，设为SHRINKING。不允许有S锁。

锁升级
锁请求队列
## 上锁步骤

一、检查事务状态

	若 txn 处于 Abort/Commit 状态，抛逻辑异常，不应该有这种情况出现。
	若 txn 处于 Shrinking 状态，则需要检查 txn 的隔离级别和当前锁请求类型。
	
二、获取table对应的**lock request queue**

	注意需要对 map 加锁，并且为了提高并发性，在获取到 queue 之后立即释放 map 的锁。

三、检查此锁请求是否为一次锁升级

	先对queue加锁。
	granted 和 waiting 的锁请求均放在同一个队列里，我们需要遍历队列查看有没有与当前事务 id（我习惯叫做 tid）相同的请求。如果存在这样的请求，则代表当前事务在此前已经得到了在此资源上的一把锁，接下来可能需要锁升级。需要注意的是，这个请求的 `granted_` 一定为 true。
	开始尝试锁升级。首先，判断此前授予锁类型是否与当前请求锁类型相同。若相同，则代表是一次重复的请求，直接返回。否则进行下一步检查。
	接下来，判断当前资源上是否有另一个事务正在尝试升级（`queue->upgrading_ == INVALID_TXN_ID`）。若有，则终止当前事务，抛出 `UPGRADE_CONFLICT` 异常。因为不允许多个事务在同一资源上同时尝试锁升级。
	然后，判断升级锁的类型和之前锁是否兼容，不能反向升级。(什么样的锁是兼容的？)

进行锁升级：
1. 可以升级吗？即我们此前的一系列判断。
2. 释放当前已经持有的锁，并在 queue 中标记我正在尝试升级。
3. 等待直到新锁被授予。

	在锁升级时，需要先释放此前持有的锁，把升级作为一个新的请求加入队列。
	假如遍历队列后发现不存在与当前 tid 相同的请求，就代表这是一次平凡的锁请求。

改进：怎么允许多个事务在同一资源上尝试锁升级？
改进：将granted和waiting队列分开。

四、创建锁升级请求，并将锁请求加入请求队列

	new 一个 LockRequest，加入队列尾部。改进：使用智能指针。
	请求队列中包括granted和waiting的锁请求，要插入到granted的后面，waiting的前面，因为锁升级请求最优先。

五、尝试获取锁

```cpp
std::unique_lock<std::mutex> lock(queue->latch_);
while (!GrantLock(...)) {
    queue->cv_.wait(lock);
}
```
在 `GrantLock()` 中，Lock Manager 会判断是否可以满足当前锁请求。若可以满足，则返回 true，事务成功获取锁，并退出循环。若不能满足，则返回 false，事务暂时无法获取锁，在 wait 处阻塞，等待资源状态变化时被唤醒并再次判断是否能够获取锁。资源状态变化指的是什么？其他事务释放了锁。
![](Pasted%20image%2020230529005858.png)

所有兼容的锁请求需要一起被授予。

两项检查通过后，代表当前请求既兼容又有最高优先级，因此可以授予锁。授予锁的方式是将 `granted_` 置为 true。并返回 true。假如这是一次升级请求，则代表升级完成，还要记得将 `upgrading_` 置为 `INVALID_TXN_ID`。

Bookkeeping 操作。Transaction 中需要维护许多集合，分别记录了 Transaction 当前持有的各种类型的锁。方便在事务提交或终止后全部释放。

## 解锁步骤

以table lock为例，需要检查其下的所有row lock是否已经释放。

一、获取对应的lock request queue

二、遍历该表的lock request queue，找到该事务unlock对应的granted请求。

三、根据隔离级别更新事务状态

	若不存在，抛异常。
	
	找到对应锁请求后，根据事务的隔离级别和锁类型修改其状态。
	当隔离级别为 `REPEATABLE_READ` 时，S/X 锁释放会使事务进入 Shrinking 状态。
	当为 `READ_COMMITTED` 时，只有 X 锁释放使事务进入 Shrinking 状态。
	当为 `READ_UNCOMMITTED` 时，X 锁释放使事务 Shrinking，S 锁不会出现。
	
	之后，在请求队列中 remove unlock 对应的请求，并将请求 delete。
	
	同样，需要进行 Bookkeeping。
	在锁成功释放后，调用 `cv_.notify_all()` 唤醒所有阻塞在此 table 上的事务，检查能够获取锁。

# Task2 Deadlock Detection

LM使用一个后台线程周期动态的构建一个等待图，并打破循环。
```cpp
  LockManager() {
    enable_cycle_detection_ = true;
    cycle_detection_thread_ = new std::thread(&LockManager::RunCycleDetection,                                                   this);
  }
```

后台线程应该在每次被唤醒时动态地构建图。你不应该维护一个图，它应该在每次线程醒来时被构建和销毁。
有向图环检测算法：DFS和拓扑排序，我们使用DFS。
始终从tid小的结点开始搜索，选择邻居时优先搜索tid小的邻居。
每次调用DFS检测算法结果必须是确定的，结果必须是最大的事务id，将其设为ABORTED.

## 构建wait-for图步骤

构建 wait for 图的过程是：
遍历 `table_lock_map` 和 `row_lock_map` 中所有的请求队列，对于每一个请求队列，用一个二重循环将所有满足等待关系的一对 tid 加入 wait for 图的边集。
满足等待关系是指，例如二元组 (a, b)，a 是 waiting 请求，b 是 granted 请求，并且 a 与 b 的锁不兼容，则生成 `a->b` 一条边。

## 破环

1. 选择成环节点中tid最大的事务，状态设为aborted，并在请求队列中移除此事务，释放锁，终止阻塞请求，并使用条件变量通知阻塞事务。
2. 移除wait-for图中该事务的边。（可能有多个环）（这可以当作一个难点）

# Task 3 Concurrent Query Execution

忽略并发索引执行。
当上锁解锁失败时，事务应该终止。事务终止后，要undo对表元组和索引的写操作。因此要维持事务的写集(这是事务管理器的Abort()方法所要求的)。
当执行其无法获得锁时，抛出`ExecutionException`异常。
一个元组可能在一个事务中被不同的查询访问不止一次，思考在不同隔离级别下的解决方法。

实现三个算子：
- seq_scan
- insert
- delete
为什么只实现了这三个算子的并发查询？因为其他算子获取的 tuple 数据均为中间结果，并不是表中实际的数据。

## SeqScan

### 隔离类型与锁

`READ_UNCOMMITTED` 则无需加锁。加锁失败则抛出 `ExecutionException` 异常。
在 `READ_COMMITTED` 下，在 `Next()` 函数中，若表中已经没有数据，则提前释放之前持有的锁。在 `REPEATABLE_READ` 下，在 Commit/Abort 时统一释放，无需手动释放。

### 锁类型

表加IS锁，行加S锁。 

# Leadboard Task

## 谓词下推到seqscan

> 阅读源码

bustub已经给出了`OptimizeMergeFilterScan`函数的实现。
1. 获取该plannode的孩子节点，递归优化所有孩子计划节点
2. 将优化后的孩子计划节点加入创建的新节点的孩子中，`CloneWithChildren`
3. 如果本节点是`Filter`类型的计划节点，那么将此过滤下推，即谓词下推至`SeqScan`计划节点
4. 将两个计划节点合成一个具有谓词的seqscan节点。

>在优化器中加入`OptimizeMergeFilterScan`函数调用

>在seqscan算子中做相应的优化

## 实现update算子

这样可以原地修改tuple，而不是先delete在insert。
锁：
表上IX锁，行上X锁。
返回值，返回更新的行数。放到tuple的value数组中传出。

## use index

将谓词下推到indexscan算子。
需要更新索引扫描计划，修改`terrier_benchmark_config.h`

将原本的filter->seq scan  plan node, 并且filter的谓词是相等比较，并且此谓词刚好是该表上某个索引的键。
那么优化成indexscan plan node  with predicate
# 亮点

使用了`std::adopt_lock`
## 条件变量

条件变量与互斥锁配合使用。首先需要持有锁，并查看是否能够获取资源。这个锁与资源绑定，是用来保护资源的锁。若暂时无法获取资源，则调用条件变量的 wait 函数。调用 wait 函数后，latch 将自动**释放**，并且当前线程被挂起，以节省资源。这就是阻塞的过程。此外，允许有多个线程在 wait 同一个 latch。
当其他线程的活动使得资源状态发生改变时，需要调用条件遍历的 `notify_all()` 函数。
`notify_all()` 可以看作一次广播，会唤醒所有正在此条件变量上阻塞的线程。在线程被唤醒后，其仍处于 wait 函数中。在 wait 函数中尝试获取 latch。在成功获取 latch 后，退出 wait 函数，进入循环的判断条件，检查是否能获取资源。若仍不能获取资源，就继续进入 wait 阻塞，释放锁，挂起线程。若能获取资源，则退出循环。这样就实现了阻塞等待资源的模型。条件变量中的条件指的就是满足某个条件，在这里即能够获取资源。

# 难点

如何使用锁来实现不同隔离级别？
![](Pasted%20image%2020230530204131.png)
