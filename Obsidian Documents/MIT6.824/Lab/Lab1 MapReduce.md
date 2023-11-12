# 介绍

构建mapreduce系统。
worker进程：调用Map和Reduce函数读写文件。
coordinator进程：将任务分发给workers，处理故障的workers。
根据MapReduce论文进行构建。

每个reduce输出一个mr-out-X文件

# 任务

实现分布式mapreduce。由worker和coordinator两个程序组成。
main中的例程会调用我写的mr中的程序。

## 编写文件

`mr/`下的`master, worker`，`rpc`。

## 细则

只有一个coordinator，一个或者多个worker并行运行在同台机器上。
workers和coordinator通过RPC通信。
worker向coordinator请求任务，然后根据任务读取一个或多个输入文件，然后将输出写到一个或多个文件。
coordinator要通知worker如果在一段时间内没有完成任务（本实验为10s），重新将此任务分配给其他worker

## 例程

`main/mrcoordinator.go` and `main/mrworker.go`
不要更改例程
将我的实现放在： `mr/coordinator.go`, `mr/worker.go`, and `mr/rpc.go`.

# 规则

- map阶段将中间key分到`nReduce` reduce任务的桶中。`nReduce`是reduce任务的数量，`main/mrcoordinator.go`传递给`MakeCoordinator()`的参数。每个mapper要创建`nReduce`个中间文件给reduce task。
- worker将第X个reduce task的输出放到文件`mr-out-X`中
- `mr-out-X`要一行一行的生成kv形式。格式参考`main/mrsequential.go`文件
- worker将中间map输出放到当前目录下文件中，稍后worker会读取作为redece task的输入。
- `mr/coordinator.go`中的Done（）函数返回true来告诉`main/mrcoordinator.go` mapreduce任务已经完成，然后worker和coordinator都退出

# 测试

## 测试文件

·`main/test-mr.sh.`
检查wc和indexer的输出是否正确、并行性、还有worker的故障恢复


## 运行程序


```go
// 构建wc应用插件
go build -buildmode=plugin ../mrapps/wc.go
// In the main directory, run the coordinator.
rm mr-out*
go run mrcoordinator.go pg-*.txt
// In one or more other windows, run some workers:
go run mrworker.go wc.so
```



同步问题最复杂：
使用channel来进行同步
使用mutex和cv
![](Pasted%20image%2020230621115244.png)


如何处理超时任务

两个任务通道

我的RPC服务器是coordinator？

可以使用检查点来容错，coordinator没那么多状态需要保存。也可以用raft有点大材小用。

Goole cloud dataflow   /  Spark 



# 瓶颈

太多问题给了coordinator来做
