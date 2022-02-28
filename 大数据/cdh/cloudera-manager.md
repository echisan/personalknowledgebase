# cloudera-manager

cm整体架构图

![img](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/v2-0f91b354d49ae47fae3b8766a3522983_720w.jpg)



## 架构

cm的核心是server，server承载了管理员控制台和应用逻辑，负责安装软件、配置、启动、停止服务以及管理运行有服务的集群。

cm的核心是Cloudera Manager Server，包括以下组件

- Server
- Agent
- Management Server
- Database
- API

### Server

管Admin Console Web Server和应用程序逻辑。它负责安装软件，配置，启动和停止服务以及管理运行服务的集群。

### Agent

安装在每台主机上，它负责启动和停止进程，解压缩配置，触发安装和监控主机。

定期发送检测信号（心跳信息）给CM，默认情况下，Agent每隔15秒向Cloudera Manager Server发送一次检测信号。

### Managerment Server

执行各种监控，报警和报告功能的一组角色的服务。

