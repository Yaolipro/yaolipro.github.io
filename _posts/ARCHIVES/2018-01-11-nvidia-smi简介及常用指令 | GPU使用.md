---
layout: post
title: nvidia-smi简介及常用指令 | GPU使用
category: ARCHIVES
tags: 系统
---

## nvidia-smi
NVIDIA系统管理界面（nvidia-smi）是基于NVIDIA Management Library（NVML）的命令行实用程序，旨在帮助管理和监视NVIDIA GPU设备。


## GPU参数查看
### 查看GPU运行情况

```
nvidia-smi
```

```
Sun Mar 28 02:40:38 2021
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.56       Driver Version: 418.56       CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  On   | 00000000:02:00.0 Off |                  N/A |
| 23%   29C    P8     9W / 250W |    611MiB / 11178MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  GeForce GTX 108...  On   | 00000000:03:00.0 Off |                  N/A |
| 23%   30C    P8     9W / 250W |      0MiB / 11178MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  GeForce GTX 108...  On   | 00000000:82:00.0 Off |                  N/A |
| 23%   30C    P8     9W / 250W |      0MiB / 11178MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  GeForce GTX 108...  On   | 00000000:83:00.0 Off |                  N/A |
| 23%   30C    P8     9W / 250W |      0MiB / 11178MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0     33777      C   /usr/bin/python                              601MiB |
+-----------------------------------------------------------------------------+
```

这是`GEFORCE GTX 1080 Ti`GPU服务器的运行信息。
* 第一行分别为：命令行工具版本、GPU驱动版本、CUDA版本
* 第一栏分别为：GPU(GPU卡号，0～4)、Fan(风扇转速，0～100%)
* 第二栏分别为：Name(显卡名字)、Temp(温度，摄氏度)
* 第三栏分别为：Perf(性能状态，P0~P12，最高性能为P0，最低性能为P12)
* 第四栏分别为：Persistence-M(持续模式，默认为关闭，比较节能，如果设置成ON，耗能比较大，但新的GPU应用启动时，花费的时间会更短)、Pwr:Usage/Cap(能耗)
* 第五栏分别为：Bus-Id(GPU总线，domain:bus:device.function)
* 第六栏分别为：Disp.A(GPU的显示是否初始化)、Memory-Usage(显存利用率)
* 第七栏分别为：Volatile GPU-Util(GPU浮动利用率)
* 第八栏分别为：Uncorr. ECC(Error Correcting Code错误检查和纠正码)、Compute M.(计算模式)
* 下面一张表为：每个GPU Processes的资源占用情况

`注`：显存占用和GPU占用是两个不一样的，显卡是由GPU和显存等组成的，显存和GPU的关系可简单理解为内存和CPU的关系。

### 获取GPU ID信息

```
nvidia-smi -L
```

从左到右分别为：GPU卡号、GPU型号、GPU物理UUID号

```
GPU 0: GeForce GTX 1080 Ti (UUID: GPU-5da6e67e-fd5a-88fb-7a0e-109c3284f7bf)
GPU 1: GeForce GTX 1080 Ti (UUID: GPU-ce9189e4-2e58-3a19-4332-cb5c7fac1aa6)
GPU 2: GeForce GTX 1080 Ti (UUID: GPU-242b3020-8e5c-813a-42d9-475766d52f9d)
GPU 3: GeForce GTX 1080 Ti (UUID: GPU-8f3d825f-7246-3daf-eaa1-37845b03aa03)
```

单独过滤出GPU卡号信息

```
nvidia-smi -L | cut -d ' ' -f 2 | cut -c 1
```

## GPU常用设置
### 启动模式设置
解决GPU启动加载慢问题

```
设置GPU持续模式：Persistence-M
sudo nvidia-smi -pm 1
```

### 节点分配
解决卡性能不均匀问题，如果是四卡机器，只使用两个节点`优先选择0和3`，边界卡槽有利于散热


## 附录
* https://developer.nvidia.com/nvidia-system-management-interface