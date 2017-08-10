---
title: c语言 - socket编程(四)
date: 2017-08-10 11:21:00
categories:
- c
tags:
- c
keywords: c语言 socket编程 Linux 5种IO模型
---

> 
c语言 - socket编程(四)，Linux中的五种网络IO模型：`阻塞IO`、`非阻塞IO`、`多路复用IO`、`信号驱动式IO`、`异步IO`

<!-- more -->

## IO模型
网络IO的本质是socket的操作，我们以recv为例：

每次调用recv，`数据会先拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间`；

所以说，当一个recv操作发生时，它会经历两个阶段：
- 第一阶段：等待数据准备
- 第二阶段：将数据从内核拷贝到进程中

对于socket流而言：
- 第一阶段：通常涉及等待网络上的数据分组到达，然后被复制到内核的某个缓冲区
- 第二阶段：把数据从内核缓冲区复制到应用进程的缓冲区

**网络IO模型**
- 同步IO(synchronous IO)
 - 阻塞IO(bloking IO)
 - 非阻塞IO(non-blocking IO)
 - 多路复用IO(multiplexing IO)
 - 信号驱动式IO(signal-driven IO)
- 异步IO(asynchronous IO)

> 
由于`信号驱动式IO`实际中并不常用，所以接下来只介绍其他四种IO模型

**阻塞IO**
应用程序调用一个IO函数，如recv：
当socket接收缓冲区中没有数据时，当前进程会被recv阻塞，一直会等待数据的到来；
当socket接收缓冲区中有数据时，recv将数据拷贝到进程空间的这个过程，也是阻塞的；

> 
所以，对于阻塞IO，在这两个阶段都被阻塞了

**非阻塞IO**
将一个套接字设置为非阻塞时，就是告诉内核，当一个请求的IO操作无法立即完成时，不要让我等待，应立即给我返回一个错误，如recv：
当socket接收缓冲区中没有数据时，recv会立即返回一个错误，errno为`EAGAIN`，表示现在没有数据，等下再来吧；
当socket接收缓冲区中有数据时，recv将数据拷贝到进程空间的这个过程，也是阻塞的；

所以对于非阻塞IO，通常采取轮询polling的方式，循环往复的主动询问内核，当前是否有数据了

> 
对于非阻塞IO，第一个阶段不会阻塞，但是第二个阶段依旧是阻塞的

**IO多路复用**
其实这种方式和第二种的非阻塞IO很相似，只不过优点是可以借助这几个特殊的系统调用(`select`、`poll`、`epoll`)，来同时轮询多个socket连接：
当调用select、poll、epoll函数时，如果所监控的socket中有部分socket可读、可写或其他事件发生时，就会返回，将其交给用户进程来处理，这个过程是阻塞的，只不过是因为select、poll、epoll系统调用而阻塞的；
当调用返回后，用户进程再调用recv，将数据从内核拷贝到进程空间中，这个过程也是阻塞的，因recv拷贝数据而阻塞；

实际上这种方式相比第二种还差一些，因为这里面包含了两个系统调用(select/poll/epoll、recv)，而第二种只有一个系统调用recv；
不过IO多路复用的优点是可以处理更多的连接，当连接数大的时候，缺点就被优点给掩盖了

IO多路复用相比`多进程/多线程 + 阻塞IO`的系统开销小，因为系统不需要创建新的进程或线程，也不需要维护多个进程、线程的执行

> 
对于多路复用IO，第一个阶段是因为select、poll、epoll而阻塞的，第二个阶段(实际IO操作)依旧是阻塞的

**信号驱动式IO**
首先我们允许socket进行信号驱动IO，并安装一个信号处理函数，进程继续运行并不阻塞；
当数据准备好时，进程会收到一个SIGIO信号，可以在信号处理函数中调用IO操作函数处理数据；

**异步IO**
对于前面四种IO模型，都是同步的，因为所有的实际IO操作(将数据从内核拷贝到进程空间的这个过程)都是阻塞的；
所谓同步IO，就是必须等待当前的IO操作完成之后，才能执行后面的指令，这个等待的过程中是不能进行其他操作的，也就是说，指令序列都是线性执行的；
而异步IO，当遇到IO操作时，直接把这个IO操作交给别人(内核、新的线程/进程等等)来做，而当前进程并不会因为这个IO操作而阻塞，可以立即执行别的操作，当IO操作完成后，进程会收到完成的通知(回调函数、信号等等)

linux2.6之后的内核提供了AIO库，提供异步IO操作，但是实际上用的比较少，目前有很多流行的开源异步IO库：libevent、libev、libuv等