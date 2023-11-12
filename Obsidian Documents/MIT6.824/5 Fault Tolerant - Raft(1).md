
raft是用于复制的，将复制分为leader和follower（选举，配置服务）。
保证leader和follower的一致性，并在leader出现故障时重新选举合适leader。
两个大作用：复制并保证一致性，leader选举。

# 特性

raft基于大多数原则。如果要容忍f个节点失败，至少要有2f + 1个节点，永远是奇数。
可以解决单点故障、网络分区问题。
**强一致性。** 读leader写leader。客户端只会请求leaer。
raft投票包括本身，投票时永远给自己投赞成
在raft中，每个节点被组织成一个复制状态机（RSM使用日志log，另一种是ST）。
Replicated And Fault Tolerant，`复制和容错`-------RAFT，是一种日志的一致性算法
节点通信：使用 `RPC`
- 请求投票（RequestVote） RPCs：选举阶段，Candidate 节点发送给他人
- 附加条目（AppendEntries）RPCs：非选举阶段，Leader 发给所有节点，复制日志+心跳

==特性（Raft 保证在任何时候都成立）==
- Leader 只附加原则：Leader 不会删除、覆盖自己的日志，只会增加
- 日志匹配：若两个日志在相同索引位置的日志的 term 号相同，则日志从头到该索引位置全部相同
- Leader 完整特性：选举出的 Leader，会包含所有已提交的日志
- 状态机安全特性：Leader 已经将给定的索引值位置的日志条目应用到状态机，其他任何服务器都已执行

读写操作都是与leader交互。
# RSM with Raft

lab3中会使用raft构建rsm。是一个go包

WAL先在raft上写日志，然后再在上层服务器写入。
日志有索引
客户端与leader交互

# raft log的作用

最后所有server的log会是相同的，包括顺序

## 重传

leader向follower同步消息时，消息可能传递失败，所以需要log记录，方便重传
## 顺序

**主要原因，我们需要被同步的操作，以相同的顺序出现在所有的replica上**
## 持久化

持久化log数据，才能支持失败重传，重启后日志恢复等机制
## 试探性操作

# log entry 格式

- command
- term：leader s term

日志条目有log index 和 leader term唯一标识
# leader选举

lab2a

leader定期向followers发送心跳，如果超时follows未收到心跳（election timeout）就会成为candidate，term+1，并开始leader选举。

通过term编号和多数原则避免了脑裂。

## 分裂选举问题-split vote

一个节点每个term只有一票。
每个节点的超时时间是随机的，election timeout is randomized: 150 ~ 300 ms.
同时出现两个candidate。会继续发送直至其中一个再次election timeout，最终赞成term大的candidate。

## 选举超时时间设置

选举时客户端是阻塞的。
- 略大于心跳包间隔
- 加入一些随机数，来减少split vote
- 尽量短，减少阻塞

## vote需记录到稳定storage

避免重复vote

**为了保证每个term，每个机器只会进行一次vote行为，以保证最后只会产生一个leader，每个参选者都需要用稳定的storage记录自己的vote行为**。


# 日志分歧

# 资料

- 生动形象的网站：[http://thesecretlivesofdata.com/raft/](https://link.zhihu.com/?target=http%3A//thesecretlivesofdata.com/raft/)
- 论文：[https://raft.github.io/raft.pdf](https://link.zhihu.com/?target=https%3A//raft.github.io/raft.pdf)

# 应用

etcd

# 读写操作

无论读写都是client请求leader。因而leader会是一个瓶颈

## 读操作

leader需要将log entry同步到其他节点。

### 读操作优化