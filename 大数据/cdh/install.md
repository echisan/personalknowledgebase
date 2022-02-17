# cdh installation

安装CDH 6.3.4

>  As of February 1, 2021, all downloads of CDH and Cloudera Manager require a username and password and use a modified URL.

参考：

https://docs.cloudera.com/documentation/enterprise/6/latest/topics/installation.html

https://docs.cloudera.com/documentation/enterprise/6/latest/topics/install_cm_cdh.html#cmig_topic_6_6

## 安装之前的准备

### 存储空间规划

规划Cloudera Manager Server和Cloudera Management Service用于存储指标和数据的存储需求和数据存储位置

如果未能规划所有组件的存储需求可能会产生以下影响：

- 集群可能无法保留历史运营数据以满足内部要求。
- 集群可能会错过未收集或保留所需时间长度的关键审计信息。
- 管理员可能无法研究过去的事件或健康状况。
- 管理员在以后需要参考或报告时可能没有历史 MR1、YARN 或 Impala 使用数据。
- 指标收集和图表可能存在差距。
- 由于将存储位置填充到 100% 的容量，集群可能会遇到数据丢失。此类事件的影响可能会影响许多其他组件。

> 您必须将每台主机的关键数据存储位置告知您的操作人员，以便他们能够充分配置您的基础架构并对其进行适当备份。

#### Cloudera Manager Server

| 配置主题                                            | Cloudera Manager 服务器配置                                  |
| --------------------------------------------------- | ------------------------------------------------------------ |
| 默认存储位置                                        | 关系型数据库：任意受支持的RMDBS【mysql5.7支持，default for Ubuntu 16.04, 18.04 LTS】 |
| Storage Configuration Defaults, Minimum, or Maximum | There are no direct storage defaults relevant to this entity. |
| 在哪里控制数据保留或大小                            | Cloudera Manager Server 数据库的大小取决于托管主机的数量和已在集群中运行的离散命令的数量。要在 Cloudera Manager 管理控制台中配置保留命令结果的大小。<br />select **Administration** > **Settings** and edit the following property: <br />**Command Eviction Age** <br />Length of time after which inactive commands are evicted from the database.默认两年 |
| 规模、规划和最佳实践                                | Cloudera Manager Server 数据库是 Cloudera Manager 部署中最重要的配置存储。此数据库包含集群、服务、角色的配置以及定义 Cloudera Manager 及其托管主机部署的其他必要信息。<br/>确保对 Cloudera Manager Server 数据库执行定期、经过验证的远程存储备份。 |
|                                                     |                                                              |

#### Cloudera Management Service

| 配置主题                     | 活动监视器                                                   |
| ---------------------------- | ------------------------------------------------------------ |
| 默认存储位置                 | 任何受支持的 RDBMS。                                         |
| 存储配置默认值/最小值/最大值 | 默认值：14 天的 MapReduce (MRv1) 作业/任务                   |
| 在哪里控制数据保留或大小     | 可以通过配置要保留的数据天数或小时数来控制 Activity Monitor 存储使用情况。旧数据被清除。<br />要在 Cloudera Manager 管理控制台中配置数据保留：<br />1. 转到 Cloudera 管理服务。<br />2. 单击配置选项卡。<br />3. 选择Scope > Activity Monitor 或 Cloudera Management Service (Service-Wide)。<br />4. 选择 Category > Main.<br />5. 找到以下属性或通过在搜索框中键入属性名称来搜索它们：<br />**Purge Activities Data at This Age**<br />在 Activity Monitor 中，清除有关 MapReduce 作业的数据并在数据达到此年龄（以小时为单位）时聚合活动。默认情况下，活动监视器将有关活动的数据保留 336 小时（14 天）。<br />**Purge Attempts Data at This Age**<br />在活动监视器中，当数据达到此年龄（以小时为单位）时，清除有关 MapReduce 尝试的数据。由于尝试数据会占用大量数据库空间，因此您可能希望比活动数据更频繁地清除它。默认情况下，活动监视器会将有关尝试的数据保留 336 小时（14 天）。<br />**Purge MapReduce Service Data at This Age**<br />要保留在活动监视器数据库中的过去服务级别数据的小时数，例如运行的总时隙。默认设置为将数据保留 336 小时（14 天）。<br />6. 输入更改原因，然后单击保存更改以提交更改。 |
| 规模、规划和最佳实践         | Activity Monitor 只监控 MapReduce 作业，不监控 YARN 应用程序。如果您不再在集群中使用 MapReduce (MRv1)，则 Cloudera Manager 或 CDH 不需要活动监视器。<br />14 天的 MapReduce 活动所需的存储空间量可能会有很大差异，这直接取决于集群的大小和使用 MapReduce 的活动级别。当您确定集群中 MapReduce 活动的“稳定状态”和“突发状态”时，可能需要调整和重新调整存储量。<br />例如，考虑以下测试集群和用法：<br />一个模拟的 1000 台主机集群，每台主机有 32 个插槽<br />每个活动（作业）有 200 次尝试（任务）的 MapReduce 作业<br />此集群的大小观察：<br />每次尝试需要 10 分钟才能完成。<br />这种使用导致每天大约 20,000 个工作，总尝试次数约为 500 万。<br />对于 7 天的保留期，此活动监视器数据库需要 200 GB。 |



#### Cloudera Management Service - Service Monitor Configuration

| 配置主题                     | 服务监视器配置                                               |
| ---------------------------- | ------------------------------------------------------------ |
| 默认存储位置                 | 在配置成Service Monitor 角色的机器上，`/var/lib/cloudera-service-monitor/` |
| 存储配置默认值/最小值/最大值 | 10 GiB 服务时间序列存储<br/>1 GiB Impala 查询存储<br/>1 GiB YARN 应用程序存储<br/>总计：最小约 12 GiB（无最大值） |
|                              |                                                              |
|                              |                                                              |
|                              |                                                              |





