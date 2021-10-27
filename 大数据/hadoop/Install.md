# Install Hadoop

## 前置

创建三个ubuntu server虚拟机，分别为master01, node01, node02

## 安装java

```shell
# 安装
sudo apt-get install openjdk-8-jdk
# 安装后的java路径
/usr/lib/jvm/java-8-openjdk-amd64
```

## 下载hadoop

```shell
wget https://archive.apache.org/dist/hadoop/common/hadoop-2.10.1/hadoop-2.10.1.tar.gz
```

### 配置免密登录

生成密钥

```shell
ssh-keygen
```

免密登录，将公钥写到本机和远程机器的`~/.ssh/authorized_key`中

```shell
ssh-copy-id -i ~/.ssh/id_rsa.pub master01
ssh-copy-id -i ~/.ssh/id_rsa.pub node01
ssh-copy-id -i ~/.ssh/id_rsa.pub node02
```

## 配置环境变量

### bashrc

```shell
vim ~/.bashrc
# 追加后面的内容到bashrc文件末尾
export HADOOP_HOME=/home/ubuntu/hadoop/hadoop-2.10.1
export PATH=${HADOOP_HOME}/bin:$PATH
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

### hadoop-env.sh

```shell
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

### core-site.xml

```xml
<configuration>
	    <property>
        <!--指定 namenode 的 hdfs 协议文件系统的通信地址-->
        <name>fs.defaultFS</name>
        <value>hdfs://master01:9000</value>
    </property>
    <property>
        <!--指定 hadoop 集群存储临时文件的目录-->
        <name>hadoop.tmp.dir</name>
	<value>/home/ubuntu/hadoop/tmp</value>
    </property>
</configuration>
```

### hdfs-site.xml

```xml
<configuration>
<property>
      <!--namenode 节点数据（即元数据）的存放位置，可以指定多个目录实现容错，多个目录用逗号分隔-->
    <name>dfs.namenode.name.dir</name>
    <value>/home/ubuntu/hadoop/namenode/data</value>
</property>
<property>
      <!--datanode 节点数据（即数据块）的存放位置-->
    <name>dfs.datanode.data.dir</name>
    <value>/home/ubuntu/hadoop/datanode/data</value>
</property>
</configuration>

```

### yarn-site.xml

```xml
<configuration>

<!-- Site specific YARN configuration properties -->
    <property>
        <!--配置 NodeManager 上运行的附属服务。需要配置成 mapreduce_shuffle 后才可以在 Yarn 上运行 MapReduce 程序。-->
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <!--resourcemanager 的主机名-->
        <name>yarn.resourcemanager.hostname</name>
        <value>master01</value>
    </property>
</configuration>

```

### mapred-site.xml

```xml
<configuration>
    <property>
        <!--指定 mapreduce 作业运行在 yarn 上-->
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

### slaves

ubuntu@master01:~/hadoop/hadoop-2.10.1/etc/hadoop$ cat slaves 

```
master01
node01
node02
```

## 分发程序

在master01中

```
scp -r /home/ubuntu/hadoop/hadoop-2.10.1 node01:/home/ubuntu/hadoop/
scp -r /home/ubuntu/hadoop/hadoop-2.10.1 node02:/home/ubuntu/hadoop/
```

## 启动集群

```shel
cd /home/ubuntu/hadoop/hadoop-2.10.1/sbin
./start-dfs.sh 
./start-yarn.sh
```

## 查看集群

访问

hdfs

http://master01:50070/dfshealth.html#tab-overview

yarn

http://master01:8088/cluster

