---
title: "Kafka学习笔记（一）---基础概念"
date: 2021-03-08T13:43:32+08:00
draft: false
author: "fan"
subtitle: "kafka笔记"
tags: ["Kafka"]
categories: ["Kafka"]
toc:
  enable: true
  auto: true
---

Kafka最重要的作用----削峰填谷

## 基本概念

发布订阅的对象是：主题（Topic）

向主题发布消息的客户端应用程序是：生产者（Producer）

订阅主题消息的客户端应用程序是：消费者（Consumer）


kafka 的服务器端由被称为Broker的服务进程构成，即一个Kafka集群由多个Broker组成，Broker负责接收和处理客户端发送过来的请求，以及对消息进行持久化。

实现高可用的手段之一就是备份机制（Replication）, 就是把相同的数据拷贝到多台机器上，而这些相同的数据拷贝在Kafka中被称为副本（Replica）

Kafka定义了两类副本：领导者副本（Leader Replica）和追随者副本（Follower Replica）。前者对外提供服务，即与客户端程序进行交互；后者被动地追随领导者副本，不能与外界进行交互。

副本的工作机制：生产者总是向领导者副本写消息；消费者总是从领导者副本读消息。
追随者副本：向领导者副本发送请求，请求领导者把最新生产的消息发送给它，从而与领导者副本保持同步。


Kafka 中的分区机制是将每个主题划分成多个分区（Partition）, 每个分区是一组有序的消息日志。生产者生产的每条消息只会被发送到一个分区中，即如果向一个双分区的主题发送一条日志，该日志要么在分区0，要么在分区1中。
kafka的分区编号从0开始。


副本是在分区这个层级定义的，每个分区下可以配置若干个副本，其中只能有1个领导者副本和N-1个追随者副本。生产者向分区写入消息，每条消息在分区中的位置信息由一个叫位移（Offset）的数据来表示，分区位移总是从0开始，假设一个生产者向一个空分区插入了10条消息，那么这10条消息的位移一次是0，1，2，3.....9


每个消费者在消费消息的过程中需要有个字段记录它当前消费到了分区的哪个位置上，这个字段就是消费者位移（Consumer offset）


Kafka设计之初就是想要提供三个方面特性：
- 提供一套API实现生产者和消费者
- 降低网络传输和磁盘存储开销
- 实现高伸缩性架构


Kafka客户端底层是使用了Java的selector, selector 在linux上实现机制epoll, 而在Windows平台上实现机制是select


对于Kafka 磁盘容量规划考虑如下：
- 新增消息数
- 消息留存时间
- 平均消息大小
- 备份数
- 是否启用压缩



## Kafka配置参数

这里只是先列举相对重要的参数

`Brokeer`端参数：

`log.dirs`: 指定了 `Broker` 需要 使用的若干个文件目录路径。`log.dirs`配置 多个路径，具体格式是一个 `CSV` 格式，也就是用逗号分隔 的多个路径，比 如`/home/kafka1,/home/kafka2,/home/kafka3`
`listeners`: 监听器，其实就是告诉外部连接者 要通过什么协议访问指定主机名和端口开放的 Kafka 服 务。
`advertised.listeners`: 是说这组监听器是 Broker 用于对外发布的

Topic管理的相关参数：

`auto.create.topics.enable`:是否允许自动创建 Topic。参数建议最好设置成 false，即不允许自动创建 Topic。

`unclean.leader.election.enable`:是否允许 Unclean Leader 选举。建议把它设置成 false 

`auto.leader.rebalance.enable`:是否允许定期进 行 Leader 选举。设置它的值
为 true 表示允许 Kafka 定期地对一些 Topic 分区进行 Leader 重选举。建议在生产环境中把这 个参数设置成 false。




Zookeeper相关参数：

Zookeeper 是一个分布式协调框架，负责协调管理并保 存 Kafka 集群的所有元数据信息,如：Broker 在运行、创建了哪些 Topic，每个 Topic 都有多少 分区以及这些分区的 Leader 副本都在哪些机器上等信息。

Kafka 与 ZooKeeper 相关的最重要参数 `zookeeper.connect`,这也是一个 CSV 格式的参数，如：zk1:2181,zk2:2181,zk3:2181。2181 是 ZooKeeper 的默认端口。

如果你有两套 Kafka 集群，假设分别叫它们 `kafka1` 和 `kafka2`，那么两套集群的`zookeeper.connect`参数可以 这样指定:`zk1:2181,zk2:2181,zk3:2181/kafka1`和 `zk1:2181,zk2:2181,zk3:2181/kafka2`。切记 `chroot` 只需要写一次，而且是加到最后的。


关于数据留存方面参数：

`log.retention.{hour|minutes|ms}`:这是个“三 兄弟”，都是控制一条消息数据被保存多长时间。从优先 级上来说 ms 设置最高、minutes 次之、hour 最低。虽然 ms 设置有最高的优先级，但是 通常情况下我们还是设置 hour 级别的多一些，比如 log.retention.hour=168表示默认保存 7 天的数据。

`log.retention.bytes`:这是指定 Broker 为消息保存 的总磁盘容量大小。这个值默认是 -1， 表明你想在这台 Broker 上保存多少数据都可以，至少在容 量方面 Broker 绝对为你开绿灯，不会做任何阻拦。

`message.max.bytes`:控制 Broker 能够接收的最大消 息大小。默认的 1000012 太少了，还不到 1MB。实际场景中突 破 1MB 的消息都是屡见不鲜的，因此在线上环境中设置一 个比较大的值还是比较保险的做法。毕竟它只是一个标尺而 已，仅仅衡量 Broker 能够处理的最大消息大小，即使设置 大一点也不会耗费什么磁盘空间的。

Topic级别的参数：
如果为所有 Topic 的数据都保存相 当长的时间，这样做既不高效也无必要。如果只能设置全局 Broker 参数，那么势必要提取所有 业务留存时间的最大值作为全局参数值，此时设置 Topic 级 别参数把它覆盖，就是一个不错的选择。

`retention.ms`:规定了该 Topic 消息被保存的时长。 默认是 7 天，即该 Topic 只保存最近 7 天的消息。一旦 设置了这个值，它会覆盖掉 Broker 端的全局参数值。

`retention.bytes`:规定了要为该 Topic 预留多大的磁 盘空间。和全局参数作用相似，这个值通常在多租户的

Kafka 集群中会有用武之地。当前默认值是 -1，表示可以 无限使用磁盘空间。
`max.message.bytes`: 它决定了 Kafka Broker 能够正常 接收该 Topic 的最大消息大小。


JVM参数：
一个通用的建议：将你的 JVM 堆大小设置成 6GB 吧，这是目前业界比较公认的一个合理值。
`KAFKA_HEAP_OPTS`：指定堆大小。
`KAFKA_JVM_PERFORMANCE_OPTS`：指定 GC 参数。

即在启动 Kafka Broker 之前，先设置上这两个环境变量：

```
$> export KAFKA_HEAP_OPTS=--Xms6g  --Xmx6g
$> export  KAFKA_JVM_PERFORMANCE_OPTS= -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true
$> bin/kafka-server-start.sh config/server.properties
```
提交时间或者说是 Flush 落盘时间,默认是 5 秒。一般情况下我们会认为这个时间太频繁了，可以适当地增加提交间隔来降低物理磁盘的写操作。




**注意: Broker 端和 Client 端应用配置中最好全部使用主机名。 **


### 怎么设置 Topic 级别参数

有两种方式可以设置：
- 创建 Topic 时进行设置
- 修改 Topic 时设置

max.message.bytes举例。设想你的部门需要将交易数据发送到 Kafka 进行处理，需要保存最近半年的交易数据，同时这些数据很大，通常都有几 MB，但一般不会超过 5MB。现在让我们用以下命令来创建 Topic：
`bin/kafka-topics.sh--bootstrap-serverlocalhost:9092--create--topictransaction--partitions1--replication-factor1--configretention.ms=15552000000--configmax.message.bytes=5242880`

请注意结尾处的--config设置，我们就是在 config 后面指定了想要设置的 Topic 级别参数。

kafka-configs来修改 Topic 级别参数:
`bin/kafka-configs.sh--zookeeperlocalhost:2181--entity-typetopics--entity-nametransaction--alter--add-configmax.message.bytes=10485760`

建议是，你最好始终坚持使用第二种方式来设置，并且在未来，Kafka 社区很有可能统一使用kafka-configs脚本来调整 Topic 级别参数。





