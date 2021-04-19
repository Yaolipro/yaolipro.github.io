> Apache Kafka 基准测试：每秒写入 2 百万（在三台廉价机器上）


#### 基于sendfile实现Zero Copy

#### 批量压缩
* 它把所有的消息都变成一个批量的文件，并且进行合理的批量压缩，减少网络 IO 损耗，通过 mmap 提高 I/O 速度，写入数据的时候由于单个 Partion 是末尾添加所以速度最优；读取数据的时候配合 sendfile 直接暴力输出


### 附录
* https://juejin.cn/post/6844903632521920519
* https://www.jianshu.com/p/650c9878dee7
* https://www.jianshu.com/p/44f2bd60b7e9