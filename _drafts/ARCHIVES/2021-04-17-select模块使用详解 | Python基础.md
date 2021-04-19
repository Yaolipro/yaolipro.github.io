---
layout: post
title: select模块使用详解 | Python基础
category: ARCHIVES
description: 描述
tags: Python
---


unix 系统中的select 模型吧, 参数原型:
int select(int maxfdpl, fd_set * readset, fd_set *writeset, fd_set *exceptset, const struct timeval * tiomeout)

第一个是最大的文件描述符长度
第二个是监听的可读集合
第三个是监听的可写集合
第四个是监听的异常集合
第五个是时间限制

对struct fd_set结构体操作的宏
FD_SETSIZE 容量，指定fd_array数组大小，默认为64，也可自己修改宏
FD_ZERO(*set) 置空，使数组的元素值都为3435973836，元素个数为0.
FD_SET(s, *set) 添加，向 struct fd_set结构体添加套接字s
FD_ISSET(s, *set) 判断，判断s是否为 struct fd_set结构体中的一员
FD_CLR(s, *set) 删除，从 struct fd_set结构体中删除成员s


python 就简单了, 直接调用封装好的select , 其底层处理好了文件描述符的相关读写监听(回头再研究下), 我们在Python 中只需这么写:
can_read, can_write, _ = select.select(inputs, outputs, None, None)

第一个参数是我们需要监听可读的套接字, 第二个参数是我们需要监听可写的套接字, 第三个参数使我们需要监听异常的套接字, 第四个则是时间限制设置

如果监听的套接字满足了可读可写条件, 那么所返回的can,read 或是 can_write就会有值了, 然后我们就可以利用这些返回值进行随后的操作了。相比较unix 的select模型, 其select函数的返回值是一个整型, 用以判断是否执行成功.



* select简介：https://my.oschina.net/u/4320183/blog/4363039
* select讲解：https://pymotw.com/2/select/#poll
