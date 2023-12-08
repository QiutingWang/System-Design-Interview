# System Design Basic Concepts

## HTTPS

- HTTP 的加密版本，如果 data 在半路被拦截，被拦截的 data 呈二进制代码状。
  - TCP 握手，建立 TCP 链接
  - 客户将加密的 TCP 信息发送到服务器
  - 服务器将回复加上 SSL 认证（包含：公钥，host name，expiry date 等信息）
  - 客户将被 public key 加密过的 session 钥匙发送到服务器
  - 服务器接收加密的 session key 并使用 private key 将其解密

## CDN内容交付网络

- Definition：地理位置分散的服务器组成的网络来传递静态内容，改善用户接收传递内容的可靠性和速度。
  - 用户和CDN服务器的物理距离越近，传输速度越快。
- process：
  - 当用户请求网站内容时，定位到距离用户最近的CDN服务器
  - 当cache中已经存在，直接调用缓存中的对应内容
  - 如果cache中还没有对应的缓存，从源服务器或其他附近的CDN服务器(edge)检索调用对应的内容，并存到CDN的cache中
  - 定时检查更新状况
- 如果想在未过expire time之前，删掉CDN里的文件内容的方法：
  - 使用CDN vendor提供的API
  - 设置object的版本的参数，做版本管理
- Push CDN & Pull CDN（主动方不同）
  - 推送式：源服务器将数据推送到CDN
  - 拉取：CDN在用户首次请求时，从源服务器获取数据并缓存

## DNS(Domain Name System)

- address book,将人可以理解的域名-->机器易读的IP地址
- 树状层次结构：不同等级使用.分隔，级别最低写在最左边
  - root name server根域
  - TLD name server顶级域，
    - 分类：
      - 国家 `.cn`,`.us`
      - 国际 `.int`
      - 通用 `.com`,`org`,`.edu`,`.gov`,...共13个
  - Authoritative name server 权威域名。
  - 主机名`www`等
- 长度不可以超过63字节
- 通过resolver解析器与根服务器、TLD各级服务器进行沟通，当然先看cache中是否之前已经解析过。计算机接收到IP地址后，与关联服务器建立连接

## Load Balancer负载均衡器

- definition：将传入的网络流量分配到多个服务器
- 构成：
  - 通过private IP来和多个web server沟通：只能在同一块网络的服务器之间communicate
    - 为什么是多个web server？
      - 为了防止failover issue,go offline
      - 在用户量和request量很大的时候增强web tier availability
      - 帮助消除单点故障
  - DNS<--user--public IP-->load balancer-->private IPs-->servers
- Application: Nginx
- 算法：
  - round robin:按照请求顺序循环地安排在不同server上，每个server轮流接受请求
  - weighted round robin：将每台服务器分配一个权重值，权重越高的服务器，处理请求越多
  - 最小宽带优先
  - 最小连接数优先
  - 最小响应时间优先
- layer4 & layer7
- Active-active & Active-passive

## Data Center

- 大装置，处理商业运作的数据，由多个主机构成，需要空调冷却以及供电防火设备
- 多个数据中心的现实挑战：
  - traffic redirection: GeoDNS将traffic导入正确（距离客户地理位置最近）的数据中心
  - 数据同步：将不同数据中心的数据进行复制，保持一致性
  - 测试和部署：automated deployment tools are needed

## IP(Identity Provider)

## Message Queues

- 支持异步
- 结构：
  - input service[Producer], create message-->[Message Queue]-->[Customer]/subscribers, perform the actions defined by the messages
- 即使 customer 并不 available to process 的时候，producer 依然可以 post message 到队列，vice visa
- 可以根据队列中任务量的大小，安排 worker 的数量
- Application：RabbitMQ，Kafka

## Cache

- cache：临时储存空间来容含expensive responses和经常accessed的数据，以此提高服务的速度
- Cache Tier:
  - layer,比database更快
  - 如果cache中没有存储过要调用的数据，则利用database将新调用的data在cache中进行存储
- Application：分布在多个位置，DNS、CDN、API Gateway、database、Client cache、GPU/CPU
- 使用cache的situation？
  - 常读，不常修改
  - 不适合persisting，因为cache server重启后，存储在cache中的data全部会丢失
  - expiration date不宜设置的太长或太短
    - TTL：time-to-live describes how long the content is cached
  - Eviction policy：
    - 当cache空间爆满时，继续add items to cache会导致之前的被remove掉
    - least-recently-used(LRU)，least-frequently-used(LFU)，FIFO, LIFO, random replacement(RR), most-recently-used(MRU)
- 分类：
  - global cache：所有的nodes使用一个cache space。但是当用户量激增的时候，cache会吃紧
  - distributed cache：使用hash function来分块
- 性能指标：
  - 命中率hit rate: 依赖缓存提供数据而非访问原始来源
  - 未命中率miss rate：1-hit rate
  - 缓存大小cache size：缓存分配的内存，较大的缓存会导致更大的命中率，但是增加了内存负担
  - 延迟latency：从缓存中访问数据需要的时间

## Proxies

- Forward Proxies
- Reverse Proxies

## API Gateway网关

- 功能：
  - 接入HTTP请求，检查黑白名单
  - 限流，指标监控
    - control the rate of traffic sent by a client or a service.限制一段时间内用户请求量。
    - 过量的请求将被拒绝
  - 确认IP的authentication和authorization
  - API请求进入系统的唯一入口,封装了系统内部结构
  - request information from services映射指定将请求路由到具体的服务
  - 日志审查monitoring, logging, tracing
  - combine responses组合改善用户体验
- 常用网关：Ngnix，spring cloud gateway

## 限流

- 目的：防止特殊目的性爬虫、恶意刷单，增强系统对突发流量的应对 将其控制在合理范围内
- 途径：拒绝服务（定向到错误页或告知资源没有了）、排队或等待、降级（返回兜底数据或默认数据）
- 常见算法：
  - Token bucket
    - 参数
      - bucket size：桶可以乘放的最大token数
      - refill rate：每秒token被放入桶中的数量
    - working process：
      - 假设一个request对应1个单位的token，每接收一个token时check一下桶里目前的token量是否>bucket_size
        - 如果还有空间，调用这些令牌，请求通过；
        - 如果没有足够的空间，新的令牌被拒绝request is dropped；
    - evaluation:
      - 方便实施，节约内存，允许一定程度的突发传输
      - 但是调节参数的时候比较难控制
  - Leaking bucket漏斗
    - 特征：request输出的速率是固定的a fixed rate，FIFO队列, request的输入没有限制，所以呈漏斗状
    - working process：request来了check一下queue是不是full：not full-->将request加入到queue中；full-->drop this request新请求被拒绝, 队列中的请求按照一个fixed rate来处理
    - 参数：
      - bucket size：队列的size
      - outflow rate：每秒处理多少个request
    - 不会出现突发流量的状况
  - Fixed window counter
    - working process:
      - 将时间轴分割成大小固定的时间窗，为每个窗设置一个计数器counter
      - 每增加一个request的时候，counter的计数会add 1，同时记录请求时间
      - 当某个窗口内的counter计数>预先设置好的threshold时，新的request将被拒绝，直到新的时间窗开启。
    - evaluation：
      - 当请求量很大的情况下，在时间窗口的开头quota就被快速消耗完，剩下的时间不处理任何请求，不合理
      - 并不能准确地将处理的request数限制在counter预先规定的threshold以内，对边界没有很好的处理
      - 实现简单
  - Sliding window log滑动日志
    - working process:
      - 通过缓存记录request的时间戳
      - 当新的request出现时，remove掉outdated时间戳
        - outdated timestamp: 比当前的时间窗口起点更早的
      - 将新的时间戳记录在日志中
      - 当log size的大小< allowed count-->request accepted
    - 我们可以选择linux或者human-readable representation的时间戳
    - 参数：
      - 每秒允许处理的request数（速率）
    - 评价：限流准确，但是很耗内存enough if the request is denied,它依然被储存在cache中
  - Sliding window counter
    - fixed window counter+sliding window log
    - 通过使用滑动窗口实现对cache需求量的减少
    - 将滑动时间窗再切分

## CAP理论

- 分布式系统中consistency，availability，partition tolerance的权衡
  - C:所有节点的数据一致，任何时刻和节点访问到的数据都是最新的
  - A：任何时刻都能访问到系统
  - P：即使网络分区出现故障(slow network connection, unavailable connection等问题)，系统仍然可以正常运行
- 不可能全部实现3个诉求，最多只能同时实现2个
- 现实中一般P是大部分系统要具备的功能，一般需要在C和A之间取舍

## URL输入浏览器后会发生什么？

- Component：`Scheme://Domain/Path/Resource`
  - scheme：向web server发送连接利用https
  - domain：网站的域名
  - path：从服务器到requested资源的路径
  - resource：想要访问的资源的名称
- process:
  - 在浏览器中输入URL
  - 浏览器在DNS cache中找是否有对应的IP
    - 数据储存在不同位置的layer中：browser/OS/OS/local network/ISP cache
    - 如果并不能找到：通过DNS resolver递归式寻找DNS服务器，直到IP地址被查找到
  - 浏览器与服务器建立TCP连接
  - 浏览器向服务器发送HTTP协议(`GET`)
  - 服务器处理request并将HTTP response送返浏览器
  - 浏览器呈递HTTP内容

### URL Shortener Design

## Horizontal Scaling VS Vertical Scaling

- Vertical Scaling:
  - definition：给一台服务器增加增多的硬件设备RAM、CPU
  - evaluation：比较直接简单,更适合traffic不是很密集的时候；但是不能无限的加硬件设施，并且没有故障转移和备援设备，如果一个设备故障了 整个app、系统都die掉了
- Horizontal Scaling
  - definition：给一个单位的设备组增加更多的服务器，每个server副本处理一部分请求
  - evaluation：具有无限拓展性，减轻单点故障

## Consistent Hashing

- 目的：为了成功achieve horizontal scaling，对请求和数据进行准确地distribute
- 传统哈希方法：
  - 哈希函数：$serverIndex=hash(key)\%N$
    - N: 服务器数量
    - 一次来将keys分布在不同的server上
    - 这个方法适用于the number of servers is fixed，数据可以均匀分配在不同的服务器时
  - 问题：并不能在有新server加入 或 现存server被删除时依旧有效，会造成大量数据迁移，导致通信压力增大，uneven imbalanced load
- 一致性哈希
  - 优点：当server的数量动态变化时，减少数据的迁移量，满足了单调性、负载均衡、分散性
  - concepts：
    - hash空间首尾相连-->hash ring
    - hash server:使用hash function根据IP或者名字将servers映射在哈希环形上
    - hash keys:将数据通过hash算法得到的对应key值映射到hash ring中
    - key按照**顺时针**方向找到对应的第一个临近的server并进行存储
    - add/remove a server:只有一小部分的key需要redistribute到新的server中，而不需要近乎全局迁移
  - 存在的问题：
    - 当hash ring中存在的机器较少时，会出现size of partitions分布不均匀,造成哈希环倾斜，平衡性弱
  - 解决途径：**虚拟节点**(virtual nodes OR replicas)
    - intuition:让节点尽可能的多，使各个server均匀的分布在hash ring中。参照物理节点通过虚拟的方式复制多个，一个机器对应多个virtual nodes
      - eg: $S0_0, S0_1, S0_2$-->都是server0的虚拟节点
    - 这样按照顺时针顺序，找到第一个临近虚拟节点，将key储存在该虚拟节点的对应server节点上
    - 更多的虚拟节点created，更好的balance性，因为data distribution的标准差减少了, representing how data are spread out.但是这需要更多的储存空间在容纳更多的虚拟节点


## TCP(Transmission Control Protocol)传输通信协议

- 最大TCP连接数=numbers of client IPs * 客户端端口数
- 三次握手：确认双方的通信能力和意愿
  - process：
    - 客户端向服务器发送SYN**同步**包，请求建立连接
    - 服务器向客户端发送SYN/ACK**同步/确认**包，确认服务器已经准备好建立连接
    - 客户端向服务器发送ACK确认包，表示客户端也准备好建立连接
  - 标识：
    - FIN:终止连接
    - RST：在发生无法恢复的错误时重置连接
    - SYN：序列号必须要重置
    - ISN：初始序列号
  - 该过程中，client状态变化`CLOSE`-->`SYN_SENT`-->`ESTABLISHED`; server状态变化`CLOSE`-->`LISTEN`-->`SYN_RCVD`-->`ESTABLISHED`
    - linux查看TCP状态：`netstat -napt`
  - 前两次握手不可以携带数据，第三次握手可以携带应用层的数据
  - 三次握手中<u>连接</u>是指什么？
    - Socket(IP地址和端口号)，序列号(用来解决乱序问题)，窗口大小(解决流量控制问题)等新的组合
  - 为什么要进行3次呢？
    - 防止重复历史连接的初始化
      - 如果是两次握手，无法阻止旧的连接
    - 同步双方的序列号
    - 避免资源浪费
- 四次挥手：断开连接并防止数据流失
  - process：
    - 四次挥手的初始状态both client&server都为`ESTABLISHED`
    - client给server发送FIN字段来启动终止进程，客户端进入`FIN_WAIT_1`阶段
    - 服务器向client回复ACK来确认是否结束🔚。客户端进入`FIN_WAIT_2`阶段
    - 服务器完成处理后，向客户端发送FIN字段，进入`LAST_ACK`状态
    - 客户端接收到后，向服务器回复ACK确认字段并进入`TIME_AWAIT`状态。服务器收到后，进入`CLOSED`状态。
    - 等待2MSL最大报文生存时间后，客户端也进入`CLOSED`状态。
  - 四次挥手为什么client向server发送ACK和FIN不能和在一起呢？