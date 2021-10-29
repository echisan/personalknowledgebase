# Zookeeper

## install

```shell
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz
```

### zoo.cfg

```
tickTIme=2000
dataDir=/home/ubuntu/zookeeper/data
clientPort=2181
initLimit=5
syncLimit=5
server.1=master01:2888:3888
server.2=node01:2888:3888
server.3=node02:2888:3888
```

再dataDir目录创建一个`myid`的文件，把id写进去

启动

```
./bin/zkServer.sh --config conf start
```

