---
layout: post
title: Signal信号量使用详解 | Python基础
category: ARCHIVES
description: 描述
tags: Python
---

## 信号定义?
* linux中信号被用来进行进程间的通信和异步处理，简单地可以理解会为回调函数，当发送一个信号时，触发相应的操作。
* [signal](https://docs.python.org/zh-cn/3/library/signal.html?highlight=signal#note-on-sigpipe)是python中用来处理信号的模块，主要针对UNIX类平台，比如：Linux、MAC OS等。
* Python支持的信号和Linux内置信号几乎一致。

## 常用信号量
* `signal.SIGHUP`   # 连接挂断，这个信号的默认操作为终止进程，因此会向终端输出内容的那些进程就会终止。不过有的进程可以捕捉这个信号并忽略它。比如wget。
* `signal.SIGINT`   # 连接中断，程序终止(interrupt)信号，按下`CTRL + C`的时候触发。
* `signal.SIGTSTP` # 暂停进程，停止进程的运行，按下`CTRL + Z`的时候触发， 该信号可以被处理和忽略。
* `signal.SIGCONT` # 继续执行，让一个停止(stopped)的进程继续执行。本信号不能被阻塞。
* `signal.SIGKILL` # 终止进程，用来立即结束程序的运行，本信号无法被阻塞、处理和忽略。
* `signal.SIGALRM` # 超时警告，时钟定时信号，计算的是实际的时间或时钟时间

## 信号操作
###  发送定时信号
* `signal.alarm(time)`
	- 设置发送SIGALRM信号的定时器

```python
signal.alarm(5)
```

### 设置信号函数
* `signal.signal(signalnum, handler)`
	- signalnum：具体信号
	- handler：信号的执行函数
* 可以设置执行默认操作，也可以自定义操作，但必须要接收两个参数，那如果我们想忽略信号的话，也可以有两种操作方法：
	- 直接在handler的函数体中写一个pass
	- 或设置signal.SIG_DFL(默认执行)、signal.SIG_IGN(程序忽略该信号。

```python
# -*- coding: utf-8 -
import signal

def alert_handler(signum, frame):
    print('Signal handler called with signal', signum)

# 1.设置定时信号
signal.signal(signal.SIGALRM, alert_handler)
print(signal.getsignal(s))
signal.alarm(3)

# 2.进程暂定，等待时钟信号
import time
time.sleep(10)

# 3.禁用定时信号
signal.alarm(0)
```

* 注意：
	- 该方法是有返回值的，返回原有默认的信号处理函数
	- 该方法只能在主线程中注册，如果在子线程中注册， 会引发一个`ValueError`

### 发送休眠信号
* `signal.pause()`
	- 进程暂停，以等待信号(什么信号都行)。因此该函数相当于阻塞进程进行，接收到信号后继续前进。

```python
signal.pause()
```

### 获取信号处理程序
* `signal.getsignal(signalnum)`
	- 根据 signalnum 返回信号对应的 handler，可能是一个可以调用的 Python 对象，或者是 signal.SIG_IGN（表示被忽略）, signal.SIG_DFL（默认行为）或 None（Python 的 handler 还没被定义）。

```python
# -*- coding: utf-8 -
import signal

signals_to_names = {
    getattr(signal, n): n
    for n in dir(signal)
    if n.startswith('SIG') and '_' not in n
}

for s, name in sorted(signals_to_names.items()):
    handler = signal.getsignal(s)
    if handler is signal.SIG_DFL:
        handler = 'SIG_DFL'
    elif handler is signal.SIG_IGN:
        handler = 'SIG_IGN'
    print('{:<10} ({:2d}):'.format(name, s), handler)
```

### 发送终止信号
* `signal`包的核心是设置信号处理函数。除了`signal.alarm()`向自身发送信号之外，并没有其他发送信号的功能。但在 os 包中，有类似于 Linux 的 kill 命令的函数：
* `os.kill(pid, sid)`
	- 给某一进程发送终止信号
* `os.killpg(pgid, sid)`
	- 给某一进程组发送终止信号

```python
# -*- coding: utf-8 -
import signal
import os
import time

# 执行打印
def receive_signal(signum, stack):
    print('Received:', signum)

# 执行退出操作
def do_exit(signum, stack):
	print('Received:', signum)
    raise SystemExit('Exiting')

# 设置用户自定义信号 1
signal.signal(signal.SIGUSR1, receive_signal)
# 设置用户自定义信号 2
signal.signal(signal.SIGUSR2, do_exit)
print('My PID is:', os.getpid())

while True:
    print('Waiting...')
    time.sleep(3)
```

* 另开启一个窗口运行命令
```shell
# 运行指令
$ kill -USR1 $pid	# 触发自定义信号1
$ kill -USR2 $pid	# 触发自定义信号2，触发SystemExit异常
$ kill -INT $pid	# 进程退出
# 具体打印结果
My PID is: 40503
Waiting...
Received: 30
Waiting...
Received: 31
Waiting...
Exiting
Traceback (most recent call last):
  File "signal_os_kill.py", line 19, in <module>
    time.sleep(3)
KeyboardInterrupt
```

## 使用案例
### 超时工具函数
* 该设置实现了函数执行超时返回默认结果的功能。
	- 先是设置了一个超时处理函数，在函数中抛出自定义的抛出异常。
	- 当超出时间后触发抛出异常`SIGALRM`，然后捕获这个异常设置默认值。
	- 最后做下清理工作将定时器取消，并且将对`SIGALRM`的处理设为默认。

```python
# -*- coding: utf-8 -

# From https://github.com/breenmachine/httpscreenshot/blob/master/httpscreenshot.py#L41
def timeoutFn(func, args=(), kwargs={}, timeout_duration=1, default=None):
    import signal

    class TimeoutError(Exception):
        pass

    def handler(signum, frame):
        raise TimeoutError()

    # set the timeout handler
    signal.signal(signal.SIGALRM, handler)
    signal.alarm(timeout_duration)
    try:
        result = func(*args, **kwargs)
    except TimeoutError as exc:
        result = default
    finally:
        signal.alarm(0)
        signal.signal(signal.SIGALRM, signal.SIG_DFL)

    return result

def test_demo():
    import time
    time.sleep(10)

print(timeoutFn(test_demo, timeout_duration=3, default=9999))
```

### 多线程信号交互
* Python的`signal`模块要求所有的`handlers`必须在`main thread`中注册。
* 代码结尾处的`signal.alarm(2)`是为了唤醒接收线程的`pause(`，否则接收线程永远不会退出。
* 以下案例在`receiver`展现了不支持`signal`子线程注册信号和在`sender`可以发送信号

```python
# -*- coding: utf-8 -
import signal
import threading
import os
import time

def usr1_handler(num, frame):
    print("received signal %s %s" % (num, threading.currentThread()))

signal.signal(signal.SIGUSR1, usr1_handler)

def thread_get_signal():
    # 如果在子线程中设置signal的handler会报错
    # ValueError: signal only works in main thread
    # signal.signal(signal.SIGUSR2, usr1_handler)

    print("waiting signal", threading.currentThread())
    # sleep 进程直到接收到信号
    signal.pause()
    print("waiting done")

receiver = threading.Thread(target=thread_get_signal, name="receiver")
receiver.start()
# 为了保证线程开启顺利，加0.1s延迟
time.sleep(0.1)

pid = os.getpid()
print('pid', pid)

def send_signal():
    print("sending signal", threading.currentThread())
    os.kill(pid, signal.SIGUSR1)

sender = threading.Thread(target=send_signal, name="sender")
sender.start()
sender.join()

# 为了让程序结束，唤醒 pause
signal.alarm(1)
receiver.join()

# ('waiting signal', <Thread(receiver, started 123145560608768)>)
# ('pid', 78335)
# ('sending signal', <Thread(sender, started 123145564815360)>)
# received signal 30 <_MainThread(MainThread, started 4319649280)>
# zsh: alarm      python signal_multithreading.py
```

* 下面展示了可以在子线程中设置`alarms`，但他们只能在`main thread`接收。

```python
# -*- coding: utf-8 -
import signal
import time
import threading

def signal_handler(num, stack):
    print(time.ctime(), 'Alarm in', threading.currentThread().name)

signal.signal(signal.SIGALRM, signal_handler)

def use_alarm():
    t_name = threading.currentThread().name
    print(time.ctime(), 'Setting alarm in', t_name)
    signal.alarm(1)
    print(time.ctime(), 'Sleeping in', t_name)
    time.sleep(3)
    print(time.ctime(), 'Done with sleep in', t_name)

# 开启一个线程，触发signal.alarm(1)
alarm_thread = threading.Thread(target=use_alarm, name='alarm_thread')
alarm_thread.start()
time.sleep(0.1)

# 等待信号(不会发生!)
print(time.ctime(), 'Waiting for', alarm_thread.name)
alarm_thread.join()

print(time.ctime(), 'Exiting normally')

# ('Thu Mar 25 23:00:16 2021', 'Setting alarm in', 'alarm_thread')
# ('Thu Mar 25 23:00:16 2021', 'Sleeping in', 'alarm_thread')
# ('Thu Mar 25 23:00:16 2021', 'Waiting for', 'alarm_thread')
# ('Thu Mar 25 23:00:19 2021', 'Done with sleep in', 'alarm_thread')
# ('Thu Mar 25 23:00:19 2021', 'Alarm in', 'MainThread')
# ('Thu Mar 25 23:00:19 2021', 'Exiting normally')
```

## 扩展阅读
* [Blinker](https://github.com/jek/blinker)：是一个快速的Python进程中信号/事件调度系统。
* `blinker`是第三方模块，使用相比`signal`较简洁，也支持在Windows下使用。

```python
# -*- coding: utf-8 -
from blinker import Namespace

# 1.定义信号
namespace = Namespace()
fire = namespace.signal("fire")

# 2.定义一个回调函数，里面至少要接收一个参数
def open_fire(sender, a, b, c):
    print(sender, a, b, c)
    print("open fire")

fire.connect(open_fire)

# 如果有多个参数，参数需要通过关键字参数指定传递
fire.send("xxx", a=1, b=2, c=3)
# 第一个参数不指定，则默认传了一个None进去
fire.send(a=1, b=2, c=3)
```

* 如果回调函数里只有一个参数，`send`过程可不用传参数，会自动将None传进去。
* 如果回调函数需接收多个参数，`send`第一个位置可不用传参数，其余必须要通过关键字指定。

## 附录
* https://docs.python.org/zh-cn/3/library/signal.html?highlight=signal#note-on-sigpipe
* https://www.programcreek.com/python/example/1220/signal.SIGALRM
* https://www.python.org/dev/peps/pep-0475/
