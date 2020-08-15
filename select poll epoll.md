# select、poll、epoll之间的区别总结整理

(1)select==>时间复杂度O(n)

它仅仅知道了，有I/O事件发生了，却并不知道是哪那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。所以**select具有O(n)的无差别轮询复杂度**，同时处理的流越多，无差别轮询时间就越长。

(2)poll==>时间复杂度O(n)

poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态， **但是它没有最大连接数的限制**，原因是它是基于链表来存储的.

(3)epoll==>时间复杂度O(1)

**epoll可以理解为event poll**，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是**事件驱动（每个事件关联上fd）**的，此时我们对这些流的操作都是有意义的。**（复杂度降低到了O(1)）**

epoll跟select都能提供多路I/O复用的解决方案。在现在的Linux内核里有都能够支持，其中epoll是Linux所特有，而select则应该是POSIX所规定，一般操作系统均有实现

**select：**

select本质上是通过设置或者检查存放fd标志位的数据结构来进行下一步处理。这样所带来的缺点是：

1、 单个进程可监视的fd数量被限制，即能监听端口的大小有限。

​      一般来说这个数目和系统内存关系很大，具体数目可以cat /proc/sys/fs/file-max察看。32位机默认是1024个。64位机默认是2048.

2、 对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低：

​       当套接字比较多的时候，每次select()都要通过遍历FD_SETSIZE个Socket来完成调度,不管哪个Socket是活跃的,都遍历一遍。这会浪费很多CPU时间。如果能给套接字注册某个回调函数，当他们活跃时，自动完成相关操作，那就避免了轮询，这正是epoll与kqueue做的。

3、需要维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大

**poll：**

poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时，被唤醒后它又要再次遍历fd。这个过程经历了多次无谓的遍历。

**它没有最大连接数的限制**，原因是它是基于链表来存储的，但是同样有一个缺点：

1、大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制是不是有意义。                   

2、poll还有一个特点是“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。

**epoll:**

epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作，而在ET（边缘触发）模式中，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无 论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读光，也就是说一直读到read的返回值小于请求值，或者 遇到EAGAIN错误。还有一个特点是，epoll使用“事件”的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，epoll_wait便可以收到通知。

**epoll为什么要有EPOLLET触发模式？**

如果采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.。而采用EPOLLET这种边沿触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你！！！**这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符**

**epoll的优点：**

1、**没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）**；
**2、效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；**
**即Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。**

3、 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。
**select、poll、epoll 区别总结：**

1、支持一个进程所能打开的最大连接数

select

单个进程所能打开的最大连接数有FD_SETSIZE宏定义，其大小是32个整数的大小（在32位的机器上，大小就是32*32，同理64位机器上FD_SETSIZE为32*64），当然我们可以对进行修改，然后重新编译内核，但是性能可能会受到影响，这需要进一步的测试。

poll

poll本质上和select没有区别，但是它没有最大连接数的限制，原因是它是基于链表来存储的

epoll

虽然连接数有上限，但是很大，1G内存的机器上可以打开10万左右的连接，2G内存的机器可以打开20万左右的连接

2、FD剧增后带来的IO效率问题

select

因为每次调用时都会对连接进行线性遍历，所以随着FD的增加会造成遍历速度慢的“线性下降性能问题”。

poll

同上

epoll

因为epoll内核中实现是根据每个fd上的callback函数来实现的，只有活跃的socket才会主动调用callback，所以在活跃socket较少的情况下，使用epoll没有前面两者的线性下降的性能问题，但是所有socket都很活跃的情况下，可能会有性能问题。

3、 消息传递方式

select

内核需要将消息传递到用户空间，都需要内核拷贝动作

poll

同上

epoll

epoll通过内核和用户空间共享一块内存来实现的。

**总结：**

**综上，在选择select，poll，epoll时要根据具体的使用场合以及这三种方式的自身特点。**

**1、表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。**

**2、select低效是因为每次它都需要轮询。但低效也是相对的，视情况而定，也可通过良好的设计改善** 

 





select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。**但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间

**1、select实现**

**select的调用过程如下所示：**

**![img](.\image\17201205-8ac47f1f1fcd4773bd4edd947c0bb1f4.png)**

（1）使用copy_from_user从用户空间拷贝fd_set到内核空间

（2）注册回调函数__pollwait

（3）遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）

（4）以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。

（5）__pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同设备有不同的等待队列，对于tcp_poll来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。

（6）poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。

（7）如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。

（8）把fd_set从内核空间拷贝到用户空间。

**总结：**

**select的几大缺点：**

**（1）每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大**

**（2）同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大**

**（3）select支持的文件描述符数量太小了，默认是1024**

**poll实现**

　　poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，其他的都差不多。

**epoll**

　　epoll既然是对select和poll的改进，就应该能避免上述的三个缺点。那epoll都是怎么解决的呢？在此之前，我们先看一下epoll和select和poll的调用接口上的不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。

　　对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

　　对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。

　　对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

**总结：**

（1）**select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。**

（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。





## Linux的socket事件唤醒回调机制(wakeup callback)

唤醒回调机制是IO多路复用机制存在的本质：

Linux通过socket睡眠队列来管理所有等待socket的某个时间的进程，同时通过唤醒机制来异步唤醒整个睡眠队列上等待事件的进程，通知进程相关事件发生（通常情况，socket的事件发生的时候，会顺序遍历socket睡眠队列上的每个进程节点，调用进程节点挂载的callback回调函数。遍历过程中，如果某个节点是排他的，终止遍历即可）：总体说，可分为以下两个逻辑

+ 睡眠等待逻辑：涉及select、poll、epoll_wait的阻塞等待逻辑

  > 【1】：select、poll、epoll陷入内核，判断监控的socket是否有关心的事情发生，如果没有，则为当前进程构建一个wait_entry节点，然后插入到监控socket的睡眠等待队列中去
  >
  > 【2】：循环监控等待关心的事情发生
  >
  > 【3】：当关心的事件发生后，将当前进程的wait_entry节点从socket的睡眠队列中移除

+ 唤醒逻辑：

  > 【1】：socket事件发生后，socket顺序遍历其睡眠队列，依次调用每个wait_entry节点的callback函数
  >
  > 【2】：直到完成队列的遍历 或者 遇到某个wait_entry节点是排他的才停止
  >
  > 【3】：callback一般情况下包含两个逻辑：
  >
  > + wait_entry自定义的私有逻辑
  > + 唤醒的共用逻辑，主要用于将该wait_entry的进程放入CPU就绪队列中，让CPU随后可以调度执行



## select

```cpp
#include <sys/select.h>
int select (int nfds, 
    fd_set *readfds, fd_set *writefds, fd_set *exceptfds,             struct timeval *timeout);
```

select函数 监视的文件描述符分3类：读、写、异常

当用户进程调用select的时候，select会将需要监控的readfds集合拷贝到内核空间（假设监控的仅仅是socket可读），然后在readfds集合中遍历自己监控的socket sk   ，挨个调用sk的poll逻辑以检查该sk是否有可读事件，遍历完所有的sk后，如果没有任何一个sk可读，那么select会调用schedule_timeout进入schedule循环，是的用户进程进入睡眠；如果在timeout时间内某个sk上有数据可读，或者等待timeout了，则调用select的进程会被唤醒，接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户，相应的伪代码如下：

```cpp
for (sk in readfds) {
    sk_event.evt = sk.poll();
    sk_event.sk = sk;
    ret_event_for_process;
}
```

select为每个socket引入了一个poll逻辑，该poll逻辑用于收集socket发生的事件，对于可读事件来说，伪代码如下：

```cpp
poll()
{
    //其他逻辑
    if (recieve queque is not empty)
    {
        sk_event |= POLL_IN；
    }
   //其他逻辑
}
```

通过上面的逻辑过程分析，可以看出select存在三个问题：

+ **被监控的fds需要从用户空间拷贝到内核空间**

+ 为了减少数据拷贝带来的性能损失，内核对**被监控的fds集合大小**做了限制，通过宏控制，为**1024**，不可变

+ 被监控的fds集合中，只要有一个有数据可读，整个socket集合就会被**遍历**一次调用sk的poll函数收集可读事件

  由于当初的需求是朴素，仅仅关心是否有数据可读这样一个事件，当事件通知来的时候，由于数据的到来是异步的，我们不知道事件来的时候，有多少个被监控的socket有数据可读了，于是，只能挨个遍历每个socket来收集可读事件

   

## poll

poll基于select只是对被监控的fds集合大小进行了调整，并没有着手解决性能问题

同样的，poll随着监控的socket集合的增加性能线性下降，poll不适合用于大并发场景

```cpp
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

## epoll

对于IO多路复用，以监控可读事件而言有两件必须的事情：

1. 准备好需要监控的fds集合
2. 探测并返回fds集合中哪些fd可读

而在select和poll模型中，每次调用select和poll都在重复地准备整个需要监控的fds集合。对于频繁调用的select和poll而言，fds集合的变化频率会降低许多，没必要每次都重新准备整个fds集合

故，epoll引入epoll_ctl系统调用，将高频调用的epoll_wait和低频的epoll_ctl隔离开。同时epoll_ctl通过EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL三个操作来分散对需要监控的fds集合的修改，有了变化才变更，将select或poll高频、大块内存拷贝变成epoll_ctl低频、小块内存的拷贝，避免了大量的内存拷贝

同时，对于高频epoll_wait的可读就绪的fd集合返回的拷贝问题，epoll通过内核与用户空间mmap内存映射到同一块内存来解决。mmap将用户空间的一块地址和内核空间的一块地址同时映射到相同的一块物理内存地址（不管是用户空间还是内核空间都是虚拟地址，最终要通过地址映射到物理地址），使得这块物理内存对内核和用户均可见，减少用户态和内核态之间的数据交换

epoll通过epoll_ctl来对监控的fds集合进行增、删、改，则会涉及到fd的快速查找问题，在Linux2.6.8版本之前，epoll是使用hash来组织fds集合的，所以在创建epoll fd时，需要初始化一个hash的大小。而在那以后，改用红黑树来组织监控的fds集合，所以epoll_create(int size)的参数size实际上没有意义了



通过上面的socket的睡眠队列唤醒逻辑可以知道，socket唤醒睡眠在其他睡眠队列的wait_entry用户进程的时候会调用wait_entry的回调函数callback，并且可以在callback中做任何事情。

为了做到只遍历就绪的fd，需要有个地方来组织那些已经就绪的fd。为此，epoll引入了一个中间层，一个双相链表ready_lest，一个单独的睡眠队列single_epoll_wait_list。并且，epoll的进程不需要同时插入到多路复用的socket集合的所有睡眠队列中，相反用户进程只是插入到中间层的epoll的单独睡眠队列中，进程睡眠在epoll的单独队列上，等待事件的发生。同时，引入一个中间的wait_entry_sk，它与某个socket sk相关联，wait_entry_sk睡眠在sk的睡眠队列上，其callback函数逻辑是将当前sk排入到epoll的ready_llist中，并唤醒epoll的single_epoll_wait_list。而single_epoll_wait_list上睡眠的进程的回调函数就是：遍历ready_list上的所有sk，挨个调用sk的poll函数收集事件，然后唤醒进程从epoll_wait返回。于是，可以分为以下几个逻辑：

+ epoll_ctl    EPOLL_CTL_ADD逻辑

  > 1. 构建睡眠 实体wait_entry_sk，将当前socket sk关联给wait_entry_sk，并设置wait_entry_sk的回调函数为epoll_callback_sk
  > 2. 将wait_entry_sk排入当前socket sk的睡眠队列上
  >
  > 回调函数epoll_callback_sk的逻辑如下
  >
  > > 1. 将之前关联的sk排入epoll的ready_list
  > > 2. 唤醒epoll的单独睡眠队列single_epoll_wait_list

+ epoll_wait逻辑

  > 1. 构建睡眠实体wait_entry_proc，将当前process关联给wait_entry_proc，并设置回调函数为epoll_callback_proc
  > 2. 判断epoll的ready_list是否为空，如果为空，则将wait_entry_proc排入epoll的single_epoll_wait_list中，随后进入schedule循环，这会导致调用epoll_wait的process睡眠。
  > 3. wait_entry_proc被事件唤醒或超时醒来，wait_entry_proc将被从single_epoll_wait_list移除掉，然后wait_entry_proc执行回调函数epoll_callback_proc
  >
  > 回调函数epoll_callback_proc的逻辑如下：
  >
  > > 1. 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件，对于监控可读事件而已，ready_list上的每个sk都是有数据可读的，这里的遍历必要的(不同于select/poll的遍历，它不管有没数据可读都需要遍历一些来判断，这样就做了很多无用功。)
  > > 2. 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process

+ epoll唤醒逻辑 整个epoll的协议栈唤醒逻辑如下(对于可读事件而言)：

  > 1. 协议数据包到达网卡并被排入socket sk的接收队列
  > 2. 睡眠在sk的睡眠队列wait_entry被唤醒，wait_entry_sk的回调函数epoll_callback_sk被执行
  > 3. epoll_callback_sk将当前sk插入epoll的ready_list中
  > 4. 唤醒睡眠在epoll的单独睡眠队列single_epoll_wait_list的wait_entry，wait_entry_proc被唤醒执行回调函数epoll_callback_proc
  > 5. 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件
  > 6. 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。

  epoll巧妙的引入一个中间层解决了大量监控socket的无效遍历问题。细心的同学会发现，**epoll在中间层上为每个监控的socket准备了一个单独的回调函数epoll_callback_sk，而对于select/poll，所有的socket都公用一个相同的回调函数**。正是这个单独的回调epoll_callback_sk使得每个socket都能单独处理自身，当自己就绪的时候将自身socket挂入epoll的ready_list。同时，epoll引入了一个睡眠队列single_epoll_wait_list，分割了两类睡眠等待。process不再睡眠在所有的socket的睡眠队列上，而是睡眠在epoll的睡眠队列上，在等待”任意一个socket可读就绪”事件。而中间wait_entry_sk则代替process睡眠在具体的socket上，当socket就绪的时候，它就可以处理自身了。

 **ET vs LT - 总结**

最后总结一下，ET和LT模式下epoll_wait返回的条件

+ ET - 对于读操作

[1] 当接收缓冲buffer内待读数据增加的时候时候(由空变为不空的时候、或者有新的数据进入缓冲buffer)

[2] 调用epoll_ctl(EPOLL_CTL_MOD)来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLIN事件，并且接收缓冲buffer内还有数据没读取。(这里不能是EPOLL_CTL_ADD的原因是，epoll不允许重复ADD的，除非先DEL了，再ADD) 因为epoll_ctl(ADD或MOD)会调用sk的poll逻辑来检查是否有关心的事件，如果有，就会将该sk加入到epoll的ready_list中，下次调用epoll_wait的时候，就会遍历到该sk，然后会重新收集到关心的事件返回。

+ ET - 对于写操作

[1] 发送缓冲buffer内待发送的数据减少的时候(由满状态变为不满状态的时候、或者有部分数据被发出去的时候) [2] 调用epoll_ctl(EPOLL_CTL_MOD)来改变socket_fd的监控事件，也就是重新mod socket_fd的EPOLLOUT事件，并且发送缓冲buffer还没满的时候。

+ LT - 对于读操作 LT就简单多了，唯一的条件就是，接收缓冲buffer内有可读数据的时候
+ LT - 对于写操作 LT就简单多了，唯一的条件就是，发送缓冲buffer还没满的时候

在绝大多少情况下，ET模式并不会比LT模式更为高效，同时，ET模式带来了不好理解的语意，这样容易造成编程上面的复杂逻辑和坑点。因此，建议还是采用LT模式来编程更为舒爽。