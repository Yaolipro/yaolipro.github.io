---
layout: post
title: Linux基础——Select、Poll、Epoll、 异步IO 介绍、IO多路复用
category: ARCHIVES
description: 描述
tags: Linux
---

### 同步IO和异步IO，阻塞IO和非阻塞IO

#### 概念说明
##### 用户空间和内核空间
* 现在操作系统都是采用虚拟存储器，那么对32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间
##### 进程切换
* 为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的
* 从一个进程的运行转到另一个进程上运行，这个过程中经过下面这些变化：
	- 保存处理机上下文，包括程序计数器和其他寄存器。
	- 更新PCB信息。
	- 把进程的PCB移入相应的队列，如就绪、在某事件阻塞等队列。
	- 选择另一个进程执行，并更新其PCB。
	- 更新内存管理的数据结构。
	- 恢复处理机上下文
	- 进程的阻塞
	- 文件描述符
	- 缓存 I/O
##### 进程的阻塞
* 正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的
##### 文件描述符fd
* 文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念
* 文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统
##### 缓存 I/O
* 缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间
* 缓存 I/O 的缺点：
	- 数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

#### IO模式
* 刚才说了，对于一次IO访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。所以说，当一个read操作发生时，它会经历两个阶段：
	- 等待数据准备 (Waiting for the data to be ready)
	- 将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)
* 正式因为这两个阶段，linux系统产生了下面五种网络模式的方案。
	- 阻塞 I/O（blocking IO）
	- 非阻塞 I/O（nonblocking IO）
	- I/O 多路复用（ IO multiplexing）
	- 信号驱动 I/O（ signal driven IO）
	- 异步 I/O（asynchronous IO）
> 注：由于signal driven IO在实际中并不常用，所以我这只提及剩下的四种IO Model。
##### 阻塞 I/O（blocking IO）
* 在linux中，默认情况下所有的socket都是blocking，一个典型的读操作流程大概是这样：
![img](assets/bVm1c3.png)
* 当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据（对于网络IO来说，很多时候数据在一开始还没有到达。比如，还没有收到一个完整的UDP包。这个时候kernel就要等待足够的数据到来）。这个过程需要等待，也就是说数据被拷贝到操作系统内核的缓冲区中是需要一个过程的。而在用户进程这边，整个进程会被阻塞（当然，是进程自己选择的阻塞）。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。
> 所以，blocking IO的特点就是在IO执行的两个阶段都被block了。
##### 非阻塞 I/O（nonblocking IO）
* linux下，可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子：
![img](assets/bVm1c4.png)
* 当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。
> 所以，nonblocking IO的特点是用户进程需要不断的主动询问kernel数据好了没有。
##### I/O 多路复用（ IO multiplexing）
* IO multiplexing就是我们说的select，poll，epoll，有些地方也称这种IO方式为event driven IO。select/epoll的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select，poll，epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。
![img](assets/bVm1c5.png)
* 当用户进程调用了select，那么整个进程会被block，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程。
> 所以，I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回。

* 这个图和blocking IO的图其实并没有太大的不同，事实上，还更差一些。因为这里需要使用两个system call (select 和 recvfrom)，而blocking IO只调用了一个system call (recvfrom)。但是，用select的优势在于它可以同时处理多个connection。
* 所以，如果处理的连接数不是很高的话，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。）
* 在IO multiplexing Model中，实际中，对于每一个socket，一般都设置成为non-blocking，但是，如上图所示，整个用户的process其实是一直被block的。只不过process是被select这个函数block，而不是被socket IO给block。
##### 异步 I/O（asynchronous IO）
* Linux下的asynchronous IO其实用得很少。先看一下它的流程：
![img](assets/bVm1c8.png)
* 用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了

#### 总结
##### blocking和non-blocking的区别
* 调用blocking IO会一直block住对应的进程直到操作完成，而non-blocking IO在kernel还准备数据的情况下会立刻返回。
##### 同步(synchronous) IO和异步(asynchronous) IO的区别
* 在说明synchronous IO和asynchronous IO的区别之前，需要先给出两者的定义。POSIX的定义是这样子的：
	- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;
	- An asynchronous I/O operation does not cause the requesting process to be blocked;
* 两者的区别就在于synchronous IO做“IO operation”的时候会将process阻塞。按照这个定义，之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO。
* 有人会说，non-blocking IO并没有被block啊。这里有个非常“狡猾”的地方，定义中所指的“IO operation”是指真实的IO操作，就是例子中的recvfrom这个system call。non-blocking IO在执行recvfrom这个system call的时候，如果kernel的数据没有准备好，这时候不会block进程。但是，当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内，进程是被block的。
* 而asynchronous IO则不一样，当进程发起IO 操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block。
* 各个IO Model的比较如图所示：
![img](assets/bVm1c9.png)
* 通过上面的图片，可以发现non-blocking IO和asynchronous IO的区别还是很明显的。在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程去主动的check，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。而asynchronous IO则完全不同。它就像是用户进程将整个IO操作交给了他人（kernel）完成，然后他人做完后发信号通知。在此期间，用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据。
##### 二、I/O 多路复用之select、poll、epoll详解
* select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。sellect、poll、epoll三者的具体区别如下：
* select 
	- select最早于1983年出现在4.2BSD中，它通过一个select()系统调用来监视多个文件描述符的数组，当select()返回后，该数组中就绪的文件描述符便会被内核修改标志位，使得进程可以获得这些文件描述符从而进行后续的读写操作
	- select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点，事实上从现在看来，这也是它所剩不多的优点之一
	- select的一个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，不过可以通过修改宏定义甚至重新编译内核的方式提升这一限制
	- 另外，select()所维护的存储大量文件描述符的数据结构，随着文件描述符数量的增大，其复制的开销也线性增长。同时，由于网络响应时间的延迟使得大量TCP连接处于非活跃状态，但调用select()会对所有socket进行一次线性扫描，所以这也浪费了一定的开销
* poll 
	- poll在1986年诞生于System V Release 3，它和select在本质上没有多大差别，但是poll没有最大文件描述符数量的限制。
	- poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。
	- 另外，select()和poll()将就绪的文件描述符告诉进程后，如果进程没有对其进行IO操作，那么下次调用select()和poll()的时候将再次报告这些文件描述符，所以它们一般不会丢失就绪的消息，这种方式称为水平触发（Level Triggered）。
* epoll 
	- 直到Linux2.6才出现了由内核直接支持的实现方法，那就是epoll，它几乎具备了之前所说的一切优点，被公认为Linux2.6下性能最好的多路I/O就绪通知方法。
	- epoll可以同时支持水平触发和边缘触发（Edge Triggered，只告诉进程哪些文件描述符刚刚变为就绪状态，它只说一遍，如果我们没有采取行动，那么它将不会再次告知，这种方式称为边缘触发），理论上边缘触发的性能要更高一些，但是代码实现相当复杂。
	- epoll同样只告知那些就绪的文件描述符，而且当我们调用epoll_wait()获得就绪文件描述符时，返回的不是实际的描述符，而是一个代表就绪描述符数量的值，你只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可，这里也使用了内存映射（mmap）技术，这样便彻底省掉了这些文件描述符在系统调用时复制的开销。
	- 另一个本质的改进在于epoll采用基于事件的就绪通知方式。在select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。
* Python select 
	- Python中的select模块专注于I/O多路复用，提供了select  poll  epoll三个方法(其中后两个在Linux中可用，windows仅支持select,官方解释：On Windows, only sockets are supported; on Unix, all file descriptors)，另外也提供了kqueue方法(freeBSD系统)
	- Python的select()方法直接调用操作系统的IO接口，它监控sockets，open files， and pipes(所有带fileno()方法的文件句柄)何时变成readable 和writeable, 或者通信错误（异常），select()使得同时监控多个连接变的简单，并且这比写一个长循环来等待和监控多客户端连接要高效，因为select直接通过操作系统提供的C的网络接口进行操作，而不是通过Python的解释器。
* select方法
* 当我们使用select方法时：
	- 进程指定内核监听哪些文件描述符(最多监听1024个fd)的哪些事件，当没有文件描述符事件发生时，进程被阻塞；当一个或者多个文件描述符事件发生时，进程被唤醒。
	- 具体过程大致如下：
		- 1、调用select()方法，上下文切换转换为内核态
    - 2、将fd从用户空间复制到内核空间
    - 3、内核遍历所有fd，查看其对应事件是否发生
    - 4、如果没发生，将进程阻塞，当设备驱动产生中断或者timeout时间后，将进程唤醒，再次进行遍历
    - 5、返回遍历后的fd
    - 6、将fd从内核空间复制到用户空间
* 使用方法：fd_r_list, fd_w_list, fd_e_list = select.select(rlist, wlist, xlist, [timeout]) 
* 参数列表：
	- rlist: wait until ready for reading
	- wlist: wait until ready for writing
	- xlist: wait for an “exceptional condition”
	- timeout: 超时时间
* 返回三个值：
	- select方法用来监视文件描述符(当文件描述符条件不满足时，select会阻塞)，当某个文件描述符状态改变后，会返回三个列表
    - 1、当参数1 序列中的fd满足“可读”条件时，则获取发生变化的fd并添加到fd_r_list中
    - 2、当参数2 序列中含有fd时，则将该序列中所有的fd添加到 fd_w_list中
    - 3、当参数3 序列中的fd发生错误时，则将该发生错误的fd添加到 fd_e_list中
    - 4、当超时时间为空，则select会一直阻塞，直到监听的句柄发生变化
	- 当超时时间 ＝ n(正整数)时，那么如果监听的句柄均无任何变化，则select会阻塞n秒，之后返回三个空列表，如果监听的句柄有变化，则直接执行。
* demo：利用select方法实现一个高并发的socket服务端 
```
# -*- coding: utf-8 -*-

import select
import socket
server = socket.socket()
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.setblocking(False)       # 设置为非阻塞状态
server.bind(("0.0.0.0", 5000))
server.listen(5)
inputs, outputs = [], []
inputs.append(server)           # 将sever作为一个fd也放入select并进行检测
while True:
    readable, writable, exceptionable = select.select(inputs, outputs, inputs)
    for r in readable:
        if r is server:         # 判断r是不是server本身，如果是本身则代表来了新的连接
            conn, addr = r.accept()
            print("客户端：{}已经连接".format(addr))
            inputs.append(conn)
        else:                   # 否则代表readable里是已经建立的连接
            data = r.recv(1024)
            if data:
                print(data)
                r.send(data)
            else:
                print("客户端已经断开")
                inputs.remove(r)
        for e in exceptionable:
            print("客户端出错！")
            inputs.remove(e)
```
客户端
```
# -*- coding: utf-8 -*-

import socket
client = socket.socket()            # 生成socket实例
client.connect(('127.0.0.1', 5000)) # 连接服务器
while True:                         # while循环用于和客户端一直交互
    data = input(">>>>")
    client.send(data.encode())
    data = client.recv(1024)
```
select文艺版
```
# -*- coding: utf-8 -*-

import select
import socket
import sys
import queue

server = socket.socket()
server.setblocking(0)
server_addr = ('localhost', 10000)
print('starting up on %s port %s' % server_addr)
server.bind(server_addr)
server.listen(5)
inputs = [server, ]  # 自己也要监测呀,因为server本身也是个fd
outputs = []
message_queues = {}
while True:
    print("waiting for next event...")
    readable, writeable, exeptional = select.select(inputs, outputs, inputs)  # 如果没有任何fd就绪,那程序就会一直阻塞在这里
    for s in readable:      # 每个s就是一个socket
        if s is server:     # 别忘记,上面我们server自己也当做一个fd放在了inputs列表里,传给了select,如果这个s是server,代表server这个fd就绪了,
            # 就是有活动了, 什么情况下它才有活动? 当然 是有新连接进来的时候 呀
            # 新连接进来了,接受这个连接
            conn, client_addr = s.accept()
            print("new connection from", client_addr)
            conn.setblocking(0)
            # 为了不阻塞整个程序,我们不会立刻在这里开始接收客户端发来的数据, 把它放到inputs里, 下一次loop时,这个新连接
            inputs.append(conn)
            #就会被交给select去监听,如果这个连接的客户端发来了数据 ,那这个连接的fd在server端就会变成就续的,select就会把这个连接返回,返回到
            # readable 列表里,然后你就可以loop readable列表,取出这个连接,开始接收数据了, 下面就是这么干 的
            # 接收到客户端的数据后,不立刻返回 ,暂存在队列里,以后发送
            message_queues[conn] = queue.Queue()
        else:       # s不是server的话,那就只能是一个 与客户端建立的连接的fd了
            # 客户端的数据过来了,在这接收
            data = s.recv(1024)
            if data:
                print("收到来自[%s]的数据:" % s.getpeername()[0], data)
                message_queues[s].put(data) # 收到的数据先放到queue里,一会返回给客户端
                if s not in outputs:
                    outputs.append(s)       # 为了不影响处理与其它客户端的连接 , 这里不立刻返回数据给客户端
            else:  # 如果收不到data代表什么呢? 代表客户端断开了呀
                print("客户端断开了", s)
                if s in outputs:
                    outputs.remove(s)   # 清理已断开的连接
                inputs.remove(s)        # 清理已断开的连接
                del message_queues[s]   # 清理已断开的连接
    for s in writeable:
        try:
            next_msg = message_queues[s].get_nowait()
        except queue.Empty:
            print("client [%s]" % s.getpeername()[0], "queue is empty..")
            outputs.remove(s)
        else:
            print("sending msg to [%s]" % s.getpeername()[0], next_msg)
            s.send(next_msg.upper())
    for s in exeptional:
        print("handling exception for ", s.getpeername())
        inputs.remove(s)
        if s in outputs:
            outputs.remove(s)
        s.close()
        del message_queues[s]
```
* 在服务端我们可以看到，我们需要不停的调用select， 这就意味着：
* 当文件描述符过多时，文件描述符在用户空间与内核空间进行copy会很费时
* 当文件描述符过多时，内核对文件描述符的遍历也很浪费时间
* select默认最大仅仅支持1024个文件描述符（在liunx上可以通过修改打开文件数来修改这个值）
* poll方法
	- poll与select相差不大，相比于select而言，内核监测的文件描述不受限制。
* epoll方法
* epoll很好的改进了select：
	- epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时，会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。
	- epoll会在epoll_ctl时把指定的fd遍历一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd
	- epoll对文件描述符没有额外限制
* 具体方法：
```
select.epoll(sizehint=-1, flags=0):创建epoll对象
epoll.close():关闭epoll对象的文件描述符
epoll.closed():检测epoll对象是否关闭
epoll.fileno():返回epoll对象的文件描述符
epoll.fromfd(fd):根据指定的fd创建epoll对象
epoll.register(fd[, eventmask]):向epoll对象中注册fd和对应的事件
epoll.modify(fd, eventmask):修改fd的事件
epoll.unregister(fd):Remove a registered file descriptor from the epoll object.取消注册
epoll.poll(timeout=-1, maxevents=-1):Wait for events. timeout in seconds (float)阻塞，直到注册的fd事件发生,会返回一个dict，格式为：{(fd1,event1),(fd2,event2),……(fdn,eventn)}
```
* 事件：
```
EPOLLIN    Available for read 可读   状态符为1
EPOLLOUT    Available for write 可写  状态符为4
EPOLLPRI    Urgent data for read
EPOLLERR    Error condition happened on the assoc. fd 发生错误 状态符为8
EPOLLHUP    Hang up happened on the assoc. fd 挂起状态
EPOLLET    Set Edge Trigger behavior, the default is Level Trigger behavior 默认为水平触发，设置该事件后则边缘触发
EPOLLONESHOT    Set one-shot behavior. After one event is pulled out, the fd is internally disabled
EPOLLRDNORM    Equivalent to EPOLLIN
EPOLLRDBAND    Priority data band can be read.
EPOLLWRNORM    Equivalent to EPOLLOUT
EPOLLWRBAND    Priority data may be written.
EPOLLMSG    Ignored.
```
* 关于水平触发和边缘触发：
	- Level_triggered(水平触发，有时也称条件触发)：当被监控的文件描述符上有可读写事件发生时，epoll.poll()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll.poll()时，它还会通知你在上没读写完的文件描述符上继续读写，当然如果你一直不去读写，它会一直通知你，如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率！！！ 优点很明显：稳定可靠
	- Edge_triggered(边缘触发，有时也称状态触发)：当被监控的文件描述符上有可读写事件发生时，epoll.poll()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll.poll()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你，这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。缺点：某些条件下不可靠
* epoll实例：
```
# -*- coding: utf-8 -*-

import socket
import select

s = socket.socket()
s.bind(('127.0.0.1', 8888))
s.listen(5)
epoll_obj = select.epoll()
epoll_obj.register(s, select.EPOLLIN)
connections = {}
while True:
    events = epoll_obj.poll()
    for fd, event in events:
        print(fd, event)
        if fd == s.fileno():
            conn, addr = s.accept()
            connections[conn.fileno()] = conn
            epoll_obj.register(conn, select.EPOLLIN)
            msg = conn.recv(200)
            conn.sendall('ok'.encode())
        else:
            try:
                fd_obj = connections[fd]
                msg = fd_obj.recv(200)
                fd_obj.sendall('ok'.encode())
            except BrokenPipeError:
                epoll_obj.unregister(fd)
                connections[fd].close()
                del connections[fd]

s.close()
epoll_obj.close()
```
```
# -*- coding: utf-8 -*-
import socket

flag = 1
s = socket.socket()
s.connect(('127.0.0.1', 8888))
while flag:
    input_msg = input('input>>>')
    if input_msg == '0':
        break
    s.sendall(input_msg.encode())
    msg = s.recv(1024)
    print(msg.decode())

s.close()
```
* selectors
	- python3.4新增selectors模块，封装了select，高层次、高效率的I/O多路复用，它具有根据操作系统平台选出最佳的IO多路机制，比如在win的系统上他默认的是select模式而在linux上它默认的epoll。
```
# -*- coding: utf-8 -*-

import selectors
import socket

sel = selectors.DefaultSelector()

def accept(sock, mask):
    conn, addr = sock.accept()  # Should be ready
    print('accepted', conn, 'from', addr)
    conn.setblocking(False)
    sel.register(conn, selectors.EVENT_READ, read)

def read(conn, mask):
    data = conn.recv(1000)      # Should be ready
    if data:
        print('echoing', repr(data), 'to', conn)
        conn.send(data)         # Hope it won't block
    else:
        print('closing', conn)
        sel.unregister(conn)
        conn.close()

sock = socket.socket()
sock.bind(('localhost', 10000))
sock.listen(100)
sock.setblocking(False)
sel.register(sock, selectors.EVENT_READ, accept)

while True:
    events = sel.select()
    for key, mask in events:
        callback = key.data
        callback(key.fileobj, mask)
```

### 附录
* https://www.cnblogs.com/wdliu/p/6898382.html

网络处理整理
* Python3 Socket文档：https://python3-cookbook.readthedocs.io/zh_CN/latest/c12/p01_start_stop_thread.html
* Socket和SocketServer源码：
	- https://www.jianshu.com/p/f1ca9406fd96
	- https://docs.python.org/3/library/socketserver.html
	- https://amylewis.github.io/2018/07/16/Python-SocketServer/index.html
* BSD Socket：https://docs.huihoo.com/gnu/linux/socket.html


https://www.jianshu.com/p/7fe5945b80e7
https://www.jianshu.com/p/7fe5945b80e7