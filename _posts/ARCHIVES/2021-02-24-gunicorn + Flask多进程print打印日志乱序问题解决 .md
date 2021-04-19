---
layout: post
title: Gunicorn + Flask多进程print打印日志乱序问题解决
category: ARCHIVES
description: 描述
tags: 标签
---

### 问题描述
* gunicorn + Flask部署Python服务，worker > 2 时发现 print 打印日志至 stdout 乱序

### 问题分析
* 多进程部署环境 print 打印不安全，需要添加全局进程锁

### 问题处理
* Python中最常见多进程锁（multiprocessing.Lock）和多线程锁（threading.Lock），多进程锁实现锁定子进程资源功能，多线程实现锁定子线程资源功能。
* gunicorn + Flask架构，gunicorn会启动多个worker子进程，每个子进程可看做一个独立的Flask进程，现在需要在所有worker进程中Flask应用内申请锁资源，并且锁资源需要在其他worker里互斥
* 通用解决方案：db记录锁、redis记录锁、文件锁，此处选用**文件锁**，具体代码如下：

```
# -*- coding: utf-8 -*-
import os
import time
import fcntl

class Lock(object):
    @staticmethod
    def get_file_lock():
        return FileLock()

class FileLock(object):
    def __init__(self):
        lock_file = 'HYDRA_FILE_LOCK'
        lock_dir = '/tmp'
        self.file = '%s%s%s' % (lock_dir, os.sep, lock_file)
        self._fn = None
        self.release()

    def acquire(self):
            try:
                self._fn = open(self.file, 'w')
                fcntl.flock(self._fn, fcntl.LOCK_EX)
            except Exception:
                pass

    def release(self):
            try:
                if self._fn:
                    fcntl.flock(self._fn, fcntl.LOCK_UN)
                    self._fn.close()
            except Exception:
                pass

if __name__ == "__main__":
    import time
    N = 50
    lock = Lock.get_file_lock()
    start_time = time.time()
    for i in range(N):
        a = 2**9999
    t1 = (time.time() - start_time) / float(N)
    
    start_time = time.time()
    for i in range(N):
        print(a)
    t2 = (time.time() - start_time) / float(N)

    start_time = time.time()
    for i in range(N):
        lock.acquire()
        print(a)
        lock.release()
    t3 = (time.time() - start_time) / float(N)

    print("数值计算耗时：%s" % str(t1))
    print("直接打印耗时：%s" % str(t2))
    print("加锁打印耗时：%s" % str(t3))
```

### 小结
* Windows环境下不能完全生效，基于Linux会正常工作，因为Linux下使用了文件锁模块，可以确保加锁过程原子操作
* 加锁势必带来性能损耗，请评估后使用

---
That's all!