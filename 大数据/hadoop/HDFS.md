# HDFS

## HDFS读写流程

### HDFS写数据流程

![image-20210902114331203](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902114331203.png)

![image-20220302150321728](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302150321728.png)

![image-20220302150329596](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302150329596.png)s

### 节点距离计算

HDFS写数据的过程中，NameNode会选择距离待上传数据最近距离的DataNode接收数据

两个节点到达最近的共同祖先的距离总和

![image-20210902113842053](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902113842053.png)

### 机架感知（副本存储节点选择）

第一个副本在Client所处的节点上，如果客户端在集群外，随机选一个

第二个副本在另一个机架的随机一个节点

第三个副本在第二个副本所在的机架的随机节点

### HDFS读数据流程

串行读

![image-20210902115302304](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902115302304.png)

## NameNode和SecondaryNameNode

### NN和2NN工作机制

fsimage存储数据
edits记录操作日志

1. 加载edits以及fsimage到内存（NN）
2. 客户端发出增删改操作请求（Client）
3. 先将操作记录到edits_inprogress日志文件当中去（NN）
4. 更新内存中的数据（NN）
5. 2NN请求NN是否需要Checkpoints（2NN）【Checkpoints触发条件：1）定时时间到 2）Edits中的数据满了】
   1. 1个小时请求一次NNCheckpoint
   2. 每一分钟请求NN询问操作次数
6. 请求执行Checkpoint（2NN）
7. 滚动正在写的Edits（将edits_inprogress_001 -> edits_001， 新创建edits_inprogress_002，后续的操作日志都写到002中）（NN）
8. 2NN请求NN拉取edits、fsimage文件（2NN）
9. 加载edits、fsimage文件到内存中合并，最终生成fsimage.chkpoint（2NN）
10. 将生成的fsimage.chkpoint文件再发送至NN（2NN）
11. 将fsimage.chkpoint文件重命名为fsimage（NN）

![image-20210902133810208](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902133810208.png)



### Fsimage和Edits解析

1. Fsimage文件：HDFS文件系统元数据的一个永久性检查点，其中包含HDFS文件系统的所有目录和文件inode的序列化信息
2. Edits文件：存放HDFS文件系统的所有更新操作的路径，文件系统客户端执行的所有写操作首先会被记录到edits文件中
3. seen_txid文件保存的是一个数字，就是最后一个edits_的数字

![image-20210902140655295](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902140655295.png)



#### oiv查看Fsimage文件

```shell
hdfs oiv -p 文件类型 -i 镜像文件 -o 转换后文件输出路径
```

####  oev查看Edits文件

```shell
hdfs oev -p 文件类型 -i 编辑日志 -o 转换后文件输出路径
```

### CheckPoint时间设置

1. 通常情况下，SecondaryNameNode每隔一小时执行一次

`dfs.namenode.checkpoint.period`

2. 一分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次

`dfs.namenode.checkpoint.txns`  value: 3600s 操作动作次数

`dfs.namenode.checkpoint.check.period`  value: 60s 1分钟检查一次操作次数


## DataNode

### DataNode的工作机制
1. 一个数据块在DataNode上以文件的形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据库的长度，块数据的检验和以及时间戳
2. DataNode启动后向NameNode注册，通过后，周期性（6小时）的向NameNode上报所有的块的信息

`dfs.blockreport.intervalMsec` 默认6个小时，单位是毫秒， DN向NN汇报当前解读信息的时间间隔

`dfs.datanode.directoryscan.interval` 默认6个小时，单位是秒，DN扫描自己节点块信息列表的时间

![image-20210902144653841](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902144653841.png)

### 数据完整性

DataNode节点保证数据完整性的方法

1. DN读取Block的时候，它会计算CheckSum
2. 如果计算后的CheckSum，与Block创建时值不一样，说明Block已经损坏
3. Client读取其他DataNode上的Block
4. 常见的校验算法crc（32），md5（128），sha1（160）
5. DataNode在其文件创建后周期验证CheckSum

### 掉线时限参数设置

`dfs.namenode.heartbeat.recheck-interval` 单位为毫秒

`dfs.heartbeat.interval` 单位为秒

![image-20210902152341954](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210902152341954.png)
