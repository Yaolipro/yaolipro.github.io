---
layout: post
title: Pytorch遍历DataLoader时报错【BrokenPipeError:[Errno 32]Broken pipe】
category: ISSUES
description: 描述
tags: Pytorch
---

## 问题描述
* GPU环境训练好模型，CPU环境部署过程成功后，尝试遍历DataLoader的时候出现了以下报错信息。具体如下：

```
Traceback (most recent call last):
  File "/usr/local/lib/python3.6/multiprocessing/resource_sharer.py", line 142, in _serve
    with self._listener.accept() as conn:
  File "/usr/local/lib/python3.6/multiprocessing/connection.py", line 455, in accept
    deliver_challenge(c, self._authkey)
  File "/usr/local/lib/python3.6/multiprocessing/connection.py", line 720, in deliver_challenge
    connection.send_bytes(CHALLENGE + message)
  File "/usr/local/lib/python3.6/multiprocessing/connection.py", line 200, in send_bytes
    self._send_bytes(m[offset:offset + size])
  File "/usr/local/lib/python3.6/multiprocessing/connection.py", line 404, in _send_bytes
    self._send(header + buf)
  File "/usr/local/lib/python3.6/multiprocessing/connection.py", line 368, in _send
    n = write(self._handle, buf)
BrokenPipeError: [Errno 32] Broken pipe
```

## 问题分析
* 逐步打断点调试定位错误代码为Pytorch遍历DataLoader引发。

```
loader =DataLoader(dataset,
	batch_size = batch_size,
    shuffle=shuffle,
    num_workers = 4, #1GPU = 4workers
    pin_memory=True)
```

* DataLoader的 num_workers 涉及多线程读取数据，而Python由于设计时有GIL全局锁，导致了多线程无法利用多核，问题大致就出于此。try-catch错误代码输出：

```
[Errno 11] Resource temporarily unavailable
```

* 大batch训练时，num_workers=4加速GPU训练效率，CPU推理过程不需要。

## 问题解决
* 将num_workers设置为 0 即可。

## 附录
* https://discuss.pytorch.org/t/very-strange-dataloader-error-simplified-code-inside/31162
* https://github.com/pytorch/pytorch/issues/14768