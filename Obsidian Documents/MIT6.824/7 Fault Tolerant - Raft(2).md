# leader选举

继续对论文图7进行讨论。
**在发现有follower log中的term比candidate log中的term高，candidate直接作废成为follower。**
acd有可能成为leader
# log catch up（unoptimized）

lab2 b要解决的问题。

心跳包发送内容：本次的log内容及term，最后一个log的term号及内容，最后一个log索引（prev index）

nextindex：乐观的，假设follower的下一个log索引和leader的下一个log索引一样。leader当前的log index + 1
matchindex：悲观的。记录leader和其他follower从0开始往后最长能匹配上的log index的位置+1，表示leader和某个follower在matchIndex之前的所有log entry都是对齐的。这里说悲观，因为leader当选时，leader会初始化matchIndex值为0，表示认为自身log中没有一条记录和其他follower能匹配上。随着leader和其他follower同步消息时，matchIndex会慢慢增加。**leader为每个自己的follower维护matchIndex，因为平时根据majority规则，需要保证log已经同步到足够多的followers上**。可以交给上层应用的。

未优化版本会一条一条进行日志追赶。需要多次才能完成日志对齐。

# 日志擦除

在领导者在它的term提交一个条目后才能commit
# 日志追赶（优化）

 lab 2c中 必须实现这种优化。论文没有详细描述。
 基本思路：拒绝中包含更多信息 来帮助leader更快的后退。
 ![](Pasted%20image%2020230622160820.png)
 减少了心跳次数，可以快速追赶。
 可以回退一次一个term。

# 持久化

## 新节点加入集群 / 节点重启策略

有两种策略。

投票选择；log；current term；
vote已经讨论
## why log？

向leader承诺提交。防止提交的log又进行一遍
## why current term？

选举时用到，要保证term增加。
## 持久化时机

以上变量变化的时候就进行持久化。先持久化再进行回应
# 服务恢复

有两种策略。类似上面。
- replay log：比较昂贵。一般不考虑。
- periodic snapshot：也能够压缩日志。从某个时刻开始重建状态。ST + RSM ?

lab 2d 快照和日志压缩。需要服务和raft进行交互，确定在哪进行了快照。

# 使用raft


# 线性一致性

==线性一致性定义！！！==

​ 在论文中对整个系统提供的服务的正确性称为**线性一致性(Linearizability)**，线性一致性需要保证满足一下三个条件：

1. **整体操作顺序一致(total order of operations)**
    
    即使操作实际上并发进行，你仍然可以按照整体顺序对它们进行排序。（即后续可以根据读写操作的返回值，对所有读写操作整理出一个符合逻辑的整体执行顺序）
    
2. **实时匹配(match real-time)**
    
    顺序和真实时间匹配，如果第一个操作在第二个操作开始前就完成，那么在整体顺序中，第一个操作必须排在第二个操作之前(换言之如果这么排序后，整体的执行结果不符合逻辑，那么就不符合"实时匹配")。
    
3. **读操作总是返回最后一次写操作的结果(read return results of last write)**

数据库世界中有类似的术语，叫做**可串行化(serializability)**。基本上线性一致性和可串行化的唯一区别是，**可串行化不需要实时匹配(match real-time)**。

# 只读操作优化

## readindex read

## lease read