---
layout: post
title: "🍉有关select/poll/epoll和socket"
date: 2021-03-24
tags:  Socket
---

说起来有些惭愧，虽然在校有接触过socket编程，但是实际上并没有深入使用过，大部分知识都只是书本上或是从面经上看来的。在实际使用过后，才能更深入地理解它们之间的区别。

select/poll/epoll都是为了更高效地管理多个socket而实现的。对于服务端来说，准备步骤中的socket()/bind()/listen()的使用都是一致的，这三种方式在管理socket上有所不同。

## select

实际上是使用bitmap来记录需要管理的socket对应的文件描述符fd，默认大小为1024个bit，也就是4个32位整数的空间大小。采用bitmap使得每次将需要监听的fd信息传入内核空间时，只需要4个32位整数大小数据的复制，即16个byte。

但是bitmap过于精简，只能记录0/1，无法记录更多状态。当中断到来，执行完相应的中断程序后，只能将1024位bit中，可操作的fd位置的bit置为1返回，这意味着原来的监听记录信息被破坏，每次select都需要重新设置。

其次，由于返回的信息过于简单，需要自身遍历fd数组，挨个检查其对应位置是否置1，来判断能够操作的socket。

当有新的连接或是连接关闭，都需要自己管理好fd数组，以便对应地更新bitmap的状态。

所以每次循环，大致分为以下过程：

1. 循环fd数组，设置bitmap - O(n)
2. select阻塞 - 拷贝16byte到内核空间
3. 得到置位后的bitmap - 拷贝16byte到用户空间
4. 循环fd数组，根据bitmap判断置位情况 - O(n)

## poll

```c
struct pollfd 
{ 
    int fd; /* 想查询的文件描述符. */ 
    short int events; /* fd 上，我们感兴趣的事件*/ 
    short int revents; /* Types of events that actually occurred. */ 
};
```

放弃使用bitmap，原来的1024bit表示一组fd直接变成了一个结构体（pollfd）表示一个fd。如上所示，我们通过管理pollfd数组来管理socket的fd，按照数据对齐原则，这样的一个结构体大小应为8 byte。

若管理n个fd，则对应数组大小为n*8 byte。原来的置位操作，变成了修改revents，events用于保存该fd希望监听的事件。

所以每次循环，大致分为以下过程：

1. poll阻塞 - 拷贝n*8byte到内核空间
2. 得到置位后的pollfd数组 - 拷贝n*8byte到用户空间
3. 循环pollfd数组，根据revents判断是否有事件发生，需要重置对应的revents为0 - O(n)

显然省了一个O(n)，但是多出了很多数据的来回拷贝。

## epoll

主要思想是别在用户空间管理fd了，拷贝来拷贝去开销太大。直接在内核中创建一个数据结构用于管理fd，结构主体为红黑树（最开始是用的数组）。所以多出来了几个方法给人调用，用于间接操作内核空间中的数据。

1. epoll_create(int size)：数据结构初始化（使用红黑树后，不需要提前分配好size了，所以size不为0即可），实际上相当于创建了一个对象（返回一个文件描述符）。
2. epoll_ctl(...)：在1中得到的对象上进行操作，包括add/delete/mod，就是常见的红黑树操作。
3. epoll_wait(...)：终于得到了解放，啥都不用传了，数据都在内核空间创建好了（大概罢了，详情见下文）

虽然fd的管理都直接在内核空间进行了，但最终要得到有事件的socket的描述符，还是得传数据到用户空间才行，如果还是所有fd状态都拷贝到用户空间，那么和select/poll这一部分的差别不大。因此epoll只将有事件的fd拷贝回用户空间，这样避免了数据的大量拷贝，也避免了遍历fd数组的过程。要实现这一效果，需要内核空间中维护一个有事件的fd队列，所幸，这个队列的维护是自动的，不需要手动干预。

需要拷贝回的数据是这样的格式：

```c
struct epoll_event
{
  uint32_t events;   /* Epoll events */
  epoll_data_t data;    /* User data variable */
} __attribute__ ((__packed__));

typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

也就是我们需要提前在用户空间分配好数据所需的空间，让后交由epoll_wait(...)返回时填充，得到的都是触发了事件的fd。

`__attribute__((__packed__))`意味着这个结构体别对齐了，拷贝数据会要命的，直接挨个紧凑排列吧，也就是说epoll_event大小为12byte。

因此每次循环，大致分为以下过程：

1. epoll_wait() - 拷贝k*12byte到用户空间（k可以自行设定，每次取多少个是自定义的）
2. 遍历取出的k个epoll_event - O(k)

可以看到，epoll通过把麻烦留给内核的方式，简化了用户侧的使用，减少了一些步骤的开销。但由于维护数据结构本身就有一定的开销，所以对于简单少量的连接，select/poll说不定会更好。

同时，由于epoll是后来实现的，时代在发展，因此也增添了一些新的特性。例如有了level-trigger和edge-trigger这两种触发模式，这里简称为lt和et。

lt模式下，数据不读取完，是一直处于事件触发状态的，需要处理完数据才会取消触发状态，就算这次不处理完，下次循环还可以继续；这对于单线程无所谓，但在多线程的情况下，这意味着可能一份数据，被拆成几份，被不同的线程在处理，这显然是不恰当的。

et模式下，一旦有线程取得事件，则该触发状态取消，这意味着取得事件的线程需要自行独立一次处理完数据，这次不处理完，下次循环也就没了；多线程情况下，有可能产生竞争，不一定能保证安全。

所以，对于fd还提供了一种事件类型，EPOLLONESHOT，这意味着该fd事件被获取后，便在红黑树中被标记为隐藏了，不再处于监听状态。因此线程可以安心处理数据，当然也是需要一次处理完，不过处理完后需要记得使用epoll_ctl()修改其事件属性，相当于重新加入监听列表。

## 实际使用中遇到的问题

服务端程序在关闭后，相当于TCP四次挥手过程中的主动断开方，其监听的ip:port会进入TIME_WAIT状态，原因有以下几点：

1. 其send(...)发送的数据可能还在缓冲区，需要一定时间发送出去；
2. 这次连接过程中，末尾部分可能有些数据包还在网络中，TIME_WAIT确保最终网络上没有了连接中的东西，完全clean，再安全关闭。

TIME_WAIT的设定使得服务端程序结束后，没办法立刻重新启动；为了解决这一点，可以在socket()创建好后，使用setsockopt()设定SO_REUSEADDR，这意味着ip:port在TIME_WAIT阶段便解除了独占状态（也仅限于此，正常状态下还是独占的），新socket不会因为TIME_WAIT占用而冲突，导致无法立刻bind。

### 关于SO_REUSEPORT

类似的，还有一个SO_REUSEPORT（Kernel 3.9，修改了bind以及连接建立过程），这个实际上才是真正的`REUSEADDR`。上文说到，SO_REUSEADDR实际上只能处理TIME_WAIT阶段的ip:port，而SO_REUSEPORT就强大得多，在如今服务器后端开发中十分重要。下面是手册中的说明：

>The new socket option allows multiple sockets on the same host to bind to the same port, and is intended to improve the performance of multithreaded network server applications running on top of multicore systems.

这意味着，不同的套接字，真正的可以绑定到完全一致的ip:port上，并由内核来分配数据，这种特性对于多核CPU来说十分友好。如果不使用SO_REUSEPORT，由于ip:port和socket一对一绑定，那么想要多个进程来处理ip:port的连接请求，就只能通过以下两种方式来解决：

1. 单线程listen()/accept()，然后将accept()得到的socket分发给工作线程处理。显然，瓶颈在listen()/accept()。
2. fork()，多个子进程共享server_socket来accept()。多进程使用server_socket会产生锁竞争，开销很大。

而使用SO_REUSEPORT的话，则可以随意扩展独立进程，来处理更多的请求，且互相之间独立，由内核完成分配工作；甚至可以新旧程序平行运行，共同处理。但是SO_REUSEPORT不是完美的，ip:port上server_socket数量变化时，可能将在server_socket_a上传输的数据包（连接建立中），发送给server_socket_b来处理，导致出错。

## 总结

至此，select/poll/epoll算是有了简单的认识，更多细节什么的，只需要翻阅文档即可。