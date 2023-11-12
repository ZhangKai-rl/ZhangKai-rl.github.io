-race参数会检测代码程序中的竞态。是一种动态分析。
sync.WaitGroup：类似muduo中的countdownLatch

# RPC

远程过程调用（Remote Procedure Call，RPC）是一个**计算机通信协议**。该协议允许客户端通过 RPC 执行服务端的函数，就像调用本地函数一样。
RPC可以基于TCP协议也可以基于HTTP协议。

![](Pasted%20image%2020230716001528.png)

一个 RPC 流程一般如下：

1. 客户端调用 client stub，并将调用参数 push 到栈（stack）中，这个调用是在本地的
2. client stub 将这些参数包装，并通过系统调用发送到服务端机器。打包的过程叫 `marshalling`（常见方式：XML、JSON、二进制编码）。
3. 客户端操作系统将消息传给传输层，传输层发送信息至服务端；
4. 服务端的传输层将消息传递给 server stub
5. server stub 解析信息。该过程叫 `unmarshalling`。
6. server stub 调用程序，并通过类似的方式返回给客户端。
7. 客户端拿到数据解析后，将执行结果返回给调用者。