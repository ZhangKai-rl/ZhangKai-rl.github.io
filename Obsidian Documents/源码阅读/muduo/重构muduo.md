
来自腾讯课堂-施磊

--------



one loop per thread：
- 一个eventloop对应一个thread
- TcpConnection对象/channel/fd，的一切函数执行都要在其所属的SubEventLoop线程中运行。
- 和这个EventLoop有关的，被这个EventLoop管辖的一切操作都必须在这个EventLoop绑定线程中执行。如何实现的？

# buffer

一堆如prependInt8，readInt8之类的函数缩减。
buffer的设计思想非常优秀，尤其是扩容那里。
分散读集中写
Buffer::readFd 中的extrabuf。
用户和内核（socket）交互的中间件

## shrink

使用c++11，vector::shrink_to_fit()来代替开新vector并swap操作

# tcpconnection

## TcpConnection::send

![](Pasted%20image%2020230609022843.png)




![](Pasted%20image%2020230609024334.png)
