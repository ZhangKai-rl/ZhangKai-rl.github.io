
# IO多路复用

>为什么使用IO多路复用？

监听多个客户端的连接。多路IO复用的系统调用有：select、poll、epoll。

>select、poll、epoll的区别

- select：使用一个bitmap维护已连接socketfd的集合，将其拷贝的内核中，内核遍历socketfd查看是否有事件发生，然后将其标记为可读可写。接着将skcketfd拷贝回用户态，在用户态再遍历找到可读可写的socketfd。**需要两次拷贝，两次遍历，使用用户态固定长度bitmap**
- poll：使用动态数组，以链表的形式组织。突破了 select 的文件描述符个数限制，当然还会受到系统文件描述符限制。**仍然是线性结构，需要两次拷贝两次遍历。**
- epoll：epoll直接在内核中使用红黑树结构来追踪socketfd。内核中维护了一个链表结构来追踪发生时间的socketfd，一旦红黑树中上的socketfd有事件发生就会加入此链表，然后拷贝回用户态。无需遍历。
![](Pasted%20image%2020230719161116.png)