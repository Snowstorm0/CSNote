<!-- GFM-TOC -->

* [一、概念说明](#一概念说明)
  
    * [用户空间与内核空间](#用户空间与内核空间)
    * [进程切换](#进程切换)
    * [进程的阻塞](#进程的阻塞)
    * [文件描述符fd](#文件描述符fd)
    * [缓存I/O](#缓存io)
    
* [二、I/O 模型](#二io-模型)
  
    * [阻塞式 I/O](#阻塞式-io)
    * [非阻塞式 I/O](#非阻塞式-io)
    * [I/O 复用](#io-复用)
    * [信号驱动 I/O](#信号驱动-io)
    * [异步 I/O](#异步-io)
    * [五大 I/O 模型比较](#五大-io-模型比较)
    
* [三、I/O 复用](#三io-复用)

* [select](#select)

* [poll](#poll)
  
    * [select和poll比较](#select和poll比较)
    
* [epoll](#epoll)
  
    * [相关函数](#相关函数)
    
    * [模型的特点](#模型的特点)
    * [工作模式](#工作模式)
    
* [区别与应用](#区别与应用)

* [send、recv、accept函数](#sendrecvaccept函数)

  <!-- GFM-TOC -->

# 套接字(Socket)

可以将套接字看作**不同主机间的进程进行双间通信的端点**，它构成了单个主机内及整个网络间的编程界面。



# 一、概念说明

在进行解释之前，首先要说明几个概念：

- 用户空间和内核空间
- 进程切换
- 进程的阻塞
- 文件描述符
- 缓存 I/O

## 用户空间与内核空间

现在操作系统都是采用虚拟存储器，那么对32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间。

## 进程切换

为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。

从一个进程的运行转到另一个进程上运行，这个过程中经过下面这些变化：
\1. 保存处理机上下文，包括程序计数器和其他寄存器。
\2. 更新PCB信息。
\3. 把进程的PCB移入相应的队列，如就绪、在某事件阻塞等队列。
\4. 选择另一个进程执行，并更新其PCB。
\5. 更新内存管理的数据结构。
\6. 恢复处理机上下文。

注：**总而言之就是很耗资源**，具体的可以参考这篇文章：[进程切换](http://guojing.me/linux-kernel-architecture/posts/process-switch/)

## 进程的阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。`当进程进入阻塞状态，是不占用CPU资源的`。

## 文件描述符fd

文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

## 缓存I/O

缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

**缓存 I/O 的缺点：**
数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

# 二、I/O 模型

一个输入操作通常包括两个阶段：

- 等待数据准备好
- 从内核向进程复制数据

对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所等待数据到达时，它被**复制到内核**中的某个缓冲区。第二步就是把数据**从内核缓冲区复制到应用进程缓冲区**。

Unix 有五种 I/O 模型：

- 阻塞式 I/O
- 非阻塞式 I/O
- I/O 复用（select 和 poll）
- 信号驱动式 I/O（SIGIO）
- 异步 I/O（AIO）

## 阻塞式 I/O

**应用进程被阻塞**，直到数据从**内核缓冲区复制到应用进程缓冲区**中才返回。

应该注意到，在阻塞的过程中，**其它应用进程还可以执行**，因此阻塞不意味着整个操作系统都被阻塞。因为其它应用进程还可以执行，所以不消耗 CPU 时间，这种模型的 CPU 利用率会比较高。

下图中，**recvfrom() 用于接收 Socket 传来的数据**，并复制到应用进程的缓冲区 buf 中。这里把 recvfrom() 当成系统调用。

```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492928416812_4.png"/> </div><br>

## 非阻塞式 I/O

应用进程执行系统调用之后，内核返回一个错误码。应用进程可以继续执行，但是需要**不断的执行系统调用来获知 I/O 是否完成**，这种方式称为**轮询（polling）**。

由于 CPU 要处理更多的系统调用，因此这种模型的 CPU 利用率比较低。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492929000361_5.png"/> </div><br>

## I/O 复用

使用 **select** 或者 **poll** 等待数据，并且可以等待**多个套接字(Socket)中的任何一个变为可读**。这一过程会被**阻塞**，当某一个套接字可读时返回，之后再使用 **recvfrom** 把数据从内核复制到进程中。

它可以让**单个进程具有处理多个 I/O 事件的能力**。又被称为 Event Driven I/O，即**事件驱动 I/O**。

如果一个 Web 服务器没有 I/O 复用，那么每一个 **Socket 连接**都需要创建**一个线程**去处理。如果同时有几万个连接，那么就需要创建相同数量的线程。相比于多进程和多线程技术，I/O 复用不需要进程线程创建和切换的开销，系统开销更小。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492929444818_6.png"/> </div><br>

## 信号驱动 I/O

应用进程使用 **sigaction** 系统调用，内核立即返回，应用进程可以继续执行，也就是说等待数据阶段应用进程是**非阻塞的**。内核在数据到达时向应用进程发送 **SIGIO (sign IO)信号**，应用**进程**收到之后在信号处理程序中调用 **recvfrom** 将数据从内核复制到应用进程中。

相比于非阻塞式 I/O 的轮询方式，信号驱动 I/O 的 CPU 利用率更高。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492929553651_7.png"/> </div><br>

## 异步 I/O

应用进程执行 **aio_read** 系统调用会立即返回，应用进程可以继续执行，**非阻塞**，内核会在所有操作完成之后向应用进程发送信号。

异步 I/O 与信号驱动 I/O 的区别在于，**异步 I/O 的信号是通知应用进程 I/O 完成**，而**信号驱动 I/O 的信号是通知应用进程可以开始 I/O**。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492930243286_8.png"/> </div><br>

## 五大 I/O 模型比较

- 同步 I/O：第二阶段（将数据从内核缓冲区复制到应用进程缓冲区），应用进程会阻塞。
- 异步 I/O：第二阶段应用进程不会阻塞。

同步 I/O 包括阻塞式 I/O、非阻塞式 I/O、I/O 复用和信号驱动 I/O ，它们的主要区别在第一个阶段。

阻塞式 I/O、I/O 复用在第一阶段（等待数据准备好）会阻塞。

非阻塞式 I/O 、信号驱动 I/O 和异步 I/O 在第一阶段不会阻塞。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492928105791_3.png"/> </div><br>

# 三、I/O 复用

**select/poll/epoll** 都是 I/O 多路复用的具体实现。它的基本原理就是 select，poll，epoll 这个 function 会不断的**轮询**所负责的**所有** socket，当某个 socket 有数据到达了，就通知用户进程。I/O 复用在等待数据准备阶段会阻塞。

**文件描述符：fd**，file descriptor。

# select

函数原型：

```c
int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, struct timeval *timeout);
```

select 允许应用**程序监视一组fd**，等待一个或者多个fd成为就绪状态，从而完成 I/O 操作。

参数：

1. **maxfd**：是需要监视的最大的fd值+1（0 到 maxfd-1）；
2. **fd_set** ：描述符集合，使用数组实现，数组大小使用 FD_SETSIZE 定义。有三种类型的描述符类型：**readset、writeset、exceptset**，分别对应**读、写、异常**条件的fd_set。若对其中任何参数条件不感兴趣，则可将其设为NULL。
3. **timeout** ：为超时参数，调用 select 会一直阻塞直到有描述符的事件到达或者等待的时间超过 timeout。timeout参数的三种可能：（1）NULL：永远等待下去，仅在有fd就绪时才返回；（2）正常时间：在不超过timeout设置的时间内，在有fd就绪时返回；（3）0：检查fd的每位后立即返回(轮询)。
4. **成功**调用返回结果**大于 0**，**出错**返回结果为 **-1**，**超时**返回结果为 **0**。

timeval 结构体：

```c
struct timeval{
    long tv_sec;    // 秒    
    long tv_usec;   // 微秒 
}
```

## 特点

（1）将文件描述符fd加入到**描述符集合fd_set （B）**中，还需要用一个**预先保存fd的数组（A）**，将文件描述符fd保存起来。原因有两个：

- 在执行select之后，集合B已经被修改为都是**有事件发生的fd位**，此时数组A中的fd可以用`FD_ISSET`来**轮询**，判断是否在集合B中；
- select返回后会把以前加入的但并**无事件发生的fd的位清空**，下一次开始 select 前要重新从数组A中取得文件描述符逐个加入到集合B中，扫描数组A的同时取得文件描述符的最大值 maxfd ，用于 select 的第一个参数。

（2）存在的问题：

- 有最大文件数限制；
- 每次调用 select 前都要重新初始化 fd_set，将 fd 从用户态**拷贝**到内核态；
- 同时每次调用 select 都需要在内核**轮询**传递进来的所有fd；
- 需要**轮询**，实现数组A中所有fd与集合B进行`FD_ISSET`操作，当fd很多时效率很低。

```c
void FD_ZERO(fd_set *fdset);         //清空集合
void FD_SET(int fd, fd_set *fdset);  //将一个给定的文件描述符加入集合之中
void FD_CLR(int fd, fd_set *fdset);  //将一个给定的文件描述符从集合中删除
int FD_ISSET(int fd, fd_set *fdset); // 检查集合中指定的文件描述符是否可以读写 
```

# poll

函数原型：

```c
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```

poll 的功能与 select 类似，也是等待一组fd中的一个成为就绪状态。通过一个**可变长度的数组**解决了 select 文件描述符受限的问题。数组中元素是**结构体**，该结构体保存fd的信息，每增加一个fd就向数组中加入一个结构体。因为poll将 events 和 revents 分开， events 的事件类型是不会变的，所以初始化一次就可以。poll 对轮寻排查的问题未解决。

参数：

1. **fds：**一个结构数组，保存**描述符**的信息。
2. **nfds：**要监视的描述符的**数目**。
3. **timeout：**用毫秒表示的时间，是指定poll在返回前没有接收事件时应该**等待**的时间。
4. **成功**调用返回结果**大于 0**，**出错**返回结果为 **-1**，**超时**返回结果为 **0**。

poll 中的描述符是 **pollfd** 类型的数组，pollfd 的定义如下：

```c
struct pollfd {
    int   fd;       // 要监听的文件描述符
    short events;   // 需要监听的事件（读、写、异常）
    short revents;  // 需要返回的事件，调用poll后的结果事件
};
```

## 特点

（1）poll没有最大连接数的限制，原因是它是基于**链表**来存储的。在select中，被监听集合和返回集合是一个集合，在poll中将**监听**和**返回**的事件都在结构体中不同的成员中，它们互不干扰，poll 中将有事件发生的fd设置为其结构体的revents，不需要向select一样用一个数组存储原来的fd。

（2）poll函数中fds数组中元素是 **pollfd 结构体**，该结构体保存描述符的信息，每增加一个文件描述符就向数组中结构体加入一个描述符，**结构体只需要拷贝一次到内核态**。poll解决了select重复初始化的问题。但轮寻检查事件发生的问题仍然未解决。

（3）与select一样，poll返回后，需要**轮询**每个pollfd结构体的revents来获取就绪的描述符。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被**整体复制**于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。 



## select和poll比较

### 1. 功能

select 和 poll 的功能基本相同，不过在一些实现细节上有所不同。

- **select 会修改描述符**，而 poll 不会；
- **select** 的描述符类型使用数组实现，**FD_SETSIZE 大小默认为 1024**，如果要监听更多描述符的话，需要修改 FD_SETSIZE 之后重新编译；而 poll 没有描述符数量的限制；
- poll 提供了更多的**事件类型**，并且对描述符的重复利用上比 select 高。
- 如果一个线程对某个描述符调用了 select 或者 poll，另一个线程关闭了该描述符，会导致调用结果不确定。

### 2. 速度

select 和 poll 速度都比较慢，**每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区**。

### 3. 可移植性

几乎所有的系统都支持 select，但是只有比较新的系统支持 poll。

# epoll

epoll没有描述符限制。epoll使用一个epoll句柄管理多个fd，将用户关心的fd的事件存放到内核的一个**事件表**中，这样在用户空间和内核空间的只需拷贝一次。

函数原型：

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

## 相关函数

### 1、epoll_create

```c++
int epoll_create(int size);
```

会在内核的高速cache区中建立一颗**红黑树**以及**就绪链表**(该链表存储已经就绪的fd)。

- **size**为监听的数目一共有多大，size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
- **返回值：**返回一个文件描述符fd，可以理解为指向内核中的一颗**红黑树**的树根，size就是创建红黑树的大小。

### 2、epoll_ctl

```c++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
```

**epoll_ctl**() 用于向内核**注册新的fd**或者是**改变某个fd的状态**。已注册的描述符在内核中会被维护在一棵**红黑树**上。在执行`epoll_ctl`的`add`操作时，不仅将fd符添加红黑树上，而且也注册了回调函数，内核在检测到某 fd 可读/可写 时会调用回调函数将fd放在**就绪链表**中。

- **epfd** 是epoll_create()的返回值的描述符；
- **op** 表示动作，用三个宏`EPOLL_CTL_ADD`、`EPOLL_CTL_MOD`、`EPOLL_CTL_DEL`来表示，控制某个epoll监听的文件描述符上的事件：**添加、修改、删除**。相当于在红黑树上操作。
- **fd** 是需要监听事件的文件描述符，
- **event** 是告诉内核需要**监听什么事件**。

- **返回值：**成功返回0，不成功返回1。

struct epoll_event结构如下：

```c++
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;
```

### 3、epoll_wait

```c++
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

进程调用 **epoll_wait**() 便可以得到**事件完成的描述符**。最多返回maxevents个事件。`epoll_wait`只用观察**就绪链表**中有无数据即可，最后将链表的数据返回给数组并返回就绪的数量。内核将就绪的文件描述符放在传入的数组中，所以只用遍历依次处理即可。

参数：

- **fd** 是epoll_create返回的文件描述符；
- **events** 是一个数组，传入传出参数；
- **maxevents** ：events 数组成员的个数，这个 maxevents 的值不能大于创建epoll_create()时的size；
- **timeout** 设置超时时间(-1 阻塞，0 立即返回，>0 指定毫秒)。
- **返回值：** 成功返回有多少文件描述符就绪，时间到时返回0，出错返回-1。返回的有事件发生的描述符都在 events 数组中，数组中实际存放的成员个数是函数的返回值个。

## 模型的特点

（1）epoll 比 select 和 poll 更加灵活而且**没有描述符数量限制**。

（2）基于事件就绪通知方式：一旦被监听的某个文件描述符就绪，内核会采用类似于callback的回调机制，迅速激活这个文件描述符。不会像select/poll中轮询检测每个描述符是否就绪。

（3）当文件描述符就绪，就会从就绪链表放到一个**数组**中，这样调用 epoll_wait 获取就绪文件描述符的时候，只要取数组中的返回的个数个元素即可，不需要轮询检测。

（4）内存拷贝是利用**mmap()文件映射**内存的方式加速与内核空间的消息传递，减少复制开销。（内核与用户空间**共享一块内存**）

（5）epoll 对多线程编程更有友好，一个线程调用了 epoll_wait() 另一个线程关闭了同一个描述符也不会产生像 select 和 poll 的不确定情况。




## 工作模式

epoll 的描述符事件有两种触发模式：**LT（level trigger，水平触发）**和 **ET（edge trigger，边缘触发）**。

### 1. LT 模式

当 epoll_wait() 检测到**描述符事件**到达时，将此事件通知进程，**进程可以不立即处理该事件**，下次调用 epoll_wait() 会**再次通知**进程。是默认的一种模式，并且同时支持 Blocking （阻塞）和 No-Blocking（非阻塞）。

### 2. ET 模式

和 LT 模式不同的是，通**知之后进程必须立即处理事件**，下次再调用 epoll_wait() 时不会再得到事件到达的通知。

很大程度上**减少了 epoll 事件被重复触发的次数**，因此效率要比 LT 模式高。只支持 No-Blocking，以避免由于一个文件句柄的 阻塞读/阻塞写 操作把处理多个文件描述符的任务饿死。

二者的主要差异在于level-trigger模式下只要某个socket处于readable/writable状态，无论什么时候进行epoll_wait 都会返回该 socket；而edge-trigger模式下只有某个socket从unreadable变为readable或从unwritable 变为writable时，epoll_wait才会返回该socket。



# 区别与应用

## 区别

1. select和poll的动作基本一致，只是poll采用链表来进行文件描述符的存储，而select采用fd标注位来存放，所以select会受到最大连接数的限制，而poll不会。
2. select、poll、epoll虽然都会返回就绪的文件描述符数量。但是select和poll并不会明确指出是哪些文件描述符就绪，而epoll会。造成的区别就是,系统调用返回后，调用select和poll的程序需要遍历监听的整个文件描述符找到是谁处于就绪，而epoll则直接处理就行了。
3. select、poll都需要将有关文件描述符的数据结构拷贝进内核，最后再拷贝出来。而epoll创建的有关文件描述符的数据结构本身就存于内核态中，系统调用返回时也采用mmap共享存储区，需要拷贝的次数大大减少。
4. select、poll采用轮询的方式来检查文件描述符是否处于就绪态，而epoll采用回调机制。造成的结果就是,随着fd的增加，select和poll的效率会线性降低，而epoll不会受到太大影响,除非活跃的socket很多。

## 应用

### 1. select

select 的 **timeout 参数精度为微秒**，而 poll 和 epoll 为毫秒，因此 select 更加适用于**实时性要求比较高**的场景，比如核反应堆的控制。

select 可移植性更好，几乎被所有主流平台所支持。

### 2. poll

poll **没有最大描述符数量**的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

### 3. epoll

只需要运行在 Linux 平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接。

需要同时监控小于 1000 个描述符，就没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势。

需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 epoll。因为 **epoll 中的所有描述符都存储在内核中**，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。并且 epoll 的描述符存储在内核，不容易调试。



# send、recv、accept函数

`send()` 函数用来向 TCP 连接的另一端发送数据。客户程序一般用 send 函数向服务器发送请求，而服务器则通常用send函数来向客户程序发送应答,send的作用是将要发送的数据拷贝到缓冲区，协议负责传输。

`recv()` 函数用来从 TCP 连接的另一端接收数据，当应用程序调用 recv 函数时，recv 先等待 s 的发送缓冲中的数据被协议传送完毕，然后从缓冲区中读取接收到的内容给应用层。

`accept()` 函数用了接收一个连接，内核维护了半连接队列和一个已完成连接队列，当队列为空的时候，accept 函数阻塞，不为空的时候 accept 函数从上边取下来一个已完成连接，返回一个文件描述符。







# 参考资料

- Stevens W R, Fenner B, Rudoff A M. UNIX network programming[M]. Addison-Wesley Professional, 2004.
- http://man7.org/linux/man-pages/man2/select.2.html
- http://man7.org/linux/man-pages/man2/poll.2.html
- [Boost application performance using asynchronous I/O](https://www.ibm.com/developerworks/linux/library/l-async/)
- [Synchronous and Asynchronous I/O](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365683(v=vs.85).aspx)
- [Linux IO 模式及 select、poll、epoll 详解](https://segmentfault.com/a/1190000003063859)
- [poll vs select vs event-based](https://daniel.haxx.se/docs/poll-vs-select.html)
- [select / poll / epoll: practical difference for system architects](http://www.ulduzsoft.com/2014/01/select-poll-epoll-practical-difference-for-system-architects/)
- [Browse the source code of userspace/glibc/sysdeps/unix/sysv/linux/ online](https://code.woboq.org/userspace/glibc/sysdeps/unix/sysv/linux/)





