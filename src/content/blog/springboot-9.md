---
title: Spring Boot（九）：注册中心Eureka-服务端视角
author: 杰哥
pubDatetime: 2023-08-29T16:08:06.130Z
postSlug: spring-boot-8
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - 注册中心
  - Eureka

description: 在微服务架构中，涉及到多个服务之间的调用关系，同一类型服务还经常会以集群的形式存在，那么，面对一个项目中诸多微服务，我们需要如何去管理呢？
---

在微服务架构中，涉及到多个服务之间的调用关系，同一类型服务还经常会以集群的形式存在，那么，面对一个项目中诸多微服务，我们需要如何去管理呢？

Spring Cloud 框架针对这种广泛的分布式场景，提供了一套比较成熟的管理框架，用于服务治理，包括服务注册、熔断、降级、限流以及负载均衡等。我们将带领大家逐渐步入微服务架构。

那么从今天开始，我们就进入其中一个比较重要的环节-服务注册

之前有一位读者提了个建议，说是可以把一个知识点分成两到三篇来写，这样不仅我不会太费劲，大家看起来也不用太费劲。
我一想，有一定的道理啊，毕竟一门技术，只写一篇的话，不仅容易讲得太泛，不够细，而且自己还累死累活的，到头来也凸显不出重点，得不偿失。所以以后我就打算这样干啦~感谢这位读者的建议~

关于注册中心，本人将献上 zookeeper、eureka、consul 以及 nacos 这四个常用的注册中心，而对于目前初步计划是每个注册中心以服务端和客户端两个角度分别写两篇，应该也会有一两次的总结文章，敬请期待哦~

首先，先一起进入 eureka 的学习吧~

# 一 原理:认识 Eureka

## 01. 什么是注册中心？

注册中心实际上是微服务架构中各个微服务的协调者，服务提供方和服务调用方均会进行注册。它会向客户端提供可供调用的服务列表，客户端在进行远程服务调用时，根据服务列表，选择服务提供方的服务地址进行服务调用。

## 02. 什么是 Eureka？

Eureka 是 Netflix 开发的服务发现框架，本身是一个基于 REST 的服务，主要用于定位运行在 AWS 域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。SpringCloud 将它集成在其子项目 spring-cloud-netflix 中，以实现 SpringCloud 的服务发现功能

如果是集群模式，其各个节点之间是平级关系。在正常情况下，在刚启动时以及客户端服务列表发生变化时，会进行某个节点的服务状态列表的更新，并复制同步给其他节点，来保持各节点服务列表的一致性

Eureka 包含两个组件：**Eureka Server** 和 **Eureka Client**

## 03. Eureka 有哪些角色？

看看 Eureka 的简单架构图

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/1.png)

总得来说，Eureka 注册中心实际上分为三个角色

- 1 **注册中心服务端**

图片

1）启动时从其他节点拉取服务

2）客户端执行注册、续约以及取消时，会向其他节点同步

3）运行过程中，定期运行剔除任务

图片

- **2 服务提供者**

图片

1）启动时注册服务

2）运行过程中：续约

3）停止时，cancel

图片

- **3 服务消费者**

图片
1）启动时，拉取服务

2）运行过程中：续约

3）服务调用时，发起远程调用

图片

本次就主要进行讲解注册中心服务端的三个主要动作的原理，关于客户端的动作以及客户端状态变化时服务端具体的变化，我们将再下一篇呈现，敬请期待哦~

## 04. 心跳机制

默认情况下，如果 Eureka Server 在一定时间内没有接收到某个微服务实例的心跳，Eureka Server 将会注销该实例（默认 90 秒）

这种机制确实能够根据定期获取并更新微服务实例状态，但这样，又会存在一个问题。。。

大家想一想，当网络分区故障发生时，微服务与 Eureka Server 之间突然无法正常通信了，根据心跳机制，微服务将会被注销。那么这种心跳机制是不是就变得不太友好了？因为这种情况下微服务本身其实是健康的，本不应该注销这个微服务

那么这种情况该如何解决呢？Eureka 通过自我保护机制来解决这个问题

## 05. 自我保护机制

当 Eureka Server 节点在短时间内丢失过多客户端（可能发生了网络分区故障），默认是 15 分钟内收到的续约低于原来的 85% 时，这个节点就会进入自我保护模式

一旦进入该模式，Eureka Server 仍能接收新服务的注册和查询请求，但是不会同步到其他节点上；同时也会保护服务注册表中的信息，不再移除注册列表中因为长时间没收到心跳而应该过期的服务。当网络故障恢复后，该 Eureka Server 节点会自动退出自我保护模式

这样获取的数据的确有可能不是最新的，但 Eureka 的这种自我保护机制，极大地保证了 Eureka 的高可用特性

# 三 实战 实战环节

## 01. 单机实现

由于 Spring Boot 已经集成了 Eureka,所以要实现注册中心比较简单，分为以下三步

**1）引入依赖**

只需要在 pom 中添加 eureka-server 依赖即可

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

**2）添加配置**

在 application.properties 中添加如下配置：

```
server.port=8761
#只做自己--注册中心
#1、不注册自己（不做服务提供者）
eureka.client.register-with-eureka=false
#2、不获取配置（不做消费者）
eureka.client.fetch-registry=false
```

指定注册中心的端口为 8761

需要注意的是，eureka.client.register-with-eureka，eureka.client.fetch-registry 这两个配置项分别表示不像提供者一样注册自己，也不像消费者一样去获取服务，它只做它纯粹的自己

**3）添加注解**

在启动类中添加 @EnableEurekaServer，来声明该微服务为注册中心

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**4）启动服务**

直接启动启动类，通过查看日志，我们发现服务已经以 8761 端口正常启动

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/2-正常启动.png)

**5）访问页面**

访问地址 http://localhost:8761/，可以看到服务注册中心的管理端页面

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/3-.png)

2.  集群实现

在实际生产环境中，对于注册中心，我们实际上是会以集群的方式搭建的。那么，如何搭建一个集群式的 eureka 注册中心呢？其实只要再添加几个上述类似单个节点，然后节点之间相互注册就可以了

两个节点的话，很好实现，只需要相互注册就可以了，为了说明根本实现方式，我们直接来看三个节点的情况

**1）再新增两个工程**

在单节点 eureka-server 的基础上，再分别克隆两个工程 eureka-server1、eureka-server2

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/4-server2.png)

**2）修改配置文件**

分别修改一下两个新增节点的端口为 8762、8763

配置各个节点的注册中心，分别配置为其他两个节点的地址，用逗号分隔

节点 1 配置变为

```
server.port=8761
#只做自己--注册中心
#1、不注册自己（不做服务提供者）
eureka.client.register-with-eureka=false
#2、不获取配置（不做消费者）
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8762/eureka/,

http://localhost:8763/eureka/
```

节点 2 配置

```
server.port=8762
#只做自己--注册中心
#1、不注册自己（不做服务提供者）
eureka.client.register-with-eureka=false
#2、不获取配置（不做消费者）
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/,

http://localhost:8763/eureka/
```

节点 3 配置

```
server.port=8763
#只做自己--注册中心
#1、不注册自己（不做服务提供者）
eureka.client.register-with-eureka=false
#2、不获取配置（不做消费者）
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/,

http://localhost:8762/eureka/
```

**3）启动各节点服务**

分别启动三个注册中心服务，我们可以看到任意一个服务，都分别进行了添加另外两个节点的动作

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/5-的动作.png)

**4）访问页面**

此时，重新访问页面 http://localhost:8761/，便可以看到其他两个节点已成功注册在该注册中心，访问其他两个页面，也是如此显示

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/8-如此显示.png)

那么，到现在我们就成功搭建好了 eureka-3 节点的注册中心，相信通过这个搭建过程，大家对于 eureka 也有了一定的了解了。

这个时候趁热打铁，跟踪源码，探究它的内部实现，实在再好不过~

# 三 源码 源码探究

接下来，我们跟着源码探究一下注册中心的真实面目

## 01. 启动时拉取服务

启动时注册中心将会从其他节点拉取服务

**1）初始化参数**

通服务启动时，进入 DiscoveryClient，进行相关 Bean 的初始化，包括健康检查处理 bean、监控检查回调 bean、事件监听 bean 以及注册前处理 bean 等

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/9-bean等.png)

**2）初始化节点以及注册信息**

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/10-注册信息.png)

进入集群节点启动方法：peerEurekaNodes.start()方法，该方法通过定义一个守护线程，来根据节点 url，分别进行：

初始化集群节点信息

初始化固定周期，更新集群节点信息的任务

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/11-的任务.png)

定期更新频率默认值为 10 分钟，可以通过 eureka.server.peerEurekaNodesUpdateIntervalMs 配置

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/12-配置.png)

**3）获取节点**

Eureka Server 在启动后会调用 EurekaClientConfig.getEurekaServerServiceUrls 来获取所有的 Peer 节点（当然上面也说到了，它也会定期更新）。

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/13-peer节点.png)

‍

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/13-2.png)

我们启动 eureka-sever3，观察其获得的 deFaultZone 中的节点信息为 http://localhost:8761/ 和 http://localhost:8762/，这是因为我们在配置文件中配置了这两个注册中心节点

```
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/,

http://localhost:8762/eureka/
```

**4）更新节点信息**

通过 PeerEurekaNodes.updatePeerEurekaNodes() 更新集群节点信息

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/14-节点信息.png)

根据获取到的节点 url 列表，分别进行删除不可用节点，添加新节点来进行最新节点列表的更新

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/15-的更新.png)

最后进入类 PeerAwareInstanceRegistryImpl，该类通过实现 PeerAwareInstanceRegistry 的 init() 方法，更新最终的集群节点列表，并将其注册到 JMX 监控器上，进行节点状态变更监控

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/16-变更监控.png)

**5）同步注册信息**

调用 PeerAwareInstanceRegistryImpl.syncUp() 方法，从集群的一个 Eureka-Server 节点获取初始注册信息，代码如下：

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/17-代码如下.png)

## 02. 同步服务状态变化

当客户端状态发生变化时，服务端会监听到这些变化，并执行注册、续约以及取消操作，同时会向其他节点同步

当我们启动一个客户端时，会调用 PeerAwareInstanceRegistryImpl.register() 进行服务的注册，并调用 replicateToPeers()方法复制给其他节点

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/18-其他节点.png)

过程中，会调用 renew() 方法，进行续约，默认周期为

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/19-周期为.png)

当我关闭客户端服务时，则会进入 cancel() 方法，进行服务的关闭

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/20-的关闭.png)

本篇主要想让大家对于 eureka 的机制有一个全局意识，因此这块我们只需要知道，当客户端启动、关闭时，服务端会实时监听到这些状态的变化，通过分别调用 register() 、renew() 以及 cancel() 方法进行服务状态列表的更新以及各节点的同步即可。

具体的逻辑，我们就安排在下一篇

## 03. 定期剔除服务

在运行过程中，定期运行剔除任务。那么这个周期是多久呢？

同样通过查看 eureka-server 的配置类 EurekaServerConfigBean，发现配置项 evictionIntervalTimerInMs 默认为 1 分钟，当然也可以通过配置修改

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/21-配置修改.png)

每隔固定周期，eureka 就会调用 AbstractInstanceRegistry.evict()，进行不可用服务的剔除。

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/22-的剔除.png)

这个方法首先会先判断自我保护机制是否开启

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/23-是否开启.png)

跟踪方法 isLeaseExpirationEnabled（），我们发现当每分钟心跳次数 renewsLastMin 小于 numberOfRenewsPerMinThreshold 时，并且开启自动保护模式开关 eureka.enableSelfPreservation = true 时，就会触发自动保护机制，不再自动过期租约

否则就会进入过期服务剔除操作，主要依据就是超过一定时间没有 renew 的服务将会被剔除掉

# 四 总结 总而言之

今天主要聊了聊作为注册中心 Eureka 的角色之一注册中心服务端

分别从初步认识，到单机模式与集群模式的实战操作，再到站在注册中心角度的部分源码探究，带领大家对于 Eureka 有了一个初步认识

不得不说，由于 Spring Boot 的支持，一切技术都变得那么简单，容易上手

想起刚开始工作的时候，我的导师对我说的一句话：你只要知道，技术只会越来越简单，所以不用去害怕它。现在越来越觉得，的确如此呢~

eureka 通过一套监听机制，分别对服务进行注册，状态变更同步以及服务定期检查剔除等主要操作

对了，之前有人问过我怎样学着去看源码，比如说想了解 Eureka 的整体机制或者 eureka 对于某个功能的处理逻辑等。在这里呢，告诉大家一个小技巧，那就是看日志，根据日志进行定位主要位置，然后通过打断点配合编辑器中的一些查看某个方法的实现呀以及查看某个方法在哪里被使用啦这些操作，去一步步定位就可以啦~

这个就是我当时初探 Eureka 单节点原理，第一步看的东西

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp9/24.png)

看完文章，我想你肯定掌握了以下几点：

1、了解 Eureka 是什么，心跳机制以及自我保护机制

2、如何实现 Eureka 单机和集群节点服务端

3、Eureka 服务端各功能实现原理

4、自己看源码的方式

嗯，就这样。每天学习一点，时间会见证你的强大~

下期预告：

Spring Boot(十)：注册中心 Eureka-客户端视角

本期项目代码已上传到 github~有需要的可以参考

https://github.com/wangjie0919/Spring-Boot-Notes

往期精彩回顾

[`Spring Boot（七）`：你不能不知道的 Mybatis 缓存机制！](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483890&idx=1&sn=ceb10a1619ef4270e8a8d38b1e91af25&chksm=96eb03e3a19c8af5b5846f72b578ee9fdb207603d5658a79d721a97e456ee1f0b9ca1e946a61&scene=21#wechat_redirect)

[`Spring Boot`（六）：Spring Boot（六）：那些好用的数据库连接池们](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483845&idx=1&sn=b604205c54cbf132a0a3a0d9fd035bba&chksm=96eb03d4a19c8ac207645109ed636d89d781daa27bf5e1290393f04ef45aa9453073d4fa9c76&token=1019255373&lang=zh_CN#rd)

[`Spring Boot`（五）：春眠不觉晓，Mybatis 知多少](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483821&idx=1&sn=7c5ed87610b93589266106d1b07191d1&chksm=96eb03bca19c8aaa5967520fe03381481020450834e7b4e518dd729dc38b71c4cb50154d7e8d&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（四）：让人又爱又恨的 JPA](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483799&idx=1&sn=f0a553daf412aefeb2f7877970b04aca&chksm=96eb0386a19c8a90c15b1dae13d39546a2d01875d3f3489c2e2f709a97f8f8ca87bee4ec58e0&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（三）：操作数据库-Spring JDBC](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483779&idx=1&sn=6ce5bbd2d8028b176ecf3b1b7fa8f3ea&chksm=96eb0392a19c8a84f1e7356a36012a2ef2f17f4d70f60c83ea42c1e72ed5a561c18d1782577e&scene=21#wechat_redirect)

[`Spring Boot`（二）：第一个`Spring Boot`项目](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483751&idx=1&sn=90bb32f21dbfd6793b899ac4cb04d52f&chksm=96eb0376a19c8a60eb9ea0b8227b3e34d55efb0feb6e1aa944d90514333f2be70635b8bc820c&scene=21#wechat_redirect)

[`Spring Boot`（一）：特性概览](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483725&idx=1&sn=c1399fcb817d5a434f272b93abd36798&chksm=96eb035ca19c8a4ac715cf62f4cfe301e0124e7b800c87b3ae78503190bbb8ec5e2754ce3fd4&scene=21#wechat_redirect)
