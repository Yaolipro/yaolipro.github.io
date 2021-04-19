[TOC]

>这篇文章主要想跟大家分享一下Gunicorn的基础逻辑和部分源码实现流程

## Gunicorn介绍

### 简介
* Gunicorn是一个被广泛使用的高性能的PythonWSGIUNIXHTTP服务器。
* 由于源码调用了fcntl、fork等接口，因此只能跑在类Unix系统上。
* 与各种Web框架广泛兼容，实施简单，服务器资源使用率低。

### 设计思想
* Gunicorn基于pre-fork的思想，由一个中央主进程（master）来管理一组工作进程（worker），worker进程负责对来自客户端的请求进行响应，gunicorn依靠操作系统来提供负载均衡。
* pre-fork服务器会预先开启一定数量（可配置参数）的worker进程，在用户请求到来时服务器并不需要等待新的进程启动，因而能够以更快的速度应付多用户请求。

#### Master
* 主进程循环监听各种worker进程信号管理正在运行的worker列表。比如：TTIN，TTOU和CHLD等信号。
* TTIN和TTOU告诉主机增加或减少正在运行的worker数量。
* CHLD指示子进程已终止，主进程会自动重新启动发生故障的工作程序。

#### Workers
* Sync Workers：同步worker，一次处理单个请求，默认不支持Keep-Alive。
* Async Workers：异步worker，多线程批量处理请求，支持greenlet、eventlet和gevent异步框架实现。
* AsyncIO Workers：基于Python3实现的异步gthread worker。主线程循环接受连接并添加至线程池中。worker线程
您还可以移植应用程序以使用aiohttp的web.ApplicationAPI并使用aiohttp.worker.GunicornWebWorker工作程序。
* Tornado Workers：官方不建议使用，忽略。


### 选择Worker类型
默认的同步工作程序假定您的应用程序在CPU和网络带宽方面受资源限制。通常，这意味着您的应用程序不应执行任何花费不确定时间的事情。花费时间不确定的一个例子是对互联网的请求。在某些时候，外部网络将以客户端将堆积在您的服务器上的方式失效。因此，从这个意义上讲，向API发出传出请求的任何Web应用程序都将从异步工作程序中受益。

这种资源受限的假设是为什么我们需要在默认配置Gunicorn之前使用缓冲代理的原因。如果将同步工作人员暴露于Internet，则通过创建将数据滴入服务器的负载，DOS攻击将变得微不足道。出于好奇，Hey是此类负载的一个示例。

需要异步工作程序的一些行为示例：

长时间阻止呼叫的应用程序（即，外部Web服务）
直接将请求服务到互联网
流式传输请求和响应
长时间轮询
网络插座
彗星

### 源码解析

重试代码编写逻辑：https://github.com/benoitc/gunicorn/blob/master/gunicorn/sock.py

源码分析：https://www.jianshu.com/p/1e5feccb37d9

