 # 阻塞与非阻塞、异步与同步

阻塞（block）与非阻塞与系统调用（system call）息息相关

阻塞会同步的等待函数返回结果，等待过程中进程\线程可以会进入阻塞状态

一个**非阻塞I/O 系统调用 read()** 操作立即返回的是任何可以立即拿到的数据， 可以是完整的结果， 也可以是不完整的结果， 还可以是一个空值。所以可能会循环不停的获取值

而**异步I/O系统调用** read（）结果必须是完整的， 但是这个操作完成的通知可以延迟到将来的一个时间点。所以可能等待回调函数来获取值



#linux 五种I/O模型

阻塞IO、非阻塞IO、IO多路复用、信号驱动IO、异步IO

前面4种都是同步IO模型



# select、poll、epoll

select O(N):内核需要将消息传递到用户空间，都需要内核拷贝动作。需要维护一个用来存放大量fd的数据结构，使得用户空间和内核空间在传递该结构时复制开销大。

- 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
- 同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
- select支持的文件描述符数量太小了，默认是1024



poll O(N)：poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，都需要将大量内容从用户态拷贝到内核态、开销很大。

poll没有最大连接数限制、其采用链表存储



epoll O(1)：

- LT，默认的模式（水平触发） 只要该fd还有数据可读，每次 `epoll_wait` 都会返回它的事件，提醒用户程序去操作，
- ET是“高速”模式（边缘触发） 只会提示一次，直到下次再有数据流入之前都不会再提示，无论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读完，即读到read返回值小于请求值或遇到EAGAIN错误

1、没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）；
2、效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；
即Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。

3、 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。



# socket

TCP：

```text
       被动socket(server)              主动socket(client)
--------------------------------------------------------------
            socket()
               |
            bind()
               |
            listen()                        socket()
               |                              |
            accept()                          |
               |                              |
               | <------------------------- connect()
               |                              |
            read() <------------------------ write()
               |                              |
            write() -----------------------> read()
               |                              |
            close()                         close()
---------------------------------------------------------------
```



UDP：

```text
             server                         client
-----------------------------------------------------------------
            socket()
               |
            bind()                          socket()
               |                              |
            recvfrom() <------------------- sendto()
               |                              |
            sendto() ---------------------> recvfrom()
               |                              |
            close()                         close()
-----------------------------------------------------------------
```