## 配置

### flink-conf.yaml

jobmanager 堆内存

taskmanager, 总内存，堆外内存。

真实运行内存：taskmanager > jobmanager

几个插槽，几个线程

`taskmanager.numberOfTaskSlots: 1`

并行度，与numberOfTaskSlots的区别，`numberOfTaskSlots` 针对一个taskmanger最大的并行能力

`parallelism.default: 1`

## Flink运行架构

### Flink运行时的组件

- 作业管理器（JobManager）

控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的 JobManager 所控制执行。

1. JobManager 会先接收到要执行的应用程序，这个应用程序会包括： 作业图（JobGraph）、逻辑数据流图（logical dataflow graph）和打包了所有的类、库和其它 资源的 JAR 包。
2. JobManager 会把 JobGraph 转换成一个物理层面的数据流图，这个图被叫做 “执行图”（ExecutionGraph），包含了所有可以并发执行的任务
3. JobManager 会向资源管 理器（ResourceManager）请求执行任务必要的资源，也就是任务管理器（TaskManager）上 的插槽（slot）
4. 一旦它获取到了足够的资源，就会将执行图分发到真正运行它们的 TaskManager 上
5. 在运行过程中，JobManager 会负责所有需要中央协调的操作，比如说检 查点（checkpoints）的协调。

- 资源管理器（ResourceManager）

主要负责管理任务管理器（TaskManager）的插槽（slot），TaskManger 插槽是 Flink 中 定义的处理资源单元。

当 JobManager 申请插槽资源时，ResourceManager 会将有空闲插槽的 TaskManager 分配给 JobManager。如果 ResourceManager 没有足够的插槽 来满足 JobManager 的请求，它还可以向资源提供平台发起会话，以提供启动 TaskManager 进程的容器。另外，ResourceManager 还负责终止空闲的 TaskManager，释放计算资源。

- 任务管理器（TaskManager）

Flink 中的工作进程。通常在 Flink 中会有多个 TaskManager 运行，每一个 TaskManager都包含了一定数量的插槽（slots）。

插槽的数量限制了 TaskManager 能够执行的任务数量。

1. 启动之后，TaskManager 会向资源管理器注册它的插槽；
2. 收到资源管理器的指令后， TaskManager 就会将一个或者多个插槽提供给 JobManager 调用
3. JobManager 就可以向插槽 分配任务（tasks）来执行了
4. 在执行过程中，一个 TaskManager 可以跟其它运行同一应用程 序的 TaskManager 交换数据。

- 分发器（Dispatcher）

可以跨作业运行，它为应用提交提供了 REST 接口。

当一个应用被提交执行时，分发器 就会启动并将应用移交给一个 JobManager。

Dispatcher 也会启动一个 Web UI，用 来方便地展示和监控作业执行的信息。Dispatcher 在架构中可能并不是必需的，这取决于应 用提交运行的方式。

### 任务提交流程

![image-20220302095731451](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302095731451.png)

将Flink部署到YARN上的提交流程：

![image-20220302100032192](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302100032192.png)

### 任务调度原理

![image-20220302100414755](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302100414755.png)

客户端不是运行时和程序执行 的一部分，但它用于准备并发送 dataflow(JobGraph)给 Master(JobManager)，然后，客户端断开连接或者维持连接以 等待接收计算结果。

1. 当 Flink 集 群 启 动 后 ， 首 先 会 启 动 一 个 JobManger 和一个或多个的 TaskManager。
2. 由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。
3. TaskManager 之间以流的形式进行数据的传输。
4. Client、JobManager、TaskManager均为独立的JVM进程。

### TaskManager与Slots

Flink 中每一个 worker(TaskManager)都是一个 JVM 进程，它可能会在独立的线 程上执行一个或多个 subtask。为了控制一个 worker 能接收多少个 task，worker 通 过 task slot 来进行控制（一个 worker 至少有一个 task slot）。

每个 task slot 表示 TaskManager 拥有资源的一个固定大小的子集。假如一个 TaskManager 有三个 slot，那么它会将其管理的内存分成三份给各个 slot。

### 执行图（ExecutionGraph）

由 Flink 程序直接映射成的数据流图是 StreamGraph，也被称为逻辑流图，因为 它们表示的是计算逻辑的高级视图

为了执行一个流处理程序，Flink 需要将逻辑流 图转换为物理数据流图（也叫执行图）

Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图。

- StreamGraph：是根据用户通过 Stream API 编写的代码生成的最初的图。用 来表示程序的拓扑结构。
- JobGraph：StreamGraph 经过优化后生成了 JobGraph，提交给 JobManager 的 数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这 样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。
- ExecutionGraph ： JobManager 根 据 JobGraph 生 成 ExecutionGraph 。 ExecutionGraph 是 JobGraph 的并行化版本，是调度层最核心的数据结构。
- 物理执行图：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个 TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。

### Connect 与 Union 区别

1. Union 之前两个流的类型必须是一样，Connect 可以不一样，在之后的 coMap 中再去调整成为一样的。
2. Connect 只能操作两个流，Union 可以操作多个。

# Window

## window类型

- CountWindow： 按照指定的数据条数生成一个 Window，与时间无关。
- TimeWindow： 按照时间生成 Window。
  - 滚动窗口（Tumbling Window）
  - 滑动窗口 （Sliding Window）
  - 会话窗口 （Session Window）

### 滚动窗口（Tumbling Windows）

将数据依据固定的窗口长度对数据进行切片

**特点：时间对齐，窗口长度固定，没有重叠。**

滚动窗口分配器将每个元素分配到一个指定窗口大小的窗口中，滚动窗口有一 个固定的大小，并且不会出现重叠。

![image-20220302140035867](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302140035867.png)

### 滑动窗口 （SlidingWindows）

滑动窗口是固定窗口的更广义的一种形式，滑动窗口由固定的窗口长度和滑动 间隔组成

**特点：时间对齐，窗口长度固定，可以有重叠。**

滑动窗口分配器将元素分配到固定长度的窗口中，与滚动窗口类似，窗口的大 小由窗口大小参数来配置，另一个窗口滑动参数控制滑动窗口开始的频率。

滑动窗口如果滑动参数小于窗口大小的话，窗口是可以重叠的，在这种情况下元素 会被分配到多个窗口中。

![image-20220302140229094](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302140229094.png)

适用场景：对最近一个时间段内的统计（求某接口最近 5min 的失败率来决定是 否要报警）。

### 会话窗口 （Session Windows）

由一系列事件组合一个指定时间长度的 timeout 间隙组成，类似于 web 应用的 session，也就是一段时间没有接收到新数据就会生成新的窗口。

**特点：时间无对齐。**

session 窗口分配器通过 session 活动来对元素进行分组，session 窗口跟滚动窗 口和滑动窗口相比，不会有重叠和固定的开始时间和结束时间的情况，相反，当它 在一个固定的时间周期内不再收到元素，即非活动间隔产生，那个这个窗口就会关 闭。一个 session 窗口通过一个 session 间隔来配置，这个 session 间隔定义了非活跃 周期的长度，当这个非活跃周期产生，那么当前的 session 将关闭并且后续的元素将 被分配到新的 session 窗口中去。

![image-20220302140627662](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220302140627662.png)

### CountWindow

CountWindow 根据窗口中相同 key 元素的数量来触发执行，执行时只计算元素 数量达到窗口大小的 key 对应的结果。

 注意：CountWindow 的 window_size 指的是相同 Key 的元素的个数，不是输入 的所有元素的总数。

## 时间语义与 Wartermark

### Flink 中的时间语义

- Event Time：是事件创建的时间。它通常由事件中的时间戳描述，例如采集的 日志数据中，每一条日志都会记录自己的生成时间，Flink 通过时间戳分配器访问事 件时间戳。
- Ingestion Time：是数据进入 Flink 的时间。
- Processing Time：是每一个执行基于时间操作的算子的本地系统时间，与机器 相关，默认的时间属性就是 Processing Time。



### watermark

- 数据流中的 Watermark 用于表示 timestamp 小于 Watermark 的数据，都已经 到达了，因此，window 的执行也是由 Watermark 触发的。 

-  Watermark 可以理解成一个延迟触发机制，我们可以设置 Watermark 的延时 时长 t，每次系统会校验已经到达的数据中最大的 maxEventTime，然后认定 eventTime 小于 maxEventTime - t 的所有数据都已经到达，如果有窗口的停止时间等于 maxEventTime – t，那么这个窗口被触发执行。

