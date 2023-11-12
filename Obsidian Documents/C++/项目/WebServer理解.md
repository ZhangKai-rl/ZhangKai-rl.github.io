
# 程序运行逻辑

main.cpp中main函数为程序入口

首先设置数据库参数和解析命令行参数（getopt）用其初始化server
```c++
	//获取日志实例
    server.log_write();
    //连接数据库
    server.sql_pool();
    //创建线程池
    server.thread_pool();
    //触发模式  ET/LT
    server.trig_mode();

    //监听  
    // 设置监听套接字；设置epoll；捕获信号；设置定时器
    server.eventListen();
    //初始化完成后运行进入事件循环
    server.eventLoop();
```
eventloop里面是一个循环
循环中的逻辑：
调用epoll_wait，根据返回值循环处理所有时间



# 面试问题

**为什么http连接fd加入到epoll事件中时要设置oneshot？**

  无论是et还是lt模式，一个线程在处理一个socket的事件，在处理的时候，这个socket又来了一个事件，那么就会调用另一个线程处理新时间，这样会产生多个线程处理同一个socket事件，出现竞态问题。注册oneshot后，如果正在处理那么这个socket出现新事件不会再去响应。
![](Pasted%20image%2020230425231000.png)

**项目亮点和难点？**

  基于生产者消费者模型建立线程池

**WebServer为什么需要将socket设置为非阻塞？**

**惊群现象是什么？怎么解决？**

当一个请求上来的时候，多个进程都会被 select/poll/epoll_wait 唤醒去 accept，然而只有一个进程(线程 accept 成功，其他进程(线程 accept 失败，然后重新阻塞在 select/poll/epoll_wait 系统调用上。

**epoll EL和LT的区别和应用场景？**

应用场景：
  EPOLL水平触发(LT)：
1.  适用于处理少量数据或者需要确保数据完整性的场景。
2.  适用于使用阻塞式IO模型（如select）的应用程序迁移到epoll的过渡期。

EPOLL边沿触发(ET):
1.  适用于高并发、高吞吐量的场景，即当数据流到达时，内核立即通知应用程序进行读写操作，避免了重复触发事件和重新读取套接字缓存的开销。
2.  适用于需要精确控制的场景，即当数据流到达时，内核只会通知应用程序一次，如果应用程序没有完整地处理数据，则会在下一次epoll_wait()调用时再次通知应用程序进行读写操作。


**为什么要将socket设置为非阻塞？**

[WebServer为什么需要将socket设置为非阻塞？_epoll 非阻塞socket client_爱吃芝麻球的博客-CSDN博客](https://blog.csdn.net/ccw_922/article/details/124625718)



# 类

## Config

### 作用

解析命令行参数

### 成员

```C++
public :
    //端口号
    int PORT;
    //日志写入方式
    int LOGWrite;
    //触发组合模式
    int TRIGMode;
    //listenfd触发模式
    int LISTENTrigmode;
    //connfd触发模式
    int CONNTrigmode;
    //优雅关闭链接    ?
    int OPT_LINGER;   
    //数据库连接池数量
    int sql_num;
    //线程池内的线程数量
    int thread_num;
    //是否关闭日志
    int close_log;
    //并发模型选择
    int actor_model;   //proactor reactor
   
    Config();
    ~Config(){};
    void parse_arg(int argc, char*argv[]);

```

疑问？
 为什么listenfd和connfd都有两种epoll触发模式

parse_arg主要是用getopt函数实现


## WebServer

## http_conn

## connection_pool

数据库连接池
![](Pasted%20image%2020230504214420.png)
在本部分内容中通过connectionRAII类改变外部MYSQL指针使其指向连接池的连接，析构函数归还连接入池。
使用connectionRAII类完成RAII机制。在该类的构造函数中获取连接池中的连接，并将外部MYSQL连接指向该连接，在该类的析构函数中释放该连接，并将其入池。这样使用，当该对象声明结束时，编译器会自动调用其析构函数，回收该连接，避免手动释放。
在该类的构造函数中获取连接池中的连接，并将外部MYSQL连接指向该连接，在该类的析构函数中释放该连接，并将其入池。这样使用，当该对象声明结束时，编译器会自动调用其析构函数，回收该连接，避免手动释放。

## threadpool

## Utils

## log

包含block_queue类和log类。

### block_queue

使用循环数组实现阻塞队列类。
具有超时处理功能。
只有异步方式写日志时才会用到阻塞队列。
阻塞队列几乎每个操作都要使用互斥锁来保证线程安全。
队列本身使用条件变量，当有数据入队列时就通知消费者线程去消费日志。

单例模式实现，使用局部懒汉来保证不加锁实现线程安全。
包括同步/异步两种日志实现。
异步使用生产者消费者模型实现阻塞队列。主线程的日志打印接口仅负责生产日志数据（作为日志的生产者），而日志的落地操作留给另一个后台线程去完成（作为日志的消费者）
![](Pasted%20image%2020230502225611.png)

### 改进

[(89条消息) 每秒百万级高效C++异步日志实践_LeechanXBlog的博客-CSDN博客](https://blog.csdn.net/linkedin_38454662/article/details/72921025)

## lst_timer.h

这个文件里面定义了client_data结构体，还有util_timer、sort_timer_lst、Utils几个类。
util_timer类，为每个用户（http连接）创建一个定时器，是升序链表中的一个节点。存储的是绝对时间
sort_timer_lst是一个定时器容器，利用升序双向链表封装用户定时器(util_timer)。整个服务器维持一个定时器链表。
sort_timer_lst::tick()
Utils，工具类。包含用于信号传递的管道fd，升序链表定时器容器，epollfd，alarm函数的TIMESLOT。
![](Pasted%20image%2020230504175521.png)






在哪里close的connfd？

在Utils::cb_func指向的函数中进行close(connfd);

页面跳转逻辑也是类似于一个有限状态机

在`void WebServer::eventListen()`函数中创建了epollfd和epoll_event。

整个服务器维护的定时器升序链表在webserver::Utils utils::sort_timer_lst m_timer_lst中




![](Pasted%20image%2020230506154215.png)





