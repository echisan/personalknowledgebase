# quickstart



```
wget https://dlcdn.apache.org/flink/flink-1.14.3/flink-1.14.3-bin-scala_2.12.tgz
tar -zxvf flink-1.14.3-bin-scala_2.12.tgz

```



Flink 附带了一个 bash 脚本，可以用于启动本地集群

```
$ ./bin/start-cluster.sh
Starting cluster.
Starting standalonesession daemon on host.
Starting taskexecutor daemon on host.
```

webui: http://localhost:8081

停止集群

```
./bin/stop-cluster.sh
```

