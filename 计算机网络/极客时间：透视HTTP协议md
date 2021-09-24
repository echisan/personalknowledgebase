# 极客时间：透视HTTP协议

- HTTP是什么

  - 协议
    - 存在多个参与者
    - 对参与者行为约定和规范
  - 传输
    - 双向
    - 请求方和响应方中间允许有若干“中间人”
  - 超文本
    - 文字、图片、音频和视频等
    - “超链接”， 可形成复杂的非线性、网状的结构关系
  - 总结
    - HTTP是一个在计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定和规范

- 代理

  - 匿名代理， 外界看到的只是代理服务器
  - 透明代理， CDN
  - 正向代理
  - 反向代理， CDN

- TCP/IP

  - 应用层(application layer)
    - HTTP、Telnet、SSH、FTP、SMTP
  - 传输层(transport layer)
    - TCP
    - UDP
  - 网际层或者网络互联层(internet layer)
    - IP协议在此层
    - IP地址取代MAC地址
  - 链接层(link layer)
  - 物理层

- OSI（开放式系统互联通信参考模型）

  - 七层模型
    - 应用层：面向具体的应用传输数据
    - 表示层：把数据转换为合适、可理解的语法和语义
    - 会话层：维护网络中的连接状态，保持会话和同步
    - 传输层
    - 网络层
    - 数据链路层
    - 物理层

- 负载均衡

  - 四层负载均衡（工作在传输层上，基于tcp/ip协议）
  - 七层负载均衡（工作在应用层，基于http协议）

- DNS

  - 域名的解析
    - 根域名服务器(Root DNS Server)：管理顶级域名服务器，返回"com","cn"等顶级域名的服务器IP地址
    - 顶级域名服务器(Top-level DNS Server): 管理各自域名下的权威域名服务器，比如com顶级域名服务器可以返回apple.com域名服务器IP地址
    - 权威域名服务器(Authoritative DNS)：管理自己域名下主机的IP地址，比如apple.com权威域名服务器可以返回www.apple.com的IP地址
  - DNS缓存
    - “非权威域名服务器“作为用户DNS查询的代理，可以缓存之前的查询结果
      - 这类的DNS服务器提供商有很多，比如谷歌的"8.8.8.8"或者CloudFlare的"1.1.1.1"
    - 操作系统DNS缓存
    - hosts映射文件
  - Nginx配置DNS服务器: resolver 8.8.8.8 valid=30s;
  - 恶意DNS
    - 域名屏蔽：直接不解析域名，返回错误
    - 域名劫持/域名污染：访问A网站，DNS给了B网站

- 键入网址按下回车，后面会发生什么？

- 状态码

  - 1xx：提示信息，是协议处理的中间状态
    - 101： Switching Protocols，客户端试用Upgrade头字段，要求再HTTP协议的基础上改成其他的协议继续通信，比如websocket
  - 2xx：成功处理请求
    - 206：Partial Content，http分块下载断点续传的基础
  - 3xx：重定向
    - 301：永久重定向
    - 302：临时重定向
    - 304：not modified，缓存重定向
  - 4xx:  客户端发送的请求报文有误
  - 5xx：服务器端的错误码

- HTTP特点

  - 灵活可扩展
  - 可靠传输
  - 应用层协议
  - 请求-应答
  - 无状态
    - TCP是有状态的，开始处于CLOSED，连接成功后是ESTABLISHED状态，断开连接后是FIN_WAIT状态，最后又是CLOSED状态。
    - HTTP协议无规定任何状态，每个请求都是互相独立、毫无关联的，协议不要求客户端或服务器记录请求相关的信息。

- 短连接

  - 在HTTP(0.9/1.0)的时候，每次请求需要先与服务器建立连接，收到响应报文后会立即关闭连接。

- 长连接

  - “持久连接”(persistent connections)，“连接保活”(keep alive)，“连接复用”(connection reuse)
  - HTTP/1.1连接默认开启长连接，只要向服务器发送了第一次请求，后续的请求都会重复利用第一次打开的TCP连接
    - Connection: keep-alive
  - 但是如果大量空闲长连接只连不发，会耗尽服务器资源。需要适当的时间关闭，不能永远保持连接。
    - 客户端： 请求头加上Connection: close。服务器看到这个字段，知道客户端主动关闭连接，响应报文也加上这个字段，发送之后调用SocketApi关闭TCP连接
    - 服务端通常不会主动关闭连接，但也可以试用一些策略，比如nginx
      - keepalive_timeout，设置长连接的超时时间，如果在一段时间内连接没有任何数据收发就主动断开连接
      - keepalive_requests，设置长连接上可发送的最大请求次数。当达到次数后会主动断开连接。

- 队头阻塞(Head-of line blocking)

  - 请求先进先出，队首的请求因为处理的太慢耽搁了时间，队列后面的请求则不得不跟着一起等待

  - 性能优化

    - 因为“请求-应答”模型不能变，队头阻塞的问题在HTTP/1.1无法解决，只能缓解

    - **并发连接** （concurrent connections），同时对一个域名发起多个长连接，用数量来解决质量的问题

    - 但是并发数太高也不行，连接太多服务器资源扛不住，或者被服务器认为恶意攻击造成拒绝服务。

      - RFC7230建议每个客户端最多并发2个连接，但实践证明这个数字太小了，众多浏览器把这个上限提高到了6~8个

    - 域名分片

       （domain sharding）

      - http协议和浏览器限制并发数量，可以多开几个域名，比如shard1.test.com,shard2.test.com，而这些域名都指向同一台服务器，这样实际上连接的数量也上去了

- **拥塞控制**

- Cookie

  - 有效期，使用Expires和Max-Age两个属性来设置

    - 两个属性可以同时设置，但浏览器会优先采用Max-Age计算失效期

  - 作用域，Domain和Path指定Cookie所属的域名和路径

    - 浏览器发送Cookie前会提取URI中取出host和path部分，对比Cookie属性，如果不满足则不会发送cookie

  - 安全性

    - 在js脚本可以用document.cookie来读写数据，带来安全隐患，可能导致跨站脚本（XSS）攻击窃取数据

    - **HttpOnly** 告诉浏览器，此Cookie只能通过浏览器Http协议传输，禁止其他方式访问，浏览器js引擎会禁用document.cookie等一切相关的API

    - SameSite

       可以防范“跨站请求伪造”（XSRF）攻击

      - SameSite=Strict，严格限定Cookie不能随着跳转链接跨站发送
      - SameSite=Lax，允许GET、HEAD等安全方法，但禁止POST跨站发送

    - **Secure** 表示Cookie仅能用HTTPS协议加密传输

- 缓存

  - Cache-Control
    - max-age，表示生存时间，类似TTL， 时间的计算起点是响应报文的创建时间
      - no-store, 不允许缓存
      - no-cache, 可以缓存，但使用之前必须要去服务器验证是否过期
      - must-revalidate: 如果缓存不过期可以继续使用，过期了想用必须去服务器验证
      - 浏览器在刷新的时候，会在请求头里加一个”Cache-Control：max-age=0“

- 代理服务

  - 作用
    - 负载均衡
    - 健康检查，使用心跳等机制监控后端服务器
    - 安全防护，限制ip地址或流量，保护后端服务器
    - 加密卸载，对外网使用ssl/tls加密通信，内网不加密，消除加密成本
    - 数据过滤
    - 内容缓存
  - 代理相关头字段
    - X-Forwarded-For，每经过一个代理节点会在字段里追加一个信息
    - X-Real-IP，xforwardfor的简化版，仅记录客户端ip地址

- HTTPS

  - SSL/TLS
  - TLS1.2握手
    - 子协议
      - 记录协议（Record Protocol）规定TLS收发数据得基本单位：记录（record）；所有其他子协议都需要通过记录协议发出。多个记录数据可以在一个TCP包一次性发出
      - 警报协议（Alert Protocol）向对方发出警报信息。
      - 握手协议（Handshake Protocol）
        - ClientHello： 随机数、客户端版本、支持的密码套件
        - ServerHello：版本号、随机数、从客户端列表选一个作为本次通信使用的密码套件
        - ServerCertificate：发送证书
        - [Server Key Exchange]：里面是公钥Server Params，用来实现密钥交换算法，再加上自己的私钥签名认证
        - ServerHelloDone
          - 此时客户端对证书进行验证，验证完成继续往下走
        - [Client Key Exchange]：客户端也生成一个公钥(Client Params)
        - Change Cipher Spec
        - Finish
        - Change Cipher Spce
        - Finish
      - 变更密码规范协议（Change Cipher Spec Protocol）：就是通知对方后续的数据都将使用加密保护
  - 会话恢复
    - SessionID
    - SessionTicket
      - 类似于HTTP的Cookie，存储的责任由服务器转移到客户端，服务器加密会话信息，用”New Session Ticket“消息发送给客户端，让客户端保存。重连的时候，客户端使用扩展”session_ticket“发送Ticket，服务端解密后验证有效期就可以恢复会话。
        - SessionTicket方案需要使用一个固定的密钥文件(ticket_key)来加密ticket，为了防止密钥被破解，保证”向前安全“，密钥文件需要定期轮换，比如设置为一小时或者一天。
    - PSK，发送Ticket的同时带上应用数据
    - **SessionID和SessionTicket两种会话复用技术已在TLS1.3中被废除，只能使用PSK实现会话复用**

- tls1.3

  - 兼容
    - 1.3使用1.2的格式，使用新的扩展协议
      - supported_version
      - supported_groups
      - key_share
      - signature_algorithms
      - server_name
  - 强化安全
    - tls1.3只保留AES、ChaCha20对称加密算法
    - 分组模式只能用AEAE的GCM、CCM和Poly1305
    - 摘要算法只能用SHA256、SHA384
    - 密钥交换算法只有ECDHE和DHE，椭圆曲线也只剩P-256和x25519等5种
  - 提升性能
    - 减少了密码套件，删除了key exchange消息
    - client hello
      - supported_groups带上支持的曲线
      - key_share带上曲线对应的客户端公钥参数
      - signature_algorithms带上签名算法

- https优化

  - 硬件优化
    - 选择更快的cpu
    - ”ssl加速卡“专门的硬件来做非对称加解密
    - ”SSL加速服务器“
  - 软件优化
    - 升级软件版本
  - 协议优化
    - 尽量采用tls1.3
    - 采用椭圆曲线的ECDHE，支持”False Start“ （需要向前安全的算法）
      - falseStart；TLS协商第二阶段，浏览器发送ChangeCipherSpec和Finished后，立即发送加密的应用层数据，而无需等待服务端确认。
    - 选择高性能的曲线，最好是x25519，次优选择P-256
    - Nginx可以使用"ssl_ciphers"、”ssl_ecdh_curve“选择密码套件
  - 证书优化
    - 证书传输
      - 服务器证书选择椭圆曲线，224位的ECC相当于2048位的RSA
    - 证书验证
      - OCSP Stapling（需要服务端开启该功能，避免客户端访问CA去验证证书）
  - 会话复用
    - sessionId
  - 会话票证
    - sessionTicket
  - 预共享密钥（Pre-sharedKey PSK）tls1.3实现0-RTT
    - 发送Ticket的同时会带上应用数据（EarlyData）
    - PSK不是完美的，容易受到”重放攻击“(Replay attack)，黑客可以截获PSK数据，向复读机那样反复向服务器发送。
      - 解决方法：仅允许安全的GET/HEAD方法，消息里加入时间戳、nonce验证，或者一次性票证限制重放

- HTTP/2

  - HTTP/2把HTTP分解成”语义“和”语法“两个部分。
    - 语义层不做改动，与HTTP/1完全一致，比如请求方法、URI、状态码、头字段等概念都保留不变，这样就消除了再学习的成本，基于 HTTP 的上层应用也不需要做任何修改，可以无缝转换到 HTTP/2。
  - 头部压缩
    - 没有使用传统的压缩算法，开发了专门的”**HPACK**“算法，在客户端和服务器两端建立”字典“，用索引号表示重复的字符串，还釆用哈夫曼编码来压缩整数和字符串，可以达到 50%~90% 的高压缩率。
  - 二进制格式
    - 把原来的”Header+Body“的消息打散为数个小片的二进制”帧“(Frame)，用Headers帧存放头数据，Data帧存放实体数据
    - 消息的”碎片“到达目的地后应该怎么组装起来呢？
      - HTTP/2定了一个流的概念，它是二进制帧的双向传输序列，同一个消息往返的帧会分配一个唯一的流ID。
      - 因为流是虚拟的，实际并不存在，所以可以在一个TCP连接上用流同时发送多个碎片化的消息，这就是常说的**“多路复用”（ Multiplexing）**——多个往返通信都复用一个连接来处理。
    - 为了更好地利用连接，加大吞吐量，HTTP/2 还添加了一些控制帧来管理虚拟的“流”，实现了优先级和流量控制，这些特性也和 TCP 协议非常相似。
    - 可以新建流主动向客户端发送消息

  ![image-20210924093601700](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210924093601700.png)

  - 强化安全

    - 主流浏览器只支持加密的HTTP/2

  - 协议栈

    - 建立在HPack、Stream、TLS1.2基础之上

  - HTTP/2协议

    - TLS握手成功之后，客户端必须要发送一个”

      连接前言

      “，用来确认建立HTTP/2连接

      - 报文格式：PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n

    - 头部压缩

      - HPack算法，需要客户端、服务端各自维护一份索引表，压缩和解压缩就是查表和更新表的操作。
      - 动态表，只有Key没有Value或者自定义字段添加到静态表后面，结构相同，但会在编码解码的时候随时更新

    - 二进制帧

      - 报头只有9字节。
      - 二进制格式
        - Frame Header
          - 帧长度（24bits=3bytes）
          - 帧类型（8bits=1byte）
            - 数据帧（HEADERS帧和DATA帧 存放HTTP报文）
            - 控制帧（SETTINGS,PING,PRIORITY等用来管理流）
          - 标志位（8bits=1byte）
            - 可以保存8个标志位。常用的标志位有END_HEADERS表示头数据结束，相当于 HTTP/1 里头后的空行（“\r\n”），END_STREAM表示单方向数据发送结束（即 EOS，End of Stream），相当于 HTTP/1 里 Chunked 分块结束标志（“0\r\n\r\n”）。
          - [R]流标识符(31bits=4byte)
            - 最高位保留不用
            - 报文头里最后 4 个字节是流标识符，也就是帧所属的“流”，接收方使用它就可以从乱序的帧里识别出具有相同流 ID 的帧序列，按顺序组装起来就实现了虚拟的“流”。
        - Frame Payload

    - 流和多路复用

      - 在 HTTP/2 连接上，虽然帧是乱序收发的，但只要它们都拥有相同的流 ID，就都属于一个流，而且在这个流里帧不是无序的，而是有着严格的先后顺序。

      - HTTP/2流的特点

        1. 流可以并发，一个连接可以同时发出多个流传输数据

        2. 客户端和服务端都可以创建流，双方互不干扰

        3. 流是双向的

        4. 流之间没有固定关系，彼此独立，但流内部的帧是有严格顺序

        5. 流可以设置优先级

        6. 流ID不能重用，只能顺序递增。

           客户端发起的ID是奇数，服务端发起的ID是偶数；

           - ID达到上限，可以发送一个控制帧"GOAWAY"，真正关闭TCP连接

        7. 流上发送RST_STREAM帧可以随时终止流，取消接收或发送

        8. 0号流比较特殊，不能关闭，也不能发送数据帧，只能发送控制帧，用于流量控制

    - 流状态转换

      - 最开始流都是空闲（idle）状态
      - 客户端发送Headers帧后，有了流ID，流进入“打开”状态，两端都可以收发数据
      - 客户端发送“END_STREAM”标志位的帧，流进入了“半关闭”状态
      - 服务端响应数据发完之后，也要带上“END_STREAM”标志位，表示数据发送完毕；这样两端都进入“关闭”状态，流结束
      - 流ID不能重用，流的生命周期是HTTP/1里的一次完整的“请求-应答”，流关闭就是一次通信结束
      - 下一次再发请求就是开一个新流（而不是新连接），流ID达到上限，发送GOAWAY再开一个新的TCP连接，流ID又重新计数
      - http2请求发生再流身上，而不是实际的tcp连接，又因为流可以并发，所以http2可以实现无阻塞的多路复用

    ![image-20210924093542884](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210924093542884.png)

    - 为什么多流就可以做到多路复用呢？TCP不是只有一条连接吗？
    - http/2的队头阻塞
      - http2虽然使用“帧”，“流”，“多路复用”，没有了“队头阻塞”，但这些手段都是在应用层里；而在下层，tcp协议里，还是会发生队头阻塞
        - 从协议栈的角度看，tcp会将http2的流拆成更小的包（segment）依次发送
        - TCP为了保证可靠传输，有“丢包重传”机制，丢失的包必须要等待重新传输确认，即使其他的包已经收到了，也只能放在缓冲区里，上层应用拿不出来。

  - HTTP/3

    - QUIC协议

      是一个传输层协议，和TCP是平级的。引入了类似HTTP/2的流和多路复用，单个流是有序的，可能因为丢包而阻塞，但其他流不会受到影响。

      QUIC不建立在TLS之上，内部包含了TLS。使用自己的帧接管了TLS里的记录，握手消息、警报消息都不使用TLS记录，直接封装成QUIC的帧发送，省掉了一次开销；

      ![image-20210924093527633](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210924093527633.png)

    - QUIC内部细节

      - QUIC的基本传输单位是包（Packet）和帧（frame），一个包由多个帧组成，“包”面向的是“连接”，帧面向的是“流”。
      - QUIC使用不透明的“**连接ID**”来标记通信的两个端点，客户端和服务端可以自行选择一组ID来标记自己，消除了TCP里连接对“IP+端口”的强绑定，支持“**连接迁移**”

      ![image-20210924093516887](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210924093516887.png)

      - QUIC的帧有多种类型，PING，ACK等帧用于管理连接，而STREAM帧专门用来实现流；
      - QUIC的流分为双向流和单向流；
      - 流标识符（流ID）做多8字节；但是保留了最低两位用作标志；所以流ID最多可用62位；
        - 第一位标记流的发起者，0表示客户端，1表示服务端
        - 第二位标记流的方向，0表示双向流，1表示单向流
        - 依上所述，客户端的ID是偶数，从0开始计数；跟HTTP2刚好相反

      ![image-20210924093503254](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210924093503254.png)

    - HTTP/3协议

      - http3仍然使用流用来发送“请求-响应”，但是是直接使用QUIC的流

      - http3的双向流完成可以对应到http2的流，而单向流在http3里用来实现控制和推送，近似地对应http2的0号流

      - http3里的帧结构，帧头只有两个类型：类型和长度；同样采用变长编码，最小只需要两个字节；

      - 头部压缩算法从HPACK升级成

        QPACK

        - 很http2一样，也分为静态表跟动态表
        - 流上发送HEADERS帧时不能更新字段，只能引用；索引表的更新需要在专门的单向流上发送指令来管理，解决了HPACK的队头阻塞的问题

      ![image-20210924093445666](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20210924093445666.png)

    - HTTP/3服务发现

      - 没有指定默认的端口号
      - 使用HTTP/2里的扩展帧；浏览器需要先使用HTTP2协议连接服务器，然后服务器在启动http2连接后发送一个“**Alt-Svc**”帧，包含一个“h3=host:port”的字符串，告诉浏览器在另一个端点上提供等价的HTTP3服务
      - 浏览器受到“**Alt-Svc**”帧，会使用QUIC异步连接指定的端口，如果连接成功，就会断开HTTP2连接，改用HTTP3收发数据

