# System Design Basic Concepts

## HTTPS

- HTTP 的加密版本，如果 data 在半路被拦截，被拦截的 data 呈二进制代码状。
  - TCP 挥手，建立 TCP 链接
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

## Data Center

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
- Strategies：
  - Read
    - Cache aside
    - read through
  - Write
    - write around
    - write back
    - write through

## Proxies
