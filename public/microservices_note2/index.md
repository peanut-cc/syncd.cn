# 微服务架构




## 微服务架构图



### 从框架层看微服

![Y3Ndu6.png](https://s1.ax1x.com/2020/05/10/Y3Ndu6.png)

### 从程序架构看微服务



![Y3U4Q1.png](https://s1.ax1x.com/2020/05/10/Y3U4Q1.png)



注意：微服务与微服务之间不是通过数据耦合的，所以微服与微服务之间都是通过接口调用，一定不是通过数据，服务与服务之间数据是隔离的。



## 服务注册和发现



分为客户端实现和服务端实现

客户端实现：

- 调用需要维护所有的调度服务地址
- 需要rpc框架支持

服务端实现

- 架构简单
- 有单点故障



订单服务需要调用产品服务，产品服务一共有三个，下面分别是客户端实现和服务端实现的原理



### 客户端实现



![Y30Tcd.png](https://s1.ax1x.com/2020/05/10/Y30Tcd.png)



- 产品服务启动之后注册到服务注册中心
- 订单服务从服务注册中心获取产品服务的列表，从列表中轮寻或随机算则服务并调用



### 服务端实现



![Y3DYd0.png](https://s1.ax1x.com/2020/05/10/Y3DYd0.png)



- 产品服务注册到服务注册中心
- 订单服务需要调用产品服务，先请求LB
- LB从服务注册中心获取产品服务列表，并根据算法选择其中一个返回给订单服务
- 订单服务调用产品服务





### 注册中心

常用的是etcd和consul

etcd注册中心：

- 分布式一致性系统
- 基于raft一致性协议

etcd使用场景：

- 服务注册和发现
- 共享配置
- 分布式锁
- Leader选举



注意：etcd是强一致性的，那么如果etcd节点中有一个出现问题，那么etcd需要重新进行选举，而在这个选举的过程中etcd是不可用的。那么这个时候我们服务就会出现问题。



## Raft协议基本认识



应用场景：

- 解决分布式系统一致性的问题
- 基于复制的



工作机制：

- leader选举
- 日志复制
- 安全性



### 基本概念

角色：

- Fllower角色
- Leader角色
- Candicate角色(候选者)，只存在于选举阶段中的一个概念



Term（任期）概念：

在raft协议中，将时间分成一个个term（任期）

![Y36ikt.png](https://s1.ax1x.com/2020/05/10/Y36ikt.png)



### 复制状态机

![Y32s3V.png](https://s1.ax1x.com/2020/05/10/Y32s3V.png)



一致性算法的工作就是保证复制日志的一致性。 每台服务器上的一致性模块接收来自客户端的命令，并将它们添加到其日志中。 它与其他服务器上的一致性模块（Consensus Module）通信，以确保每个日志最终以相同的顺序包含相同的命令，即使有一些服务器失败。 一旦命令被正确复制，每个服务器上的状态机按日志顺序处理它们，并将输出返回给客户端。 这样就形成了高可用的复制状态机。



### 心跳（heartbeats）和超时机制（timeout）



在Raft算法中，有两个timeout机制来控制领导人选择：

一个是选举定时器（eletion timeout）:即Follower等待成为candidate状态的等待时间，这个时间被随机设定为150ms～300ms之间

另外一个是heartbeat timeout: 在某个节点成为Leader以后，它会发送Append Entries消息给其他节点，这些消息就是通过hearbeat timeout来传送，Follower接收到Leader的心跳包的同时也重置选举定时器



### Leader选举

触发条件：

- 一般情况下，Follower接收到Leader的心跳时，把选举定时器清零，不会触发
- Follower的选举定时器超时发生（比如Leader故障了），会变成候选者，触发Leader选举

选举过程：

- 当定时器到期时，转换为candidate（候选者）角色
- 把当前任期加1,并为自己投票
- 发起RequestVote的RPC请求，要求其他的节点为自己投票
- 如果得到半数以上节点同意，就成为Leader
- 如果选举超时，还没有选出Leader,则进入下一个任期，重新选举



选举过程中限制条件：

每个节点在任期内，最多能给一个候选人投票，采用先到先服务的原则

如果没有投过票，则对比candidate的log和当前节点的log哪个更新，比较方式为：谁的lastLog的term越大谁越新，如果term相同，谁的lastLog的index越大谁越新，如果当前节点更新，则拒绝投票。



### 日志复制

1. Client 向Leader提交命令（如：SET 5）, Leader收到命令后，将命令追加到本地日志中，此时，这个命令处于uncommited状态，复制状态机（state machine）不会执行该命令.
2. Leader将命令（SET 5）并发复制给其他节点，并等待其他节点将命令写入到日志中，如果此时有些节点失败挥着比较慢，Leader节点会一致重试，直到集群中有一半的节点都保存了命令追加到日志中，之后Leader节点就提交命令（即被状态机执行命令，这里是SET 5），并将结果返回给Client节点。
3. Leader节点在提交命令之后，一下次的心跳包中就带有通知其他节点提交命令的消息，其他节点收到Leader的消息后，就会将命令应用到状态机（State Machine），最终每个节点的日志都保持了一致性



### 安全性

- 一个Candidate（候选者）节点要成功赢得选举，就需要和网络中大部分节点进行通信，这就意味着每一条已经提交的日志条目最少在其中一台服务器上出现。如果候选人的日志至少和大多数服务器上的日志一样新，那么它一定包含有全部的已经提交的日志条目，RequestVote RPC实现了这个限制：这个RPC包含候选人的日志信息，如果它自己的日志比候选人的日志要新，那么它会拒绝候选人的投票请求。
- 如果两个日志的任期号不同，任期号大的日志内容更新
- 如果任期号一样大，则根据日志中最后一个命令的索引（index）,谁大谁更新



## RPC调用相关概念



数据传输：

- Thrift协议
- Protobuf协议
- Json协议
- msgpack协议

负载均衡：

- 随机算法
- 轮询
- 一致性hash

异常容错：

- 健康检查
- 熔断
- 限流



## 服务监控相关概念

日志收集：

日志收集端-> kafka集群->数据处理->日志查询报警

Metrics打点：

- 实时采样服务器的运行状态
- web平台直观的报表展示



## 延伸阅读

http://thesecretlivesofdata.com/raft/

https://github.com/maemual/raft-zh_cn/

https://www.jianshu.com/p/aa77c8f4cb5c

https://blog.container-solutions.com/raft-explained-part-33-safety-liveness-guarantees-conclusion

https://raft.github.io/raft.pdf

http://www.yuxiumin.com/2017/08/12/raft-protocol-intro/
























