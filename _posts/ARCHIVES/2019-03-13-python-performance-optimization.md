---
layout: post
title: Python性能优化
category: ARCHIVES
description: 优化
tags: Python
---

### 问题现象
* 1.服务CPU利用率达到99%，持续不下降
* 2.服务CPU利用率突然下降低至0%，请求卡住


### 问题解决
#### 算法优化
```
对比发现：
    600kb图片请求一次服务需要1s内返回结果
    1.3M图片请求一次服务需要20s返回结果
具体解决：
    查询发现为pyzbar组件问题，适合进行小图分析，不适合大图分析
处理逻辑：
    加一层检测算法，检测初二维码图片，进行缩放后进行后续的pyzbar识别处理
```

#### 服务问题
* 查看卡死进程相关信息：https://blog.csdn.net/chroming/article/details/82057397
```
    所需组件：strace、lsof
    查看进程号：ps auxf
    查看进程卡在那个进程回调：strace -p 7（发现有很多死循环socket监听请求）
    查看进程所有文件：lsof -p 7(用处不大)
```
* 查看进程运行代码相关信息：https://mozillazg.com/2017/07/debug-running-python-process-with-gdb.html
```
    所需组件：gdb python2.7-dbg
    查看调试中的进程：gdb python 7
    查看调试中进程代码：py-list、py-bt
```
* 通过上面发现代码timeout等待在waitress调用中
```
    查找各种无果，最终发现为组件bug，具体见：https://github.com/Pylons/waitress/issues/218
    处理措施：使用Gevent进行服务部署，替换waitress
```

备注：
    1.strace：可查看卡死的进程级别的原因
    2.gdb和python2.7-dbg：可查看卡死进程的具体运行中代码

