一开始都是follower。
是一种leader节点的强一致性算法。

# leader选举

当长时间收不到leader的心跳包会electon timeout，然后超时的节点会term++, 成为candidate，先给自己投一票，并向其他节点发送request vote,的票超过半数成为learder，不断向其他节点发送心跳。

# failover

# figure 7

在c d宕机的情况下可以大多数当选leader
