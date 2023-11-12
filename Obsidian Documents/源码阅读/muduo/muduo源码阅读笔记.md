大部分都写在了wsl的muduo库里面和陈硕书里面

----

分为base和net部分。

# 概览

采用shared_ptr管理对象生命周期。
网络库，最终会编译成动态/静态库
主从reactor
![](Pasted%20image%2020230719155106.png)

![](Pasted%20image%2020230708175441.png)
**decode compute encode是我们用户在OnMessage处理的业务逻辑。**
每一个线程都是一个独立的Reactor。
![](Pasted%20image%2020230710155743.png)
reactor对应eventloop；demultiplex对应poller

# 核心代码

## base

### noncopyable

delete拷贝和赋值操作，使用默认构造和析构函数

#### 子类

TcpServer
基类的拷贝和赋值被delete，因而这些派生类可以正常构造析构，但是都不可赋值拷贝

### TimeStamp

获取时间戳的类
比较重要的是静态now和tostring函数
成员变量为int64 ；

#### 改写

使用
tm* tm_time = localtime(&time(NULL));
totring使用snprintf
### Log日志

**多线程异步高性能日志。**
#### 日志库模型

日志格式：
日志文件命名格式：

是diagnostic logging而不是数据库的write-ahead logging
分为前端和后端.
整个程序只有一个日志后端线程，收集日志消息，缓存并写日志文件，单例模式。
![](Pasted%20image%2020230708205715.png)
#### 日志前端

包括：Logger, LogStream（取代iostream），FixedBuffer，SourceFile。
类图关系：
![](Pasted%20image%2020230708205817.png)
SourceFile类对__FILE__传来的字符串路径进行包装，只记录基本的文件名。

Logger::setOutput设置输出位置，默认为stdout，需要指定为AsyncLogging对象，才能写日志文件。

Logger使用了==桥接模式==
#### 日志后端：AsyncLogging

后端主要包括：AsyncLogging, LogFile, AppendFile，MutexLock。
AsyncLogging 提供后端线程，定时将log缓冲写到磁盘，维护缓冲及缓冲队列。
LogFile 提供日志文件滚动功能，写文件功能。
AppendFile 封装了OS提供的基础的写文件功能。
与日志前端为一对多

异步日志。多线程中有多个日志前端，但仍然只有一个日志后端。是个多生产者-单消费者问题。整个程序使用相同的日志后端，因而是个sigleton。

需要一个队列来将日志前端的消息传到日志后端，防止每一条消息都通知后端。
muduo采用双缓冲区技术实现队列：准备两块buffer A和B ，前端往A中写数据，后端将B的数据写入文件。A满之后，交换A B。批处理操作
![](Pasted%20image%2020230713174520.png)


#### 日志问题

> 日志flush

如果程序崩溃，最后几条日志丢失问题：muduo库使用两个办法：1. 缓存区日志定期flush到硬盘。2 .

>性能优化

1. 时间戳字符串中日期和时间两部分是缓存的，可以复用

### CurrentThread

作用：返回当前线程的tid，并缓存。
--thread int t_cachedTid使用__thread来保存每个线程的线程tid。
void cacheTid()；// 获取tid（全局唯一）并缓存tid，缓存是为了防止多次系统调用
inline int tid()； 不为0，直接返回tid，否则调用cachetid获取再返回tid

### CountDownLatch

倒计时计数器，同步辅助类。
在完成一组正在其他线程中执行的操作之前（计数器大于0），它允许一个或多个线程一直等待。
执行了线程 D 之后，调用 countDownLatch.wait()方法，将会进入阻塞状态，直到 countDownLatch 的 count 参数值为 0；
对条件变量做个封装，加了count。

### 定时器

muduo库取舍：
- 获取时间：gettimeofday(2)
- 定时任务：timefd_* 
使用时间轮，连接超时时间为8s左右。

## net

### Channel

**封装一个文件描述符、感兴趣event和实际发生事件。**
有两种channel：listenfd-AcceptorChannel；connfd-ConnectionChannel。
成员变量包含其监听事件，实际发生事件，所属eventloop，还有四种事件回调，index被poller使用，具体作用未知，tie tcpconnection中使用，用于延长生命周期。
成员函数包括注册事件（底层使用epollctl）、回调函数设置、Channel::handleEventWithGuard：根据epoll监听到fd上发生的事件调用相应的事件回调函数。
muduo 使用`Channel::tie(const boost::shared_ptr<void> &)`来防止`Channel::handleEvent()`引起的`tcpConnection`的析构,

#### 回调函数

都是tcpconnection（read也有acceptor）给设置的。
有四个事件回调函数成员变量：`readCallback_`, `writeCallback_`, `closeCallback_`, `errorCallback_`。由上层设置。
`TcpConnection`会绑定一个socketfd（connfd），因而也会绑定一个channel，在`TcpConnection`的构造函数中调用`setReadCallback`，`setWriteCallback`, `setCloseCallback`, `setErrorCallback`，将connfd channel的四个回调设置为`TcpConnection::handleRead`等。
即channel中发生事件后，执行的是tcpconnection的函数，因而要保证tcpconnection存活------tie成员。

#### 成员

```cpp
fd；event；revent；*loop；tie；四个回调；index（初始化为-1，代表未添加如poller中）

void handleEvent(Timestamp receiveTime); 处理fd上的revents
void update();根据感兴趣事件event，在poller中注册相应事件。通过el与poller交互。
void tie(const std::shared_ptr<void>&); 防止channel被手动remove掉，channel还在执行以上回调，因为el里面有removechannel 调用时机？TcpConnection::connectEstablished。绑定参数为shared_from_this，**即当前TcpConnection对象的共享指针。**。
```

### Poller

基类，派生类为epollpoller和pollpoller。Demultiplex。

#### epoll注册操作

构造函数对应epoll_create
**epoll_wait对应poll函数**
epoll_ctl add/mod/del 对应 update/remove channel函数

channel无法直接与poller交互，因而在其update中调用el的update，最终在调用poller的update。
调用过程：channel remove update => eventloop updateChannel remveChannel => Poller updateChannel removeChannel

#### 成员

```cpp
std::map<int, Channel*> channels_; 存储此poller上的监听的fd及对应channel
EventLoop* ownerLoop_;  // 所属的el， 1对1关系
```

### Eventloop

**事件循环（反应器reactor），==一个线程对应一个eventloop==**，负责IO和定时器事件的分派。使用eventfd来唤醒。
分为mainloop（用户创建/ IO线程）和subloop（子线程创建， 工作线程负责监听连接上的读写事件并进行处理 ）。无论是mainloop的listenfd还是subloop的connfd，都必须设置非阻塞fd。
channels + poller + wakeupfd
loop()
![](Pasted%20image%2020230711013154.png)

> 什么时候将需要监听的fd/channel加入eventloop？

TcpServer::newConnection ->TcpConnection::connectEstablished-> Channel::enableReading-> Channel::update。

>什么时候将该eventloop的wakeupfd加入eventloop？

EventLoop::EventLoop() -> wakeupFd_(createEventfd()), wakeupChannel_(new Channel(this, wakeupFd_)) -> wakeupChannel_->enableReading(); -> Channel::update() 。

main reactor使用轮询的方式给sub reactor来分配新连接。

#### 成员

- `const pid_t`；记录创建此loop的线程，此loop归属此线程。
- `std::unique_ptr< Poller >`；一个loop拥有一个poller，监听多个channel/fd。
- `int wakeupFd_`; **每个eventloop有一个唯一的；**wakeupFd由建立此eventloop的线程监听，因此通过往wakeup中写，可以唤醒此eventloop对应的线程。
- `std::vector< Channel* > activeChannels_`；所有有事件发生的channel，用于epoll_wait传出参数。
- `Channel* currentActiveChannel_`；当前正在处理的channel。

```cpp
  std::unique_ptr<Poller> poller_;  只有一个多路事件分发器
  ChannelList activeChannels_;      监听多个channel
  int wakeFd;                       唯一标识一个subreactor。事件通知机制。据此唤醒。当mainloop获取一个新用户连接的channel，通过轮询算法选择一个subloop，通过该成员唤醒subloop来处理。tinywebserver/libevent使用的是sockpair。

void loop();
void runInLoop(Functor cb);
void queueInLoop(Functor cb);
void wakeup();
```

### EventLoopThreadPool

eventloop的线程池（所有的subreactor），里面每一个线程都是一个EventLoopThread。主要是tcpserver的成员。
默认以轮询的方式给subloop分配connfd channel。

主要成员为：
```cpp
EventLoop* baseLoop_;                mainreactor
int numThreads_;                     subreactor数量
// 保存着这个线程池中的线程及对应loop，不包括主的       subractor集合。
std::vector<std::unique_ptr<EventLoopThread>> threads_;
std::vector<EventLoop*> loops_;

void start(const ThreadInitCallback& cb = ThreadInitCallback());  // 创建subreactor
EventLoop* getNextLoop();  // 默认以轮询的方式给subloop分配channel；不断获取这个线程池中的loop
```

#### EventLoopThread

##### 作用

**one loop per thread实现**。将一个loop和thread封装绑定在一起的类。
EventLoopThread在构造函数中创建thread。

##### 成员

```cpp
EventLoop* loop_ GUARDED_BY(mutex_); // threadFunc中创建
Thread thread_;  // 主要成员变量是这两个。构造函数中船舰

void threadFunc();       thread创建的子线程的运行函数。
EventLoop* startLoop();  **调用thread_.start(); 关键方法：进行子线程的创建。创建的子线                          程会运行void threadFunc()；创建eventloop，并调用                                     loop_.loop() 开始监听。**
```

##### Thread

使用这个类来创建一个线程（start函数），并将其信息封装在成员变量中。在新线程中创建eventloop。 
一个thread只能创建一个eventloop，并与之绑定。

###### 成员

```cpp
pthread_t  pthreadId_;       // 进程内唯一的。改写使用智能指针封装的c++11线程                                           std::shared_ptr<std::thread> thread_;
pid_t      tid_;             // 全局唯一
std::function<void ()> func_; **最重要的**，线程函数。这个是EventLoopThread传进来的。
CountDownLatch latch_;        // 对条件变量的一个封装
static AtomicInt32 numCreated_;  //记录产生的线程的个数。改写使用std::atomic

void start()；
```


### Socket

封装fd，成员变量只有`int sockfd_;`
用于Acceptor

### Buffer

用于TcpConnection中（应用层），作为读写事件的缓冲区（输入输出缓冲区）。AsyncLogging中也使用了（好像不是这个Buffer）
可以解决粘包问题。存储未读数据，和无法一次写入tcp缓冲区的数据。
![](Pasted%20image%2020230711231814.png)
0-readerIndex_为prependable，用于表明长度，解决粘包问题。
readerIndex_-writerIndex_为缓冲区可读的字节。
writerIndex_-size为缓冲区可写的字节。

#### 为什么使用Buffer

见陈硕书
1. 同步发送，tcp缓冲区大小有限制。这样相当于异步。
2. 解决粘包，读到完整数据再拿出。
#### 成员

```cpp
std::vector<char> buffer_;
size_t readerIndex_;
size_t writerIndex_;

const char* peek();返回可读的首地址
void retrieve(size_t len)；如果全部读取完，将可读可写放到默认位置。否则更新读位置。
ssize_t readFd(int fd, int* savedErrno);  // data： kernel socket =》 buffer
```
### TcpConnection

mainloop检测到连接事件将其打包成TcpConnection，发给subloop注册到poller上。
**TcpConnection和Acceptor是两种对立类型。**
代表一个客户端连接。
标识成员为：connfd，connChannel。
一个el（线程）中有多个TcpConnection，此subloop相当于工作线程

#### 回调函数

四个回调函数成员变量：connectionCallback_、messageCallback_、writeCompleteCallback_、highWaterMarkCallback_、closeCallback_

这四个成员变量不在构造函数中初始化，而是在TcpServer::newConnection中调用TcpConnection：：setConnectionCallback等成员函数注册。

#### 创建位置

> 唤醒eventloop

使用eventfd。每个eventloop都有成员变量wakeFd和wakeChannel。每个线程的都不一样。
eventloop阻塞在pool函数，如果不希望某个eventloop阻塞，那么可以向他监听的专有的eventfd中，添加写事件，从而解除阻塞。

#### 成员

```cpp
void handleRead(Timestamp receiveTime);
void handleWrite();  ****
void sendInLoop(const void* message, size_t len);  ***读完客户端数据后，给客户端返回东西。发送数据，应用写的快，而内核发送数据慢，需要把待发送的数据写入缓冲区，而且设置水位回调（降低发送速率）。与handleWrite来做写数据的缓冲。一般是用户调用，比如在echoserver的onmessage中

enum StateE { kDisconnected, kConnecting, kConnected, kDisconnecting }; // 在构造函数中为kDisconnected，析构为kDisconnected
Buffer inputBuffer_;  // 接受数据到inputbuffer  内核buffer =》 inputbuffer =》 用户read
Buffer outputBuffer_; // FIXME: use list<Buffer> as output buffer.  用户send到outputbuffer  用户send =》 outputbuffer =》 kernel buffer   TCP缓冲区是有大小限制的
 std::unique_ptr<Socket> socket_;  // 这是accept返回的connfd。 Acceptor => mainloop, TcpConnection => subloop
 std::unique_ptr<Channel> channel_;
 EventLoop* loop_;  // 此tcpConnection属于的subeventloop。==一个el对应多个tcpConnection==
```

### InetAddress

允许拷贝
封装IP地址和端口号

#### 成员

成员变量：
- sockaddr_in addr_;
- 
inet_addr作用：1. 转成点分十进制 2. 转成网络字节序

### Acceptor

运行在mainreactor。即TCPserver的成员变量。
listenfd的一个封装。
构造中创建listenfd，封装成channel，最后绑定acceptchannel读回调（acceptor::handleread => tcpserver::newconnection）。AcceptChannel只关心读事件（有连接到来）

#### 成员

```cpp
EventLoop* loop_;    // 所属的mainloop
Socket acceptSocket_;  // listenfd
Channel acceptChannel_;  // listenfd =》 channel
int idleFd_;  // accept失败，fd数量不够时时用到

void handleRead();  listenfd上有读事件（IO事件/新用户连接时）执行此函数// 是个回调函数，主listenfd上的连接事件处理函数，即进行accept。注册到accept channel的readCallback_
NewConnectionCallback newConnectionCallback_;  // 此acceptor监听到读事件（连接事件）创建连接后调用的回调函数。这个TcpServer::newConnection函数的功能是轮询一个subEventLoop，并把已经接受的连接分发给这个subEventLoop。
```

### TcpServer

muduo库对外提供的接口。
mainreactor。打包服务器模型所有东西。
进行一个总体的调度。连接各个模块
构造函数中，创建acceptor、eventloopthreadpool，给acceptor绑定用户连接回调
![](Pasted%20image%2020230711031942.png)

#### 成员

- **acceptor eventloop(base loop)的指针**；用作连接分发器。
- Acceptor的独占指针；包装listenfd及其绑定监听。
- eventloopthreadpool的共享指针；
- 四个回调（用户赋值给tcpserver，然后tcpserver在创建tcpconnection的时候设置其回调）；
- std::map<string, TcpConnectionPtr>保存所有tcpconnection名称到其共享指针的映射。

# 使用muduo库编程

需要定义传入baseloop
InetAddress打包服务器IP地址和端口号
设置用户回调：如echoserver设置onMessage回调，收到消息后触发，调用send方法发送同样数据。==用户onMessage回调流程==：`EchoServer::EchoServer()中调用TcpServer::setMessageCallback(EchoServeronMessage)` => `TcpServer::newConnection调用conn->setMessageCallback(TcpServer::messageCallback_);`  => `TcpConnection::handleRead()中调用TcpConnection::messageCallback_ ` => `TcpConnection::TcpConnection中调用Channel::setReadCallback(TcpConnection::handleRead)`，当subloop上有可读事件发生时就会进行这个流程。

# 关键问题

## 生命周期

### 创建与销毁

#### Channel

TcpConnection构造函数中创建

#### TcpConnection

TcpServer::newConnection中创建

# 编程习惯

构造函数多用explicit，防止函数因出现隐式类型转换而程序出现未期望行为。explicit定义时不用写

参数默认值声明定义只能出现一次

# 改写

使用using代替typedef。
只有epoll。
thread类中不再使用pthread而是使用std::thread，线程技术使用std::atmoic。

## 问题

###### one loop per thread

1. 使用thread_local(__ thread)如果不为空说明该线程已经绑定了一个eventloop了。
2. 使用wakeupfd（socketpair双端可读写或者eventfd内核事件通知机制）确保每个线程处理自己的连接的事件。runinloop / queueinloop / wakeup  判断EventLoop 对象是否在自己的线程里面。bool isInLoopThread() const {return threadId_ == CurrentThread::tid();} 四个函数。


###### 服务器惊群现象解决

惊群现象分两种：accept惊群（一半会设置listenfd非阻塞可以避免，linux内核后期已解决）和epollwait惊群。muduo没有惊群现象。
什么是惊群：多个线程/进程阻塞等待同一个事件时，如果这个事件发生，会唤醒所有线程/进程，但最终这个事件只会分配给一个线程，其他线程再次休眠，造成资源浪费。
解决方法：
1. 参照nginx，引入一把锁。使用全局互斥锁，每个子进程在epoll_wait()之前先去申请锁，申请到则继续处理，获取不到则等待，并设置了一个负载均衡的算法（当某一个子进程的任务量达到总设置量的7/8时，则不会再尝试去申请锁）来均衡各个进程的任务量。
2. SO_REUSEPORT， 多个线程监听同一个端口，让内核决定新连接发给哪个线程


###### 为什么使用LT

- LT 能够兼容很多主机；
- 低延迟处理且不会漏消息；
- 并且由于ET模式需要一次性将数据读完才返回，导致进入epoll_wait 的时间延迟了，从用户看来，就是当有一条连接进来的时候，延迟变高了。
- 平均的分配每一个socket上的读取时间，相当于雨露均沾，这并不是坏处，相当于所有的连接都能并发了。

###### subreactor唤醒步骤

每个subreactor都监听属于自己的wakeupchannel。当需要唤醒某个subreactor时，mainreactor向该subreactor所属wakeupfd中写入数据，然后该线程subreactor就会从poll函数处解除阻塞，完成唤醒。

###### mainreactor派发新连接的算法

轮询，然后wakeup。而不是使用异步生产者消费者模型，放到同步队列里面。没有缓冲。
使用系统调用eventfd，创建wakeupfd，直接做线程间的notify，效率更高。
libevent使用的是sockpair系统调用。

###### C++新特性

share_ptr 、 weak_ptr(channel的tie，**实现弱回调**)、std::function、std::bind、enable_shared_from_this、enable_shared_from_this

**weak_ptr 和 share_ptr**
	muduo的源代码，该源码中对于智能指针的应用非常优秀，其中借助shared_ptr和weak_ptr解决了这样一个问题，**多线程访问共享对象的线程安全问题**，解释如下：线程A和线程B访问一个共享的对象，如果线程A正在析构这个对象的时候，线程B又要调用该共享对象的成员方法，此时可能线程A已经把对象析构完了，线程B再去访问该对象，就会发生不可预期的错误。
参考博客：[weak_ptr 的几个应用场景 —— 观察者、解决循环引用、弱回调_函数指针回调 weak_ptr_爱好学习的青年人的博客-CSDN博客](https://blog.csdn.net/qq_53111905/article/details/122240842)

**enanble_shared_from_this**
	当类`A`被`share_ptr<A>`管理，且在类`A`的成员函数里需要把当前类对象`this`作为参数传给其他函数时，就需要传递一个指向自身的`share_ptr`。- 为何不直接传递`this`指针？使用智能指针的初衷就是为了方便资源管理，如果在某些地方使用智能指针，某些地方使用原始指针，很容易破坏智能指针的语义，从而产生各种错误。- 可以直接传递`share_ptr<this>`吗？答案是不能，因为这样会造成两个非共享的`share_ptr`指向同一个对象，未增加引用计数导对象被析构两次（这里可能会有疑问，为什么`bp2=bp1->getptr()`后，`bp2`和`bp1`并非共享的，我在文末会贴出`StackOverflow`上的解释）。
	参考博客：[学习muduo时对enable_shared_from_this的思考_晨哥是个好演员的博客-CSDN博客](https://blog.csdn.net/gc348342215/article/details/123215888)
	可以看到用户通过实现外部回调函数如连接回调onConnection、事件发生时的消息回调onMessage，将这些回调分别设置到TcpServer的成员变量上，TcpServer在创建TcpConnection对象时，将这些用户写好的回调注册给TcpConnection，同时TcpConnection创建时，会将TcpConnection自己的成员函数注册给当前accept返回的connfd对应的Channel对象上，注册的回调如下：
	![](Pasted%20image%2020230719224752.png)
	TcpConnection class是muduo里`唯一默认使用shared_ptr来管理的class，也是唯一继承enable_shared_from_this的class`。

###### TCP粘包问题的解决

有四种方法：定长包、包尾加上\r\n、包头中加上包体长度（muduo buffer做法）、使用复杂的应用层协议。

###### 怎么优雅关闭连接

什么是优雅关闭连接：
- 保持连接的某一端想关闭连接了，但它需要确保要发送的数据全部发送完毕以后才调用close，此种情况下也需要使用优雅关闭；

优雅关闭连接的方法：
- `setsockopt(fd, SOL_SOCKET, SO_LINGER,` `(char*)&ling,` `sizeof(ling));` +`close(fd);`
- 使用`shutdown(s, SHUT_WR);` `//就是说不会再有人往s上写数据了，那么服务端读取时自然就会读到EOF`进行优雅关闭连接（关闭写端）
###### 判断对端连接关闭

有三种方法：
- ==一是使用read返回值，如果返回0，并且errno=EAGAIN，则说明连接被对方关闭（muduo使用的是这一种）==
- 使用心跳包，长时间没有接到心跳包时，说明连接断开
- 使用getsockopt判断连接状态，若是TCP_ESTABLISHED，则说明连接未断开，否则说明连接断开；
###### 如果连接中需要处理复杂任务，如文件传输怎么半？

用muduo中的线程池做复杂计算任务。新创建一个线程，防止长时间阻塞。 
![](Pasted%20image%2020230801163854.png)

###### muduo库连接关闭处理

连接关闭分为：主动关闭、被动关闭和错误关闭三类（服务器视角），不论哪一类都要保证将此连接移除。

- 主动关闭：只有shutdown这一方法。当用户（服务端用户）主动关闭了连接后，服务器将关闭写操作一方，客户端还可以照常向服务器进行写操作，如果客户端一直不调用read来处理shutdown发来的FIN包，那么将一直存在此半连接状态。但是如果一旦read被调用将返回0，从而客户端主动close连接，此时客户端向服务器发送FIN包，服务器的epoll将对此事件触发EPOLLIN+EPOLLHUP（对端关闭）事件，服务器的handleRead函数处理此事件返回0，从而将连接移除，最后将此连接彻底移除。
- 被动关闭：客户端主动发起连接关闭操作，然后epoll触发EPOLLIN事件，调用handleRead函数read返回0，从而关闭移除连接。
- **客户端错误关闭**：EPOLLERR+EPOLLHUP，这两个会和EPOLLIN同时出现。由于它们和EPOLLIN一起出现那么在handleRead函数中的read操作会返回0（注意，read如果直接读取RST包会返回-1，但是如果经过epoll接收到RST后触发的EPOLL那么read将返回0），当read返回0后服务器将会调用handleClose函数，从而将连接进行关闭和移除。
- **服务端错误关闭**：进行信号捕获和处理

例如如果客户端连接已经关闭，而服务器端仍然用send或者EPOLLOUT向对端发送数据，那么对端将发送RST包，从触发epoll上的EPOLLIN+EPOLLERR+EPOLLHUP，从而开始关闭移除连接和错误处理。
###### 详解几种epoll事件

- EPOLLIN：套接字可读
- EPOLLOUT：套接字可写；缓冲区有空间就会一直触发
- EPOLLRDHUP：对端关闭了套接字，或者对端关闭了写
- EPOLLPRI：套接字上有紧急数据到达
- EPOLLHUP：对端挂断了套接字；
- EPOLLONESHOT

###### ET和LT区别


###### 怎么解决超时连接？

传统的定时方案是以固定频率调用起搏函数tick，进而执行定时器上的回调函数。而时间堆的做法则是将所有定时器中超时时间最小的一个定时器的超时值作为心搏间隔，当超时时间到达时，处理超时事件，然后再次从剩余定时器中找出超时时间最小的一个，依次反复即可。
具体结构使用一个heaptimer定时器容器，里面使用vector作为存放timenode的小根堆，使用一个map来映射连接到堆索引。
```cpp

// 定时器节点
struct TimerNode{
    int id;
    TimeStamp expires;
    TimeoutCallBack cb;
    bool operator<(const TimerNode& t){
        return expires < t.expires;
    }
};

class HeapTimer{
private:
    std::vector<TimerNode> heap_;
    std::unordered_map<int, size_t> ref_;  // key：节点id, value：数组索引 
};
```
###### reactor和proactor

- **Reactor 是非阻塞同步网络模式，感知的是就绪可读写事件**。在每次感知到有事件发生（比如可读就绪事件）后，就需要应用进程主动调用 read 方法来完成数据的读取，也就是要应用进程主动将 socket 接收缓存中的数据读到应用进程内存中，这个过程是同步的，读取完数据后应用进程才能处理数据。
- **Proactor 是异步网络模式， 感知的是已完成的读写事件**。在发起异步读写请求时，需要传入数据缓冲区的地址（用来存放结果数据）等信息，这样系统内核才可以自动帮我们把数据的读写工作完成，这里的读写工作全程由操作系统来做，并不需要像 Reactor 那样还需要应用进程主动发起 read/write 来读写数据，操作系统完成读写工作后，就会通知应用进程直接处理数据。
###### 压力测试

###### 多线程与锁

访问临界区时需要上锁：可能是共享数据结构
锁类型：互斥锁（**sleep-waiting**），自旋锁（**busy-waiting**），读写锁，条件锁（条件变量），递归锁(可重入锁)
实际类型：
- mutex：简单互斥锁。互斥量是信号量的特例，即值为1.
- semaphore：信号量。当value=1时相当于互斥锁。无法一次唤醒所有的。有一个状态值。
- condition.wait(guard, [] {return true;});：条件变量。需要和互斥锁一起使用。wait signal notify_one notify_all
- lock_guard(mutex, std::adopt_lock_t)：RAII。**这个构造函数假定：当前线程已经上锁成功，所以不再调用lock()函数**。析构函数中自动解锁，不能中途解锁
- std::unique_lock：std::lock_guard的加强版。可以随时加解锁。创建时可以不锁定（通过指定第二个参数为 std::defer_lock），而在需要时再锁定。条件变量需要该类型的锁作为参数（此时必须使用 unique_lock）
锁类型相关的 Tag 类：
- constexpr adopt_lock_t adopt_lock {};
- constexpr defer_lock_t defer_lock {};
- constexpr try_to_lock_t try_to_lock {};

**muduo多线程实现**
- std::atomic：保证Atomic对象在多线程环境下的安全。
- `std::mutex`：保证对象在临界区中唯一持有，避免多线程出现数据竞争的情况。
- `std::condition_variable`：条件变量，在多线程中配合互斥锁使用，进行阻塞同步等待执行通知。
- `CountDownlatch`：用在主线程阻塞等待其他多个线程准备就绪后继续执行。使用mutex+condition+count(int)实现。**其实就是个信号量？**，类似，但是这个是线程计数器。**等待多个线程全部执行完，再执行某个任务**

## 难点

事件发生如何处理？回调。

==Acceptor对应TCPConnection，分别代表listenfd和connectfd，分别位于mainloop/mainreactor和subloop/subreactor中，但是mainloop和Acceptor一一对应，一个subloop中可以有多个TCPconnection==

>处理客户端异常退出



## 发送流程

1、调用TcpConnection::send
2、调用TcpConnection::sendInLoop 

      1、如果目前可以发送（channel没有在写，并且缓冲区没有数据）,先尝试发送
       2、如果没有完全发送，将数据添加到buffer中，添加对channel对应的socket写事件关心，等待socket可写，然后调用handlewrite
       3、如果发送完全，调用用户回调函数
3、如果没有发送完全，并且socket可读，调用TcpConnection::HandleWrite。继续发送，如果发送完全，就取消对socket可写事件关系。如果没有发送完全，等待下一次的发送。


## 连接建立流程

## 连接关闭流程

函数调用栈：用户tcpconnection::shutdown => shutdowninloop => socket => shutdownwrite => Poller触发epollhup事件 => channel::closecallback => 由tcpconection给出，为tcpconnection::handleclose => channel::disableall =>closecallback => tcpserver::remveConnection => TcpConnection::connectDestroyed => channel_->disableAll(); => channel_->remove();