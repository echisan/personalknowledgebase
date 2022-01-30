# Envoy

https://mp.weixin.qq.com/s/jY-qpgTQRUBvFrTR8sNLxA?forceh5=1

## Envoy的四个概念

- Listener：监听器负责监听数据端口，接受下游的连接和请求。
- Cluster：集群是对真实后端服务的抽象和封装，管理后端服务连接池、负责后端服务健康检查、实现服务级熔断等等。
- Filter：过滤器主要负责将 Listener 接收的客户端二进制数据包解析为结构化的协议数据，比如 HTTP 二进制流解析为具体的 Header、Body、Trailer、Metadata 诸如此类并进行各种流量治理。
- Route：路由一般是作为某个协议解析 Filter 的一部分存在。筛选器解析出结构化数据后会根据路由中具体规则选择一个 Cluster，最终发数据转发给后端服务。

Envoy 将请求的来源方向称之为下游（Downstream），而请求转发的去向称之为上游（Upstream）。

![image-20211108104451582](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20211108104451582.png)

总结来说：**Downstream** 请求自 **Listener** 进入 Envoy，流经 **Filter** 被解析、修改、记录然后根据 **Route** 选择 **Cluster** 将其发送给 **Upstream** 服务。



## Envoy 关键特性

envoy最核心的三个特性

- 基于xDS协议的动态配置
- 基于核心概念Filter的可扩展性
- 可观察性

### Envoy xDS协议

xDS全称x Discovery Service，x表示某种资源；其他三种资源都有一个对应的 DS 与之关联，分别是 LDS、CDS、RDS：

- 通过 LDS，就可以动态的下发 Listener 的配置，可以在运行时打开新的监听端口，关闭旧的监听端口或者更新某个 Listener 的 Filter 配置等等。
- 通过 CDS，就可以动态控制 Envoy 可以访问哪些服务。比如说，新启动了一个 Service，就可以通过 CDS 向 Envoy 下发一个新的 Cluster。之后，来自客户端的流量就可以通过 Envoy 访问到对应的服务。
- 通过 RDS，可以实现路由规则的动态更新和加载，可以动态的对路由表进行删改和生效。

前面最核心的四种资源之中，Filter 比较特殊，它的配置一般来说都嵌入在 LDS/CDS/RDS 之内。

此外，还有一类关键 DS：EDS。EDS  是对 CDS 的补充。很多时候，服务一旦创建，就不会经常变动。但是服务的实例可能会经常变动，比如 Deploy 滚动更新之类的。EDS，就是用于在 CDS 不变的前提下，动态更新每个 Cluster 后面可用的实例。

![image-20211108110329662](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20211108110329662.png)

当使用 Isito Pilot 作为 xDS Server 时，如何利用 xDS 来动态更新 Envoy 配置。一般情况下，是用户修改了 K8s 集群中的一些 CRD 资源亦或者注册中心有配置更新才会触发配置更新；之后 Pilot watch 到相关变化变更将相关变化抽象成各种 Envoy 中对应的资源，如 Listener、Cluster，然后通过各个 xDS 将对应资源推送到 Envoy。

![image-20211108110631978](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20211108110631978.png)

### Envoy 可扩展性

Envoy 提供了一个 Filter 机制，让开发者可以完全无侵入的在各个层级定义和扩展 Envoy 功能。

总结来说就是：Listener Filter 处理连接、Network Filter 处理二进制数据、L7 Filter 处理解析后结构化数据。

下图是一个相对完整的 Envoy 插件链执行流程图。原本应该还有所谓的 encoder/decoder 层级的，它们才是真正负责协议解析的组件。但是在目前实现当中，encoder/decoder 一般都是嵌在 Network Filter 中作为某个 Network Filter 的一部分，所以这里干脆简化掉了。

![image-20211108111023541](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20211108111023541.png)

### Envoy 可观察性

可观察性是指在软件程序运行过程中，获取软件程序内部状态或者发生的一个能力。如果程序执行起来之后，开发者和运维人员就对内部状态一无所知，那很多问题就根本无法定位。

按照侧重面的不同，Envoy 的可观察性主要依靠三个部分构成。分别是日志、指标、以及分布式追踪。

- 日志：日志是对 Envoy 内部事件（或者直白的说，就是一个请求处理过程）的详细记录。Envoy 提供了非常丰富的日志字段，而且可以灵活的配置。同时还提供了类似 L7 Filter 一样的 Access Log Filter 来自定义日志过滤和筛选、二次修改之类的能力。
- 指标监控数据是对 Envoy 内部事件的数值化统计，本质上，指标数据就是一个个计数器。记录诸如请求次数、正确请求次数、错误请求次数之类的统计数据。指标监控数据要结合 Prometheus 之类的时间数据库来使用，用于观察 Envoy 的整体流量趋势。
- 指标监控和日志描述的都是单个实例内部的状态。但是在微服务架构中，往往不同的服务实例内事件是有关联的。比如服务 A  调用服务 B ，服务 B 又调用了服务 C 和 服务 D。在服务 A B C D 中的 4 个事件是具有因果关系的。分布式跟踪就是为了记录这种因果关系，进行全链路的监控。



