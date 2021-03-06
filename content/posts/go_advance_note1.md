---
title: "Go进阶笔记-微服务概览与治理"
date: 2020-12-02T00:07:36+08:00
draft: false
author: "fan"
subtitle: "Go进阶笔记-微服务概览与治理"
tags: ["go"]
categories: ["go"]
toc:
  enable: true
  auto: true
---

基本上在产品的最开始阶段，为了快速构建产品，都是单体架构，尽快我们也会按照业务划分模块，但是这个样子始终最终部署的时候还是单体式应用。
如我们早期可以使用Python 的Django快速迭代一个web应用，我们会在Django中划分不同的模块，也就是Django中的app。
而随着业务的迭代发展，项目越来越复杂，可能就会导致应用的扩展，可靠性越来越低，最终导致敏捷开发和自动化部署变得无法完成。



## 微服务定义



关于SOA

> 面向服务的架构（SOA）是一个组件模型，它将应用程序的不同功能单元（称为服务）进行拆分，并通过这些服务之间定义良好的接口和协议联系起来。接口是采用中立的方式进行定义的，它应该独立于实现服务的硬件平台、操作系统和编程语言。这使得构建在各种各样的系统中的服务可以以一种统一和通用的方式进行交互。



所以我们可以把微服务看做是SOA的一种实践：

- 小即是美：小的服务代码少，bug也少，易于测试，易于维护，也更容易不断迭代完善。
- 单一职责：一个服务只需要干好一件事情，专注才能做好。



### 什么是微服务？

围绕业务功能构建的，服务关注单一业务，服务间采用轻量级的通信机制，可以全自动独立部署，可以使用**不同的编程语言和数据存储技术**。微服务架构通过业务拆分实现服务组件化，通过组件组合快速开发系统，业务单一的服务组件又可以独立部署，使整个系统变得清晰灵活。

- 原子服务
- 独立进程
- 隔离部署
- 去中心化服务治理

**注意：基础设施的建设，复杂度高。**

**自己的理解：**

- 简单说就是微小的服务或应用，比如linux上的各种工具：ls,cat,awk等
- 微服务就是让每个小的服务专注的做好一件事
- 每个服务单独开发和部署，服务之间是完全隔离的



### 微服务的优缺点

微服务也不是万金油，并不是所有的情况都需要做成微服务，同时微服务也有自己的缺点或者说微服务也会带来一些问题：

- 微服务应用是分布式系统，因此系统必然会比单体应用的时候复杂：开发者不得不适用**RPC**或者**消息传递**来实现进程间通信；必须要写代码来处理消息传递中速度过慢或者服务不可用等局部失效问题。
- 分区的数据库架构，同时更新多个业务主体的事务很普遍。这种事务对单体式应用来说很容易，因为只有一个数据库。在微服务架构中，需要更新不同服务使用的不同的数据库，从而对开发者提供了更高的要求和挑战。
- 测试一个基于微服务的应用也变的很复杂。
- 服务模块的依赖，应用的升级可能会涉及多个服务模块的修改。



优点：

- 迭代周期短，极大的提升开发效率
- 独立部署，独立开发
- 可伸缩性好，能够针对指定的服务进行伸缩
- 故障隔离，不会相互影响

缺点：

- 复杂度增加，一个请求往往要经过多个服务，请求链路比较长
- 监控和定位问题困难
- 服务管理比较复杂



### 组件化服务

微服务的核心是**组件化服务**，通过将之前复杂的巨石机构，拆分成不同的服务，来实现组件化。即将应用拆散为一系列的服务运行在不同的进程中。单一的服务变化只需要重新部署对应的服务进程。


### 区中心化

- 数据去中心化
- 治理去中心化
- 技术去中心化

**注：治理区中心化，可以理解为消除架构中的热点，例如，我们通常在架构中使用的Nginx，所有的流量都会先经过Nginx,虽然也可以扩容，但是相对来说收益就比较低。**

每个服务独享自身的数据存储设施(缓存，数据库等)，而不是像传统应用共享一个缓存和数据库，这样有利于服务的独立性，隔离相关干扰。

### 基础设施自动化

无自动化不微服务。自动化包括测试和部署。
单一进程的传统应用被拆分为一系列的多进程服务后，意味着开发，调试，测试，监控和部署的复杂度会增加，必须要有合适的自动化基础设施来支持微服务架构，否则开发和运维的成本会大大增加。

- CICD
- Testing
- K8s

**落地微服务的关键因素**

配套设施：

- 微服务框架研发和维护
- 打包，版本管理，上线平台支持
- 硬件层支持，比如容易和容器调度
- 服务治理平台支持，比如分布式链路追踪和监控
- 测试自动化支持，比如上线前自动化case

组织架构

- 微服务框架开发团队
- 私有云研发团队
- 测试平台研发团队



### 硬件层架构

![JHhJAI.png](https://s1.ax1x.com/2020/04/30/JHhJAI.png)



### 可用性 & 兼容性设计

微服务架构采用粗力度的进程间通信。关于可用性和兼容性主要包含以下方面：

- 隔离
- 超时控制
- 负载保护
- 限流
- 降级
- 重试
- 负载均衡

注意：服务的提供者的变更可能引发服务消费者的兼容性破坏，时刻谨记服务契约的兼容性。
总结一句话：发送时要保守，接收时要开放。



## 微服务设计

### API Gateway

常见的开源网关：Kong, APSix,


**面向用户场景的API,而不是面向资源的API**

BFF（Backend for Frontend） 可以认为是一种适配服务，将后端的微服务进行适配（主要包括聚合裁剪和适配逻辑），向无线端设备暴露友好和统一的API，方便无线设备介入访问后端服务。

BFF 可以理解为主要进行数据的组装，业务场景的聚合API

网关在微服务架构中承担着非常重要的角色，它是解偶拆分和后续升级的利器。在网关的配合下，单块BFF 实现解偶拆分，各业务团队可以独立开发和交付各自的微服务。
把跨横切面逻辑从BFF 剥离到网关上，BFF的开发可以更加专注于业务逻辑交付。实现架构上的关注分离。

### Mircoservice划分

相对来说有两种不同不同的划分服务边界：通过业务职能（Business Capability）划分和DDD的限界上下文(Bounded Context)

Business Capability: 由公司内部不同部门提供的只能
Bounded Context：这里的业务边界的含义是“解决不同业务问题”的问题域和对应的解决方案域，为了解决某种类型的业务问题，贴近领域知识，也就是业务。

DDD 通过领域对象之间的交互实现业务逻辑与流程，并通过分层的方式将业务逻辑剥离出来，单独进行维护，从而控制业务本身的复杂度。

**注意：微服务与微服务之间不是通过数据耦合的，所以微服与微服务之间都是通过接口调用，一定不是通过数据，服务与服务之间数据是隔离的。**

#### 什么是CQRS

CQRS — Command Query Responsibility Segregation，故名思义是将 command 与 query 分离的一种模式。

CQRS 将系统中的操作分为两类，即「命令」(Command) 与「查询」(Query)。命令则是对会引起数据发生变化操作的总称，即我们常说的新增，更新，删除这些操作，都是命令。而查询则和字面意思一样，即不会对数据产生变化的操作，只是按照某些条件查找数据。

CQRS 的核心思想是将这两类不同的操作进行分离，然后在两个独立的「服务」中实现。这里的「服务」一般是指两个独立部署的应用。在某些特殊情况下，也可以部署在同一个应用内的不同接口上。

Command 与 Query 对应的数据源也应该是互相独立的，即更新操作在一个数据源，而查询操作在另一个数据源上。


### Mircoservice安全

关于外网的请求，通常在API Gateway进行统一的认证拦截，认证成功后，使用JWT方式通过RPC元数据传递的方式带到BFF层，BFF校验Token完整性后把身份信息注入到应用的Context中，BFF到其他下层的微服务，建议是直接在RPC Request中带入用户身份信息（UserID）请求服务

对于服务内部，一般要区分身份认证和授权

对于身份认证：如果是gRPC，可以很容易进行身份认证，如：证书...
对于授权：通过配置中心做一个RBAC的服务，下发到服务，服务加载的时候就可以很容易构建一个RBAC的认证，从而判断这个请求是否有权限。


## gRPC && 服务发现

- 多语言：语言中立，支持多种语言
- 轻量级，高性能：序列化支持PB(Protocol Buffer) 和JSON, PB是一种语言无关的高性能序列化框架
- 可插拔
- IDL：基于文件定义服务，通过proto3工具生成指定语言的数据结构/服务端接口以及客户端Stub
- 设计理念：如元数据的传递
- 移动端：基于标准的HTTP2设计，支持双向流，消息头压缩,单TCP的多路复用/服务端推送等特性。
- 服务而非对象，消息而非引用：促进微服务的系统间粗粒度消息交互设计理念
- 负载无关的：不同的服务需要使用不同的消息类型和编码
- 流：streaming API
- 阻塞式和非阻塞式：支持异步和同步处理在客户端和服务端交互的消息序列
- 元数据交换：常见的横切关注点，如认证或追踪，依赖数据交换。
- 标准化状态码：客户端通常以有限的方式响应API调用返回的错误

### Health Check

gRPC 有一个标准的健康监测协议，在gRPC的所有语言实现中基本都提供了生成代码合用于设置运行状态的功能。

主动健康检查可以在服务提供者服务不稳定时，被消费者所感知，临时从负载均衡中摘除，减少错误请求。当服务提供这重新稳定后，health check 成功，重新假如到消费者的负载均衡中，回复请求，health check 同样也被用于外挂方式的容器健康检测，或者流量检测


healthCheck 可以做什么 ？

- 在我们的服务注册与发现中，假如服务的提供者Provider到Discoery 之间通信时正常的，但是我们的服务调用者Consumer到服务提供者Provider之间出现网络问题，这个时候如果没有健康检查，我们的服务调用这就会继续调用，但是这个时候其实是会调用失败的，而healthCheck 就可以避免这种情况的发生。它会对从Discoery中获取到的Provider进行健康检查，虽然Discoery中有这个Provider，但是如果健康检查有问题，那么就会把这个provider进行剔除。避免调用失败的问题。

- 平滑发布


### 服务发现 

**CAP原理**

- C: consistency, 一致性，每次总是能够读到最近写入的数据或者失败
- A: available, 每次请求都能读到数据
- P: partition tolerance 分区容忍,不管任意个消息由于网络原因失败,系统都能能够继续工作

CAP原理中，P是必须满足的，C 和A 可以根据业务需要选择，要么是CP系统，要么是AP系统



**客户端发现**

一个服务实例启动时，它的网络地址会被注册到注册中心，当服务实例终止时，再从注册中心删除。这个服务实例的注册表通过心跳机制动态刷新；客户端使用一个负载均衡算法，去选择一个可用的服务实例，来响应这个请求。


**服务端发现**

客户端通过负载均衡器向一个服务发送请求，这个负载均衡器会查询服务注册表，并将请求路由到可用的服务实例上。服务实例在服务注册表上被注册和注销


[![DNhMgH.jpg](https://s3.ax1x.com/2020/11/24/DNhMgH.jpg)](https://imgchr.com/i/DNhMgH)


对比两种服务发现：

- 客户端发现：直连，比服务端服务发现少一次网络跳转，Consumer需要内置特定的服务发现客户端和发现逻辑。
- 服务端发现：Consumer无需关注服务发现具体细节，只需要知道服务的DNS域名即可，支持异构语言开发，需要基础设施支撑，多了一次网络跳转，可能有性能损失。


**注意：微服务的和兴是去中心化，所以相对来说使用客户端服务发现模式比较好**

推荐的服务发现：
https://nacos.io/zh-cn/docs/what-is-nacos.html
https://github.com/bilibili/discovery  学习一下代码


服务发现中的保护机制：

- 如果发现短时间内大量服务提供这下线，会开启自我保护模式。这个时候不会剔除服务。
- 如果服务消费者和服务注册中心通信故障，这个时候本身服务消费者会缓存配置，即使短时间内通信故障也不会有太大影响。



## 多集群 & 多租户

对于特别重要的服务通常是要考虑多级群。

- 从单一集群考虑，多个节点保证可用性，我们通常使用N+2的方式来冗余节点。
- 从单一集群故障带来的影响面角度考虑冗余多套集群。
- 单个机房内的机房故障导致的问题。


多套冗余的集群对应多套独占的缓存，带来更好的性能和冗余能力
尽量避免业务隔离使用或者sharding带来的cache hit影响（按照业务划分集群资源）

但是这里会有一个问题需要考虑：
根据不同的业务划分集群后，如果其中一个业务的进群挂了之后，将流量切到正常集群的时候，这个时候因为独占缓存，所以就会导致产生到两的cache miss 透传到DB，这个时候DB的压力会瞬间变大。


解决办法：可以和所有集群建立连接，通过负载均衡的方式，这样请求就会均摊的打到不同的集群中
上，从而防止缓存击穿的情况。


注意这里还有一个问题：
对于服务中的个别服务可能会存在有大量的其他服务都会依赖这个服务的情况，如帐号服务，那么这个时候health check 的检查可能会占用一定的资源，并且随着规模的增加，光health check 就会占用非常高的资源，如何解决这个问题呢？

是否可以从全集群中选取一批节点（子集），利于划分子集限制连接池大小？

通常20-100个后端，部分场景需要大子集，比如批量读写操作。
后端平均分给客户端。
客户端重启，保持重新均衡，同时对后端重启保持透明，同时连接的变动最小。

**需要思考这个算法的实现。**


### 多租户

> 在一个微服务架构中允许系统共存是利用微服务稳定性及模块化最有效的方式之一。这种方式一般被称为多租户。租户卡一是测试，金丝雀发布，影子系统，甚至服务层或产品线，使用租户能够保证代码的隔离性并且能够基于流量租户做路由决策。

**多租户就是解决RPC的路由或者叫做RPC染色**

并行测试需要一个和生产环境一样的过渡（staging）环境，并且知识用来处理测试流量。在并行测试中，工程师团队首先完成生产服务的一次变动，然后将变动的代码部署到测试栈，这种方法可以在不影响生产环境的情况下让开发者稳定的测试服务，同时能够在发布前更容易的识别和控制bug，尽管并行测试是一种非常有效的集成测试方法，但是它也带来了一些可能影响服务架构成功的挑战：

- 混用环境导致的不可靠测试
- 多套环境带来的硬件成本
- 难以做负载测试，仿真线上真实流量情况


 使用这种方法(内部叫染色发布)，我们可以把待测试的服务 B 在一个隔离的沙盒环境中启动，并且在沙盒环境下可以访问集成环境(UAT) C 和D。我们把测试流量路由到服务 B，同时保持生产流量正常流入到集成服务。服务 B 仅仅处理测试流量而不处理生产流量。另外要确保集成流量不要被测试流量影响。生产中的测试提出了两个基本要求，它们也构成了多租户体系结构的基础：

- 流量路由：能够基于流入栈中的流量类型做路由。
- 隔离性：能够可靠的隔离测试和生产中的资源，这样可以保证对于关键业务微服务没有副作用。

[![DUeeCq.png](https://s3.ax1x.com/2020/11/24/DUeeCq.png)](https://imgchr.com/i/DUeeCq)

这里可以理解为，对于不同的流量区别对待，对于测试的流量，也会在请求的时候带上对应的染色标记，这样到达系统的时候就会根据不同的染色标记走不同的路由，路由到具有相同染色的服务上。



## 小结

- 对于微服整体有一认识
- 对于公司现有系统架构的一些思考，可以跟着课程的深入学习，慢慢对公司现有架构整理出自己的意见和一些可行性的方案

需要关注的书籍与链接：

- https://microservices.io/
- https://nacos.io/zh-cn/docs/what-is-nacos.html
- https://github.com/bilibili/discovery
- https://hellosean1025.github.io/yapi/
- 《UNIX环境高级编程 》
- 《Google SRE》

