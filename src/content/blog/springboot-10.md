---
title: Spring Boot（十）：注册中心Eureka-客户端视角
author: 杰哥
pubDatetime: 2023-08-30T16:08:06.130Z
postSlug: spring-boot-9
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - 注册中心
  - Eureka

description: 这一节我们将站在 Eureka-Client 的角度，分别通过跟踪注册、续约以及下线分别在 Eureka-Client、Eureka-Server 的处理流程以及交互处理流程，让大家对于 Eureka 的整体架构有更深入的了解
---

大家好，今天呢，正式进入注册中心-Eureka 篇章第二节内容的学习

上一节我们从 Eureka-Server 的视角，分别了解了如何实现单机和集群的 Euerka 注册中心，了解了 Eureka-Server 对于服务实例的获取、更新、以及定时剔除等处理流程

那么这一节我们将站在 Eureka-Client 的角度，分别通过跟踪注册、续约以及下线分别在 Eureka-Client、Eureka-Server 的处理流程以及交互处理流程，让大家对于 Eureka 的整体架构有更深入的了解

首先，我们先通过一个例子，看看客户端要如何使用注册中心~

# 一 实战 客户端注册

**1）引入依赖**

只需要在 pom 中添加 eureka-client 依赖即可，当然也要同时引入 Spring-cloud 的依赖，注意版本需要对应

```
<properties>
   <java.version>1.8</java.version>
   <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
</properties>
<!-- 使用eureka作为注册中心-->
      <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
      </dependency>
<!-- 引入Spring-cloud依赖，以保证可以找到eureka的依赖-->
<dependencyManagement>
   <dependencies>
      <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-dependencies</artifactId>
         <version>${spring-cloud.version}</version>
         <type>pom</type>
         <scope>import</scope>
      </dependency>
   </dependencies>
</dependencyManagement>
```

**2）添加配置**

在 bootstrap.properties 中添加如下配置：

```
spring.application.name=waiter-service
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/,
http://localhost:8762/eureka/,http://localhost:8763/eureka/
```

这里我们演示的是注册中心为集群模式的配置方式，配置方式为各个节点之间以逗号隔开即可

eureka.client.serviceUrl.defaultZone 默认值为

http://localhost:8761/eureka/

因此，若只需要配置本地的单机模式（端口也为默认端口 8761 时），那么只需要引入依赖，Spring Boot 就帮你配置好注册中心的地址啦

**3）添加注解**

在启动类中添加@EnableDiscoveryClient，来声明该微服务为注册中心

```
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
}
}
```

**4）启动服务**

直接启动启动类，通过查看日志，我们发现服务已经成功注册

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/1-test.png)

**5）访问页面**

访问地址 http://localhost:8761/，可以看到服务注册中心的管理端页面已有一个实例 waiter-service 已经成功注册，并且状态是 UP

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/2-up.png)

好了，了解了如何使用 Spring Boot 实现客户端使用注册中心，接下来让我们一起通过跟踪源码，站在客户端的角度，探究注册中心的实现机制吧~

# 二 初始化 启动时初始化

Eureka-Client 启动时，进 入 DiscoveryClient，进行相关 Bean 以及定时任务的初始化，包括健康检查处理 bean、监控检查回调 bean、事件监听 bean 以及注册前处理 bean 等

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/3-bean等.png)

调用 initScheduledTasks() 方法进行定时任务的初始化

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/4-初始化.png)

分别初始化心跳检测定时任务、实例信息复制定时任务

其中，实例信息复制定时任务用于实例信息注册；而心跳检测定时任务，则用于客户端向服务端发送续租请求

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/5-续租请求.png)

# 三 注册 注册实例信息

总的来说，客户端启动时，会进行服务的注册。在运行过程中，会定期检查客户端的实例信息与服务端的实例信息是否一致，若不一致，则需要进行服务更新

## 01. 注册实例信息（Eureka-Client）

**1）创建实例信息变更监听器**

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/6-监听器.png)

**2）注册状态变更监听器，并通过调用 InstanceInfoReplicator.start()开启实例复制调度器**

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/7-实例复制调度器.png)

**3）InstanceInfoReplicator.start()处理流程**

进入 InstanceInfoReplicator.start() 方法，我们可以看到该方法调用了 instanceINfo.setIsDirty() 修改实例信息的状态为 true，表示当前的实例信息与实际值不一致，需要更新（目的是实例 InstanceInfo 在刚被创建时，它的 eureka-server 信息为空，因此需要加载一次实例信息）

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/8-实力信息.png)

那么，实例信息第一次开始就会进行刷新实例信息操作。具体如何实现注册与检查更新操作呢？我们直接看下一步

## 02. 刷新应用实例信息（Eureka-Client）

**1）InstanceInfoReplicator.run()处理流程**

客户端实例启动时、在服务运行过程中出现实例信息有更新时，均会进行服务状态信息的注册

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/9-注册.png)

注册完成以后，更新实例为已同步状态，并创建下一个实例复制的定时任务，进行相同的实例变更监听更新操作

一直这样，循环进行下去，实现实例信息的周期性检查更新动作

那么，实例信息的检查更新具体逻辑是怎样的呢？

**2）实例信息检查更新**

进入 DiscoveryClient.refreshInstanceInfo() 方法，进行实例信息的检查与更新

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/10-检查与更新.png)

包含以下 3 步：

**1）更新 hostname**

调用 ApplicationInfoManager.refreshDataCenterInfoIfRequired()
进行数据中心相关信息的更新

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/11-的更新.png)

将当前实例信息的地址与注册中心中的地址进行比较。若不一致，说明 hostname 有变化，则需要调用 updataInstanceInfo(...)方法进行更新

**2）更新租约信息**

调用 ApplicationInfoManager.refreshDataCenterInfoIfRequired()进行租约信息检查更新

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/12-更新.png)

分别判断续租时间的续租时间、续租间隔时间，与配置中的租约信息是否一致，若某个值不一致，说明信息有过变化，那么需要更新这两个值为当前值；更新以后。将实例变成待同步状态，在下次续约时，进行更新同步

**3）获取并更新实例状态**

最后，通过调用健康检查处理类方法，获取并更新实例状态即可

## 03. 发起注册请求（Eureka-Client）

调用 DiscoveryClient.register() 方法，Eureka-Client 向 Eureka-Server 注册应用实例

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/13-应用实例.png)

## 04. 接收注册请求（Eureka-Server）

Eureka-Server 通过映射 ApplicationResource.addInstance()方法来接收应用实例信息的请求

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/14-的请求.png)

## 05. 注册应用信息（Eureka-Server）

接收到实例信息的注册请求，调用 AbstractInstanceRegistry.register(...)方法，进行具体注册流程处理
![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/15-流程处理.png)

具体实现流程，可参考上一篇文章-Spring Boot（九）：注册中心 Eureka-服务端视角

# 四 续约 renew 篇

在初始化信息步骤中，我们看到了客户端会创建心跳检测的定时任务。那么，它就会在固定周期内进行续约

## 01. 发起续约请求（Eureka-Client）

**1）心跳线程**

使用心跳线程 DiscoveryClient.HeartbeatThread，执行 客户端向 Eureka-Server 发起续租( renew )请求的动作

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/16-的动作.png)

该线程会通过调用 renew()方法，判断是否续约成功。若续约成功，则直接更新最后一次成功心跳时间戳为当前系统时间

**2）renew()方法**

进入 renew()方法

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/17-方法.png)

调用 sendHeartBeat(),向 Eureka-server 发起续约请求。若请求结果为 404（Eureka-server 在处理续约请求失败时，会返回 404），那么就需要调用 register()方法，重新发起注册

若注册成功，则更新实例为已同步状态

**3）进入 sendHeartBeat()方法**

进入 AbstractJerseyEurekaHttpClient.sendHeartBeat()方法

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/18-方法.png)

通过 put 请求，调用 Eureka-server 端的 url: apps/appName/id，发起续约请求

## 02. 接收续约请求（Eureka-Server）

Eureka-Server 通过映射 InstanceResource.renewLease()方法来接收应用实例信息的请求,用于处理客户端实例信息的续约请求操作

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/19-请求操作.png)

## 03. 处理续约请求（Eureka-Server）

依旧是通过 renewLease()进行续约请求的处理

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/20-的处理.png)

该方法调用 renew()方法，进行具体续约操作

若续约失败，则返回 404-NOT FOUND 给客户端，客户端就会重新发起注册

# 五 下线 shutdown 篇

## 01. 发起下线通知请求（Eureka-Client）

当客户端服务通过需要下线时，调用 DiscoveryClient().shutdown()方法，设置实例状态为下线，并通过 unregistr()方法，向 Eureka-server 发起下线请求

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/21-下线请求.png)

以下是 unregister()方法，它会向 Eureka-server 发起下线的 http 请求

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/21-http请求.png)

## 02. 接收下线请求（Eureka-Server）

Eureka-Server 通过映射 InstanceResource.cancelLease() 方法来接收处理客户端实例信息的下线请求操作

## 03. 处理下线请求（Eureka-Server）

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/22-server.png)

调用 cancel()方法，对该实例信息进行下线处理

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/23-下线处理.png)

该方法通过调用父级 cancel()方法，对该 Eureka-server 节点进行下线操作，若下线成功，则向其他节点同步下线操作，保持服务信息的一致性

# 六 总结 总而言之

今天通过一个客户端 Eureka-Client 注册 Eureka-server 的实例，并站在客户端的角度，聊了服务注册、续约以及下线三个主要操作的执行流程与逻辑

Spring Boot（九）：注册中心 Eureka-服务端视角以及本篇文章联合，我们分别站在服务端和客户端的视角，联通了 Eureka 的整个生命周期内的整体机制

总得来说呢，Eureka 可以分为三个角色：服务器端、服务提供者和服务调用者

Eureka 具有高可用性。服务器端 Eureka-server 是以平级关系存在（并没有 leader 与 follower 机制，大家都是平起平坐的关系），每次服务实例信息有变更，它会通过监听将其变化情况注册在其中一个节点上，并都会复制给其他节点。

如有某个节点挂了，其他节点依旧能够进行服务的协调工作，只是信息不一定是最新的（不保证信息的一致性）

需要注意的是，我们平常说的 Eureka 只能保证可用性，不保证一致性，都是针对服务出现故障的情况说的~

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img-sp10/24.png)

如图，服务端 Eureka-server 会存储服务实例信息，通过复制实现服务实例信息在各个节点同步，并定期去检查服务实例信息状态；

各个客户端也会通过健康检查等机制进行自我状态检查，要是信息有了变化，均会向服务 Eureka-server 发起请求；

Eureka-serve r 则会分别处理这些状态变化，来保持实例信息的更新。

其中需要注意的是，当 Eureka Server 节点在短时间内丢失过多客户端（可能发生了网络分区故障），默认是 15 分钟内收到的续约低于原来的 85%时，这个节点就会进入自我保护模式

那么现在大家肯定了解了，为什么在服务启动时，会初始化那么多处理包含健康检查在内的 Bean，以及定时任务，对吧？这些处理 Bean 以及定时任务对于 Eureka 的整体机制起了很重要的作用，如果你还不太清楚，那么建议再回过头再好好看看两篇文章哈~若是跟着源码走一遍 就更能理解透彻了~

嗯，就这样。每天学习一点，时间会见证你的强大~

下期预告：

Spring Boot(十一)：注册中心 Zookeeper

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
