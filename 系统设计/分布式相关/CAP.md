
* [CAP理论](#cap理论)
  * [一致性 Consistency](#一致性-consistency)
  * [可用性 Availability](#可用性-availability)
  * [分区容错性 Partition Tolerance](#分区容错性-partition-tolerance)
  * [权衡](#权衡)
  * [常见的注册中心](#常见的注册中心)

# CAP理论

CAP 定理指出对于一个分布式系统来说，最多只能同时满足以下三点中的两个：

- **一致性（Consistency）**：所有节点在同一时刻看到相同的数据
- **可用性（Availability）**：每个请求都能在有限时间内得到响应
- **分区容错性（Partition Tolerance）**：系统在网络分区故障时仍能正常服务

CAP 理论中分区容错性 P 是一定要满足的（网络分区不可避免），因此实际上是在可用性 A 和一致性 C 之间做权衡。

### 一致性 Consistency

一致性就是读写数据必须是一致的。例如一条数据分别存在两个服务器中，通过 server1 将数据修改为 b 后，访问 server1 得到 b，访问 server2 如果还返回 a 则不符合一致性，返回 b 才符合。

### 可用性 Availability

只要向服务器发送请求，服务器必须进行响应，保证服务器一直可用。

### 分区容错性 Partition Tolerance

分布式系统分布在多个位置（如北京、上海），可能出现网络无法通信的情况，这就是分区。分区容错是不可避免的，P 必然存在。

### 权衡

在分布式系统中，分区容忍性必不可少（总是假设网络不可靠），因此实际要在可用性和一致性之间权衡：

- **CP 系统**：为了保证一致性，不能访问未同步完成的节点，失去部分可用性
- **AP 系统**：为了保证可用性，允许读取所有节点的数据，但数据可能不一致

可用性和一致性往往是冲突的，很难同时满足。

### 常见的注册中心

| |Nacos|Eureka|Consul|CoreDNS|Zookeeper|
|---|---|---|---|---|---|
|一致性协议|	CP+AP|	AP|	CP|	— |	CP|
|健康检查|	TCP/HTTP/MYSQL/Client Beat|	Client Beat|	TCP/HTTP/gRPC/Cmd|	—	|Keep Alive|
|负载均衡策略|	权重/metadata/Selector|	Ribbon|	Fabio|	RoundRobin|	— |
|雪崩保护|	|有	|有	|无	|无	|无|
|自动注销实例|	|支持	|支持	|支持	|不支持	|支持|
|访问协议|	HTTP/DNS|	HTTP|	HTTP/DNS|	DNS	|TCP|
|监听支持|	支持|	支持|	支持	|不支持|	支持|
|多数据中心|	支持|	支持	|支持	|不支持|不支持|
|跨注册中心同步|	支持|	不支持|	支持|	不支持|	不支持|
|SpringCloud集成|	支持|	支持|	支持|	不支持|	支持|
|Dubbo集成|	支持|	不支持|	支持|	不支持|	支持|
|K8S集成|	支持|	不支持|	支持|	支持|	不支持|
