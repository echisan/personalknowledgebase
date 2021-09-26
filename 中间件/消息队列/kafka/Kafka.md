# Kafka

参考：深入理解Kafka：核心设计与实践原理

多分区、多副本、分布式消息系统；

kafka扮演的三大角色

- 消息系统：具备传统消息系统的功能（解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等）。同时kafka提供了大多数消息系统难以实现的**消息顺序性保障**以及**回溯消费**的功能。
- 存储系统：Kafka把消息持久化到磁盘，得益于消息持久化功能以及多副本机制可以把Kafka当作长期的数据存储系统来使用
- 流式处理平台：kafka不仅为每个流行的流式处理框架提供了可靠的数据来源，还提供了一个完整的流式处理类库，比如窗口、连接、变换和聚合等各类操作

Kafka为分区引入了多副本机制，通过增加副本数量可以提升容灾能力。

同一分区的不同副本中保存的是相同的消息（同一时刻，副本之间并非完全一样）

副本之间是”一主多从“的关系，leader副本负责读写，**follower副本只负责与leader副本的消息同步**



分区中的所有副本统称为AR（Assigned Replicas）

所有与leader副本保持一定程度同步的副本（包括leader副本在内）组成ISR（In-Sync Replicas）

ISR是AR集合中的一个子集

消息会先发送至leader副本，然后follower副本从leader副本拉取消息进行同步，同步期间follower副本相对于leader副本而言会有一定程度的滞后。

与leader副本同步滞后过多的副本（不包括leader副本）组成OSR（Out-of-Sync Replicas）

AR = ISR + OSR

正常情况下，所有的follower副本都应该与leader副本保持一定程度的同步，即AR = ISR， OSR集合为空



leader副本维护和跟踪follower副本的滞后状态，当follower副本落后太多或失效时，leader副本会把它从ISR集合中移除。

如果OSR集合中的follower副本”追上“了leader副本，那么leader副本会把它从OSR集合转移到ISR集合

默认情况下，当leader副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader，而OSR集合中的副本没有任何机会（不过这个原则也可以通过修改相应的参数配置来改变）

ISR与HW和LEO也有紧密的关系，分区ISR集合中的每个副本都维护自身的LEO，而ISR集合中最小的LEO即为分区的HW，对消费者而言只能消费HW之前的消息。

- HW是High Watermark的缩写，俗称高水位，标识了一个特定的消息偏移量（offset），消费者只能拉取到这个offset之前的消息
- LEO是Log End Offset的缩写，它标识当前日志文件中下一条待写入消息的offset

![image-20210923093644277](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210923093644277.png)

