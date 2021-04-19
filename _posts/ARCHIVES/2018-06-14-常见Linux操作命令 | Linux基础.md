---
layout: post
title: Linux使用总结
category: ARCHIVES
description: 常见Linux操作案例
tags: Linux
---

## 文件授权

```
当前文件所有人皆可读取：chmod a+r file.txt
当前文件所有人皆可读取：chmod 777 file.txt
当前目录下所有文件和子目录所有人皆可读取：chmod -R a+r /files
```

## 压缩相关

```
.tar
解压：tar xvf filename.tar
压缩：tar cvf filename.tar dirname/

.zip
解压：unzip filename.zip
压缩：zip filename.zip dirname/

.rar
解压：rar a filename.zip
压缩：rar e filename.zip

.gz
解压：gzip -d filename.gz 或 gunzip filename.gz
压缩：gzip filename/
```

## 软链接

```
ln -s /www/test hash
```


## .txt文件行数统计

```
wc -lcw 2018-02-22.txt 2018-02-22.txt 
- c 统计字节数。
- l 统计行数。
- w 统计字数。

举例分析：
    1.统计demo目录下，js文件数量：
        find demo/ -name "*.js" |wc -l
    2.统计demo目录下所有js文件代码行数：
        find demo/ -name "*.js" |xargs cat|wc -l 或 wc -l `find ./ -name "*.js"`|tail -n1
    3.统计demo目录下所有js文件代码行数，过滤了空行：
        find /demo -name "*.js" |xargs cat|grep -v ^$|wc -l
```

## 统计操作

```
统计文件个数：ls -lR|grep "^-"|wc -l
```

## 过滤操作

```
输出符合条件的进程ID：ps -ef | grep 'app.py 5000' | grep -v grep|awk '{print $2}'
```

## Crontab的使用

```
语法：crontab(选项)(参数)
参数：
    -e：编辑该用户的计时器设置
    -l：列出该用户的计时器设置
    -r：删除该用户的计时器设置
    -u<用户名称>：指定要设定计时器的用户名称

定时配置：minute hour day month week command   顺序：分 时 日 月 周
eg：配置每分钟  */1 * * * *

```

## 查看系统

```
查看操作系统版本：cat /etc/issue 或 head -n 1 /etc/issue
查看详细版本号：lsb_release -a
查看内核/操作系统/CPU信息：uname -r 显示所有(-a)
查看CPU信息：cat /proc/cpuinfo
查看计算机名：hostname
列出所有PCI设备：lspci -tv
列出所有USB设备：lsusb -tv
列出加载的内核模块：lsmod
查看环境变量：env
查看机器型号: sudo dmidecode | grep "Product Name"
```

## 查看资源

```
df -h
du -h --max-depth=1 /www        # 遍历文件大小
free -m                         # 查看内存使用量和交换区使用量
du -sh /www                     # 查看指定目录的大小
grep MemTotal /proc/meminfo     # 查看内存总量
grep MemFree /proc/meminfo      # 查看空闲内存量
uptime                          # 查看系统运行时间、用户数、负载
cat /proc/loadavg               # 查看系统负载

cat /proc/meminfo               # 查看内存
```

## 查看磁盘

```
mount /dev/sdb1 /data1
mount 172.18.34.20:/data1 /data1    # 磁盘挂载
swapon -s                           # 查看所有分区
```


## 查看网络

```
ifconfig               # 查看所有网络接口的属性
iptables -L            # 查看防火墙设置
route -n               # 查看路由表
netstat -lntp          # 查看所有监听端口
netstat -antp          # 查看所有已经建立的连接
netstat -s             # 查看网络统计信息
```

## 查看进程

```
ps -ef                    # 查看所有进程
top                       # 实时显示进程状态
htop                      # 实时显示进程状态
pstree -p <PID> | wc -l   # 查看进程相关线程数
```

## 查看用户

```
w                         # 查看活动用户
id <用户名>                # 查看指定用户信息
last                      # 查看用户登录日志
cut -d: -f1 /etc/passwd   # 查看系统所有用户
cut -d: -f1 /etc/group    # 查看系统所有组
crontab -l                # 查看当前用户的计划任务

用户密码修改:
    sudo passwd <username> 或者 sudo /usr/bin/passwd <username>
    Password：当前密码
    Enter new UNIX password：新密码
    Retype new UNIX password：确认新密码
```

## 查看服务

```
打印上下文：curl -i 'url' 
打印 头 部：curl -v 'url'
打印处理时间：time curl 'url'

```

## 查看Linux日志排错

```
Out of Memory错误：sudo grep "Out of memory" /var/log/syslog
同理：
    登录日志：/var/log/auto.log
    定时日志：/var/log/cron
    信息日志：/var/log/messages
    系统日志：/var/log/syslog
```

## 开发环境操作

```
设置环境变量：export HOST='XXX'
去除环境变量：unset HOST
```

## 查看CPU信息（型号） 

```
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c 
    8  Intel(R) Xeon(R) CPU            E5410   @ 2.33GHz 
(看到有8个逻辑CPU, 也知道了CPU型号) 

cat /proc/cpuinfo | grep physical | uniq -c 
    4 physical id      : 0 
    4 physical id      : 1 
(说明实际上是两颗4核的CPU) 

getconf LONG_BIT 
    32
(说明当前CPU运行在32bit模式下, 但不代表CPU不支持64bit) 

cat /proc/cpuinfo | grep flags | grep ' lm ' | wc -l 
    8 
(结果大于0, 说明支持64bit计算. lm指long mode, 支持lm则是64bit) 
```

## Ubuntu机器查看网卡速率
```
apt-get install eth-tool
ethtool bond0 | grep Speed
```
