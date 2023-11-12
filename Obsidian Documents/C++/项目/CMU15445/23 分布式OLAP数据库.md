![](Pasted%20image%2020230602140249.png)
决策支撑系统
![](Pasted%20image%2020230602140323.png)

# AP数据架构

![](Pasted%20image%2020230602140437.png)
一个中心多个爪，两级结构。可能会有数据冗余，不统一。

雪花型架构
**![](Pasted%20image%2020230602142221.png)

![](Pasted%20image%2020230602142320.png)

问题：
![](Pasted%20image%2020230602142859.png)
分布式数据库如何join，大数据单机数据库放不下，必须分区。

# 本节课内容

![](Pasted%20image%2020230602142939.png)

# 分布式执行模型

![](Pasted%20image%2020230602143027.png)

## push query to data

多用于shared nothing 架构。
![](Pasted%20image%2020230602143353.png)

## pull data to query

多用于shared disk架构
![](Pasted%20image%2020230602143501.png)

>思考

![](Pasted%20image%2020230602143538.png)
长OLAP事务崩溃怎么办 ？

![](Pasted%20image%2020230602143656.png)
假设节点查询执行期间不会崩溃。中间结果可以放入disk种，崩溃重启后进行恢复。

# 分布式查询计划

更加复杂和重要。
![](Pasted%20image%2020230602143850.png)
- 谓词下推
- 早物化
- join顺序
依旧可用。

要多考虑数据分区位置和网络传输花销。

## 查询计划碎片

切分计划的方案：
![](Pasted%20image%2020230602144013.png)
- 分区执行计划，分给对应分区
- 切分SQL语句

![](Pasted%20image%2020230602144146.png)

>思考

![](Pasted%20image%2020230602144247.png)
分布式join依赖于数据分区策略。

# 分布式join算法

![](Pasted%20image%2020230602144726.png)
将可以连接的部分，放到同一个节点

## 场景

![](Pasted%20image%2020230602144836.png)
如果一个表经常join，可以将这个表在所有节点上都有副本。

![](Pasted%20image%2020230602144951.png)
都按连接键分区。
![](Pasted%20image%2020230602145050.png)


![](Pasted%20image%2020230602145127.png)
![](Pasted%20image%2020230602145226.png)
使用数据广播。

分区和连接键没关系。进行数据重分布。
![](Pasted%20image%2020230602145400.png)


## SIMI-JOIN

![](Pasted%20image%2020230602150434.png)

![](Pasted%20image%2020230602150604.png)

# 云系统

![](Pasted%20image%2020230602150700.png)

![](Pasted%20image%2020230602151211.png)
第二种：云原生DBMS。设计时就知道要运行在云上。首选shared-disk架构。

![](Pasted%20image%2020230602152000.png)
不需要服务器数据库，更云的架构。
![](Pasted%20image%2020230602152116.png)

