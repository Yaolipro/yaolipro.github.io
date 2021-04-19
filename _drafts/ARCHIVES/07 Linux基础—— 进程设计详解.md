[TOC]


## Linux进程基础
### 进程概述
* 计算机实际上可以做的事情实质上非常简单，比如计算两个数的和，再比如在内存中寻找到某个地址等等。这些最基础的计算机动作被称为指令 (instruction)。所谓的程序(program)，就是这样一系列指令的所构成的集合。通过程序，我们可以让计算机完成复杂的操作。程序大多数时候被存储为可执行的文件。这样一个可执行文件就像是一个菜谱，计算机可以按照菜谱作出可口的饭菜。
* 那么，程序和进程(process)的区别又是什么呢?
* 进程是程序的一个具体实现。只有食谱没什么用，我们总要按照食谱的指点真正一步步实行，才能做出菜肴。进程是执行程序的过程，类似于按照食谱，真正去做菜的过程。同一个程序可以执行多次，每次都可以在内存中开辟独立的空间来装载，从而产生多个进程。不同的进程还可以拥有各自独立的IO接口。
* 操作系统的一个重要功能就是为进程提供方便，比如说为进程分配内存空间，管理进程的相关信息等等，就好像是为我们准备好了一个精美的厨房。

### 看一眼进程
首先，我们可以使用$ps命令来查询正在运行的进程，比如$ps -eo pid,comm,cmd，下图为执行结果:
(-e表示列出全部进程，-o pid,comm,cmd表示我们需要PID，COMMAND，CMD信息)

```
	PID COMMAND         CMD
    1 systemd         /sbin/init nospectre_v2 nopti
    2 kthreadd        [kthreadd]
    3 ksoftirqd/0     [ksoftirqd/0]
    5 kworker/0:0H    [kworker/0:0H]
    7 rcu_sched       [rcu_sched]
    8 rcu_bh          [rcu_bh]
    9 migration/0     [migration/0]
   10 watchdog/0      [watchdog/0]
   11 watchdog/1      [watchdog/1]
   12 migration/1     [migration/1]
   13 ksoftirqd/1     [ksoftirqd/1]
   15 kworker/1:0H    [kworker/1:0H]
   16 watchdog/2      [watchdog/2]
   ...
```

* 每一行代表了一个进程。每一行又分为三列。第一列PID(process IDentity)是一个整数，每一个进程都有一个唯一的PID来代表自己的身份，进程也可以根据PID来识别其他的进程。第二列COMMAND是这个进程的简称。第三列CMD是进程所对应的程序以及运行时所带的参数。
* (第三列有一些由中括号[]括起来的。它们是内核的一部分功能，被打扮成进程的样子以方便操作系统管理。我们不必考虑它们。)
* 我们看第一行，PID为1，名字为init。这个进程是执行/bin/init这一文件(程序)生成的。当Linux启动的时候，init是系统创建的第一个进程，这一进程会一直存在，直到我们关闭计算机。这一进程有特殊的重要性，我们会不断提到它。

### 如何创建一个进程
* 实际上，当计算机开机的时候，内核(kernel)只建立了一个init进程。Linux内核并不提供直接建立新进程的系统调用。剩下的所有进程都是init进程通过fork机制建立的。新的进程要通过老的进程复制自身得到，这就是fork。fork是一个系统调用。进程存活于内存中。每个进程都在内存中分配有属于自己的一片空间 (address space)。当进程fork的时候，Linux在内存中开辟出一片新的内存空间给新的进程，并将老的进程空间中的内容复制到新的空间中，此后两个进程同时运行。
* 老进程成为新进程的父进程(parent process)，而相应的，新进程就是老的进程的子进程(child process)。一个进程除了有一个PID之外，还会有一个PPID(parent PID)来存储的父进程PID。如果我们循着PPID不断向上追溯的话，总会发现其源头是init进程。所以说，所有的进程也构成一个以init为根的树状结构。
如下，我们查询当前shell下的进程：

```
ps -o pid,ppid,cmd

 PID  PPID CMD
 5860 31925 ps -o pid,ppid,cmd
31925 31924 -bash
```
* 我们可以看到，第二个进程bash是第一个进程sudo的子进程，而第三个进程ps是第二个进程的子进程。
* 还可以用$pstree命令来显示整个进程树：

```
systemd─┬─NetworkManager─┬─dhclient
        │                ├─dnsmasq
        │                ├─{gdbus}
        │                └─{gmain}
        ├─accounts-daemon─┬─{gdbus}
        │                 └─{gmain}
        ├─acpid
        ├─agetty
        ├─atd
        ...
```

fork通常作为一个函数被调用。这个函数会有两次返回，将子进程的PID返回给父进程，0返回给子进程。实际上，子进程总可以查询自己的PPID来知道自己的父进程是谁，这样，一对父进程和子进程就可以随时查询对方。

通常在调用fork函数之后，程序会设计一个if选择结构。当PID等于0时，说明该进程为子进程，那么让它执行某些指令,比如说使用exec库函数(library function)读取另一个程序文件，并在当前的进程空间执行 (这实际上是我们使用fork的一大目的: 为某一程序创建进程)；而当PID为一个正整数时，说明为父进程，则执行另外一些指令。由此，就可以在子进程建立之后，让它执行与父进程不同的功能。

### 子进程的终结(termination)
当子进程终结时，它会通知父进程，并清空自己所占据的内存，并在内核里留下自己的退出信息(exit code，如果顺利运行，为0；如果有错误或异常状况，为>0的整数)。在这个信息里，会解释该进程为什么退出。父进程在得知子进程终结时，有责任对该子进程使用wait系统调用。这个wait函数能从内核中取出子进程的退出信息，并清空该信息在内核中所占据的空间。但是，如果父进程早于子进程终结，子进程就会成为一个孤儿(orphand)进程。孤儿进程会被过继给init进程，init进程也就成了该进程的父进程。init进程负责该子进程终结时调用wait函数。

当然，一个糟糕的程序也完全可能造成子进程的退出信息滞留在内核中的状况（父进程不对子进程调用wait函数），这样的情况下，子进程成为僵尸（zombie）进程。当大量僵尸进程积累时，内存空间会被挤占。

### 进程与线程(thread)
* 尽管在UNIX中，进程与线程是有联系但不同的两个东西，但在Linux中，线程只是一种特殊的进程。多个线程之间可以共享内存空间和IO接口。所以，进程是Linux程序的唯一的实现方式。

### 总结
程序，进程，PID，内存空间
子进程，父进程，PPID，fork， wait

## 扩展
* [Blinker](https://github.com/jek/blinker)：一个快速的Python进程中信号/事件调度系统
* 不同于signal模块，blink是第三方模块。个人觉得blink使用起来要比signal简单许多，而且这个模块可以在Windows下使用

```
from blinker import Namespace

# 1.定义信号
namespace = Namespace()
fire = namespace.signal("fire")

# 2.监听信号
# 首先定义一个回调函数，里面至少要接收一个参数


def open_fire(sender):
    print(sender)
    print("open_fire")


# 监听
fire.connect(open_fire)


# 3.发送信号
# send里面的参数则会传给open_fire
fire.send("xxx")
```

总结一下：步骤就三步
* 1.定义信号

```
namespace = NameSpace()
signal = namespace.signal("signal")
```
* 2.监听信号，创建需要至少要接收一个参数的函数，然后通过signal.connect进行注册
* 3.发送信号

```
调用signal.send("message")方法，然后会触发绑定的函数，同时message也会传进去，如果不传则为None
如果回调函数里面只有一个参数，又不想传的话，直接send即可，会自动将None传进去
但是如果回调函数要接收多个参数，那么第一个sender必须通过位置参数指定，其余要通过关键字参数指定
```


## 附录
* https://docs.python.org/zh-cn/3/library/signal.html?highlight=signal#note-on-sigpipe
* https://www.programcreek.com/python/example/1220/signal.SIGALRM



目录
[隐藏]

1 用户进程间通信
1.1 System V IPC对象管理
1.1.1 System V IPC数据结构
1.1.1.1 （1）IPC对象属性结构kern_ipc_perm
1.1.1.2 （2）结构ipc_ids
1.1.1.3 （3）结构ipc_namespace
1.1.2 IPC对RCU的支持
1.1.2.1 （1）RCU前缀对象结构
1.1.2.2 （2）分配IPC对象时加入RCU前缀对象
1.1.2.3 （3）修改IPC对象引起延迟更新
1.1.3 IPC对象查找
1.1.4 释放IPC命名空间
1.2 管道
1.2.1 管道的实现
1.3 消息队列
1.3.1 消息队列结构
1.3.2 消息队列文件系统
1.3.3 消息队列系统调用函数
1.4 共享内存
1.4.1 共享内存相关结构
1.4.2 共享内存文件系统
1.4.3 共享内存系统调用
1.4.4 创建共享内存
1.4.5 映射函数shmat
1.5 信号量
1.5.1 信号量数据结构
1.5.1.1 （1）信号量结构sem
1.5.1.2 （2）信号量集结构sem_array
1.5.1.3 （3）信号量集合的睡眠队列结构sem_queue
1.5.1.4 （4）信号量操作值结构sembuf
1.5.1.5 （5）死锁恢复结构sem_undo
1.5.2 系统调用函数功能说明
1.5.3 系统调用函数的实现
1.5.3.1 （1）初始化函数sem_init
1.5.3.2 （2）系统调用sys_semget
1.5.3.3 （3）系统调用sys_semop
1.6 快速用户空间互斥锁（Futex）
1.7 信号
1.7.1 信号概述
1.7.1.1 （1）信号定义
1.7.1.2 （2）实时信号与非实时信号
1.7.1.3 （3）信号响应
1.7.2 信号相关系统调用说明
1.7.3 信号相关数据结构
1.7.3.1 （1）进程描述结构中的信号域
1.7.3.2 （2）信号描述结构signal_struct
1.7.3.3 （2）信号处理结构sighand_struct
1.7.3.4 （3）信号响应结构k_sigaction
1.7.3.5 （4）挂起信号及队列结构
1.7.4 设置信号响应
1.7.5 信号分发
1.7.6 信号响应
1.7.6.1 （1）系统调用返回触发信号响应
1.7.6.2 （2）从进程上下文中获取信号并进行信号的缺省操作
1.7.6.3 （3）利用堆栈处理用户注册的响应函数