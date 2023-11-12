##### 原子操作

##### one loop per thread怎么实现的?怎么确保一个thread绑定一个eventloop

在上层应用中会使用`TcpServer::start()`和`EventLoop.loop()`(事件循环)。
在`TcpServer::start()`调用了`EventLoopThreadPool::start`，在事件循环线程池中创建了多个`EventLoopThread`。
EventLoopThread有成员变量`Thread thread_;`,在该类的构造函数中进行创建`thread_(std::bind(&EventLoopThread::threadFunc, this), name)`。将线程函数传入，`EventLoopThread::threadFunc`中创建了subloop并初始化了`EventLoopThread::loop_`。
以上只是构造了含有资源的类。`Thread::start()`中真正使用了线程信息，调用`pthread_create`进行了线程的创建。

在`Thread::start()`会创建线程执行`EventLoopThread::threadFunc`，其中调用语句`Eventloop loop;`，其会有`__thread EventLoop* t_loopInThisThread = 0;`全局变量，会在构造函数中进行检查，如果其值不为0，那么已经有线程绑定了此Eventloop。**此变量为实现 one loop per thread** 的关键。
##### 时间堆

断开超时连接，可以使用：双向链表，时间轮（muduo使用），时间堆。前两者使用tick脉搏函数。

##### 四缓冲区异步日志

使用枚举结构体定义日志级别。日志后端使用单例模式，运行在一个独立的日志线程。
使用异步日志，开启一个日志后端独立线程进行日志文件的写入，防止写入日志时阻塞服务器。
- tinywebserver前端和后端使用阻塞队列进行交互，前端向阻塞队列中装填日志，后端取出日志写入文件。使用环形数组实现阻塞队列。

##### epoll+LT 主从reactor

##### 内存池

![](Pasted%20image%2020230807091115.png)

##### 线程池

手写一个线程池
```cpp
#include <memory>
#include <functional>
#include <thread>
#include <iostream>
#include <vector>

class Thread
{
public:
    using ThreadFunction = std::function<void()>;
    Thread(ThreadFunction func,int id):func_(func),thread_id_(id){}
    void Start()
    {   
        std::cout<<"thread "<<thread_id_<<" start working"<<std::endl;
        thread_ = std::make_shared<std::thread>(func_);
    }
    void Stop()
    {
        if(thread_->joinable())
        {
            thread_->join();
        }
    }
    ~Thread(){}
private:
    std::shared_ptr<std::thread> thread_;
    int thread_id_;
    ThreadFunction func_;
};


class ThreadPool
{
public:
    ThreadPool():threads_nums_(0){};
    ~ThreadPool()
    {
        for(int i = 0;i < threads_nums_;++i)
        {
            thread_pool_[i]->Stop();
            delete thread_pool_[i];
        }
    }

    void Start(int threads_num)
    {
        threads_nums_ = threads_num;
        for(int i = 0;i<threads_num;++i)
        {
            thread_pool_.push_back(new                  Thread(std::bind(&ThreadPool::RunInThread,this,i),i));
        }

        for(int i = 0;i<threads_num;++i)
        {
            thread_pool_[i]->Start();
        }
    }
private:
    std::vector<Thread*> thread_pool_;
    int threads_nums_;
    void RunInThread(int id)
    {
        std::cout<<"RunInThread id = "<<id<<std::endl;
    }
};
int main()
{
    ThreadPool pool;
    pool.Start(8);

    std::this_thread::sleep_for(std::chrono::microseconds(1000));
    return 0;
}
```

##### 怎么实现的高性能，体现在哪里？

- 使用主从reactor + 非阻塞IO：避免阻塞在read write等io系统调用上，IO线程只阻塞在IO多路复用函数上
- 使用线程池、连接池和内存池
- 使用one loop per thread思想 + 线程池 + 任务队列：使用reactor来处理IO，使用线程池来处理计算
- 使用双缓冲异步日志：准备两块buffer，前端负责向buffer a填日志消息，后端负责将buffer b的数据写入文件，进行磁盘IO。当a写满之后交换a、b。这样在前端写日志时不必等待磁盘io操作，也避免了每条消息都唤醒日志后端。
- 使用应用层缓冲区：
	- 输出缓冲：使用write发送100kb数据，但可能tcp缓冲只有80kb空间，如果没有应用层缓冲那么就只能阻塞在这，因此网络库应该接管这无法发送的20kb，注册POLLOUT事件，发送完成后停止关注，避免busy loop
	- 输入缓冲：TCP是一个无边界的字节流协议，接收方必须要处理“收到的数据尚不构成一条完整的消息”“一次收到两条消息的数据”等情况。  用于解决粘包
- 大文件传输使用mmap + sendfile
- 使用**时间堆**管理连接，断开超时连接
##### 为什么使用LT

- 与poll兼容
- LT模式不会发生漏掉事件的BUG，但POLLOUT事件不能一开始就关注，否则会出现busy loop，而应该在write无法完全写入内核缓冲区的时候才关注，将未写入内核缓冲区的数据添加到应用层output buffer，直到应用层output buffer写完，停止关注POLLOUT事件。-- 相当于写完成回调，处理写完成事件。
- 读写的时候不必等候EAGAIN，可以节省系统调用次数，降低延迟。（注：如果用ET模式，读的时候读到EAGAIN，写的时候直到output buffer写完成或者EAGAIN）
##### 多线程操作

- mutex、semaphore、condition_variable
- __ thread、threadlocal
##### HTTP解析的状态机

![](Pasted%20image%2020230807130959.png)
##### ET和LT  EPOLLONESHOT

- 使用边缘触发模式时，当被监控的 Socket 描述符上有可读事件发生时，**服务器端只会从 epoll_wait 中苏醒一次**，即使进程没有调用 read 函数从内核读取数据，也依然只苏醒一次，因此我们程序要保证一次性将内核缓冲区的数据读取完；**边缘触发模式一般和非阻塞 I/O 搭配使用**，程序会一直执行 I/O 操作，直到系统调用（如 `read` 和 `write`）返回错误，错误类型为 `EAGAIN` 或 `EWOULDBLOCK`。
- 使用水平触发模式时，当被监控的 Socket 上有可读事件发生时，**服务器端不断地从 epoll_wait 中苏醒，直到内核缓冲区数据被 read 函数读完才结束**，目的是告诉我们有数据需要读取；
ET两个请求如果离得太近，可能会丢失。

EPOLLONESHOT：对于注册了EPOLLONESHOT事件的[文件描述符](https://so.csdn.net/so/search?q=%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6&spm=1001.2101.3001.7020)，**操作系统最多触发其上注册的一个可读、可写或者异常事件，且只触发一次**。当一个线程在处理某个socket时候，其他线程是不可能有机会操作该socket的。反过来，注册了EPOLLONESHOT事件的socket一旦被某个线程处理完毕，该线程就应该立即重置这个socket上的EPOLLONESHOT事件，以确保这个socket下一次可读时，其EPOLLIN事件能被触发，进而让其他工作线程有机会继续处理这个socket。
##### reactor和proactor

Reactor 是非阻塞同步网络模式，而 **Proactor 是异步网络模式**。
- **Reactor 是非阻塞同步网络模式，感知的是就绪可读写事件**。在每次感知到有事件发生（比如可读就绪事件）后，就需要应用进程主动调用 read 方法来完成数据的读取，也就是要应用进程主动将 socket 接收缓存中的数据读到应用进程内存中，这个过程是同步的，读取完数据后应用进程才能处理数据。
- **Proactor 是异步网络模式， 感知的是已完成的读写事件**。在发起异步读写请求时，需要传入数据缓冲区的地址（用来存放结果数据）等信息，这样系统内核才可以自动帮我们把数据的读写工作完成，这里的读写工作全程由操作系统来做，并不需要像 Reactor 那样还需要应用进程主动发起 read/write 来读写数据，操作系统完成读写工作后，就会通知应用进程直接处理数据。
**Reactor 可以理解为「来了事件操作系统通知应用进程，让应用进程来处理」**，而 **Proactor 可以理解为「来了事件操作系统来处理，处理完再通知应用进程」**。这里的「事件」就是有新连接、有数据可读、有数据可写的这些 I/O 事件这里的「处理」包含从驱动读取到内核以及从内核读取到用户空间。
无论是 Reactor，还是 Proactor，都是一种基于「事件分发」的网络编程模式，区别在于 **Reactor 模式是基于「待完成」的 I/O 事件，而 Proactor 模式则是基于「已完成」的 I/O 事件**。

##### 怎么优雅断开连接（发送完关闭实现）

[优雅的关闭连接 --- 使用shutdown和setsockopt(SO_LINGER)实现_iocp 优雅的关闭套接字_六一要努力哦的博客-CSDN博客](https://blog.csdn.net/qq_36316285/article/details/115287746)

**总结：**
	**通过setsockopt可以设置SO_LINGER,从而实现优雅的关闭连接**

**定义：**
**优雅关闭**：如果发送缓存中还有数据未发出则其发出去，并且收到所有数据的ACK之后，发送FIN包，开始关闭过程。TCP连接线关闭一个方向，此时另外一个方向还是可以正常进行数据传输。
**强制关闭**：如果缓存中还有数据，则这些数据都将被丢弃，然后发送RST包，直接重置TCP连接。两边都关闭了，服务端处理完的信息没有正常传给客户端。

**close函数和shutdown函数对比：**
- 原型：`int close(int fd);`    `int shutdown(int sock, int howto); `
- close会关闭连接了，并释放所有连接对应的资源，而shutdown并不会释放掉套接字和所有资源。
- close有引用计数，例如父子进程都打开了某个文件描述符，其中某个进程调用了close函数，会使close函数的引用计数减1，直到套接字的引用计数为0，才会真正的关闭连接。而shutdown函数可以无视引用计数，直接关闭连接。
- close的引用计数的存在导致不一定会发出FIN结束报文，而shutdown一定会发出FIN报文。
- shutdown() 用来关闭连接，而不是套接字，不管调用多少次 shutdown()，套接字依然存在，直到调用 close() / closesocket() 将套接字从内存清除。
- 调用 close()关闭套接字时，或调用 shutdown() 关闭输出流时，都会向对方发送 FIN 包。FIN 包表示数据传输完毕，计算机收到 FIN 包就知道不会再有数据传送过来。
- 默认情况下，close()引用计数为0后会立即往网络中发送FIN包，不管输出缓冲区中是否还有数据，而shutdown() 会等输出缓冲区中的数据传输完毕再发送FIN包。也就意味着，调用 close()将丢失输出缓冲区中的数据，而调用 shutdown() 不会。

**使用setsockopt设置SO_LINGER实现优雅的关闭连接** 
LINGER结构包含了指定套接字的信息，当调用了close()函数关闭该套接字时，LINGER结构中的信息指定了如何处理未发送的数据。

##### C++11新特性的使用

channel使用了`weak_ptr<void> tie_;`。
作用：[Muduo分析及总结（二）Channel_奔跑的哇牛的博客-CSDN博客](https://blog.csdn.net/H514434485/article/details/90339554)****
![](Pasted%20image%2020230809215825.png)
# 一些其它类问题
##### muduo相对于别的网络库来说，他的优势和特点是什么呢

来自陈硕本人的回答：[[转] 【开源访谈】Muduo 作者陈硕访谈实录 - 枪侠 - 博客园 (cnblogs.com)](https://www.cnblogs.com/qiangxia/p/4892562.html)

------

先说特点。Muduo有一个很明显的特点是不可移植的。一般的网络库会把跨平台可移植当做一个卖点。而我特意选择只在 Linux 上实现并优化，这个算是特点。因为大规模分布式系统通常都会在Linux上开发部署，支持其他平台没有多大意义，我个人精力与知识面也不够。还有一个特点 是只支持 TCP。有的网络库会以支持TCP、UDP、ICMP、串口等各种协议为卖点，muduo则不然。Muduo的特点是只支持 TCP ，而且只支持 IPv4。因为我不认为开发公司内部使用的分布式系统会用到其他传输协议，更不会在内网用IPv6。muduo只支持one event loop per thread这一种并发模型，只使用非阻塞IO，因为这是Linux下使用native语言编写高性能网络程序最成熟的模式，Muduo适合编写有较多并 发TCP长连接的网络服务。甚至连 DNS 解析都只支持异步的解析，没有直接的一个函数调用就能从域名拿到 IP，因为这样会阻塞。总之，我认为在开发公司内部系统中用不到的东西我都没有支持，muduo是有明确的适用范围的，它不是那种大而全的网络库。减少选 择，让你节省时间，少走弯路。  
再说优势。优势之一是API设计。Muduo是一个现代的 C++ 网络库。现代和古代的API区别在于两方面。一个是事件回调，另外一个是资源管理。一般的网络库设计API的方式是定义一个接口（抽象基类），包含几种网 络事件对应的处理函数。你的代码去继承这个接口，这个接口会定义收到消息是回调哪个虚函数，然后你覆盖一下这个虚函数。然后把你的对象注册到网络库中，发 生事件的时候就回调你的虚函数。一般的 Framework 都这么搞，这就是传统的或者说古代的 C++ 网络库的做法，也是Java网络库的做法。这种做法在C++中面临的一个直接问题是对象的生命期管理，因为C++的动态绑定只能通过指针和引用来实现，你 必须把基类指针传给framework，才能获得事件回调。那么这个派生类对象何时销毁就成了难点，它的所有权到底归谁？有的网络库甚至在事件处理函数中 出现了delete this;这种代码，让人捏一把汗。  
我现在的回调方式是用boost::function，它在TR1时已经进入 C++ 标准库。Boost::function不对类型和函数名做限制，只对参数和返回类型做部分限制。如果你通过传统的继承来回调的话，你这个类型必须是 framework里某个基类的派生类，函数的名字必须一样，参数列表必须一样，返回类型也基本肯定是一样。但是boost::function没有这些 限制。Muduo网络库不是一个面向对象(object-oriented)的库，它是一个基于对象(object-based)的库。它在接口上没有表 现出继承的特性，它用的是boost function的注册/回调机制，网络事件的表示就用 boost function。所以对Muduo来讲，它不需要知道你写什么类，也不强迫继承，更不需要知道你的函数叫什么名字，你给它的就是一个 boost function对象，限制就很少。而且你没有把对象指针传给网络库，那么就可以按原有的方式管理对象的生命期。  
还有一个优势就是资源管理，Muduo在一处最关键的地方用了引用计数（Reference Counting）型智能指针，当然我没有自己写，用的是标准库的shared_ptr。我只在表示 TCP 连接的class上使用了引用计数，是因为TCP连接是短命对象(short-lived)。但是当连接被动断开的时候，网络库不能立刻销毁对象，因为用 户可能还持有它的引用，准备用来发消息。如果直接delete，有可能造成空悬指针。因此既然TCP对象是网络库和用户代码共同拥有，那就用引用计数好 了。Muduo用引用计数是经过仔细考虑的，也没有用在其他长命的对象上，这些长命对象的生命期可以由用户代码直接管理。用Muduo你就不用担心指针失 效的问题，可以避免一些古老的 C++ 程序中的一些内存错误。  
这种用对象来封装文件描述符等系统资源的做法是C++独有的资源管理方式，称为RAII。通过把文件描述符的生命期与对象等同起来，我们还能有效地避免串 话(cross talk)。比如说，操作系统给你一个新的TCP连接，文件描述符就是一个小整数，这个整数可能等于刚刚关闭的某个TCP连接的文件描述符。比如你现在有 一个连接号是3，你把连接关了再打开有可能还是3，所以就带来连接管理方面的一些麻烦。如果你是用 C 写，不小心的话就会造成你这里关了3这个连接，但是程序其他地方还在往3这个连接发消息（考虑多线程的话更头疼），但其实3这个连接已经指向其他地方了， 就跟使用野指针一样。用RAII就没有这个困扰，因为3这个连接的生命期和对象绑定，对象活着，连接就不会关闭，也就不会有其他对象同时使用了3这个文件 描述符。  
最后，Muduo的性能也是让人满意的。我在编写Muduo的时候没有以“高性能”为首要目标。在完成并开源之后，受网友启发，拿它和其他一些网络库做了 性能对比，发现相比通用的跨平台网络库（libevent2、Boost.Asio），muduo有明显的性能优势。相比专用的网络程序 （Nginx，ZeroMQ），muduo的性能也不落下风。