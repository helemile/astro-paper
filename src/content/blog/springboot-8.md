---
title: Spring Boot（八）：Spring Boot的监控法宝：Actuator
author: 杰哥
pubDatetime: 2023-08-26T16:08:06.130Z
postSlug: spring-boot-8
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - 监控
  - Actuator

description: 在日常项目中，除了开发过程比较重要以外，实际上运维过程也尤为重要。而 Spring Boot 也为我们考虑到了这一点，它为我们提供了 Actuator 这一组件,帮助我们监控、管理应用程序
---

在日常项目中，除了开发过程比较重要以外，实际上运维过程也尤为重要。而 Spring Boot 也为我们考虑到了这一点，它为我们提供了 Actuator 这一组件,帮助我们监控、管理应用程序。

正如官网中所说的那样，它可以通过很小的动作产生巨大的变化

# 一 原理:认识 Actuator

## 01. 什么是 Actuator

**Spring Boot Actuator**，可在您将应用程序投入生产时帮助您监控和管理应用程序。分别支持 HTTP 和 JMX 两类端点。

## 02. 支持哪些监控项

总得来说，分为三类：

1）**应用配置类**：获取应用程序中加载的应用配置、环境变量、自动化配置报告等与 **Spring Boot** 应用密切相关的配置类信息

2）**度量指标类**：获取应用程序运行过程中用于监控的度量指标，比如：内存信息、线程池信息、HTTP 请求统计等

3）**操作控制类**：提供了对应用的关闭等操作类功能

常用端点如下：

```
1）auditevents 公开当前应用程序的审核事件信息

2) beans 该端点用来获取应用上下文中创建的所有Bean

3) caches 展示可用的缓存

4）conditions 显示在配置和自动配置类上评估的条件以及它们匹配或不匹配的原因

5）configprops 所有@ConfigurationProperties的配置列表

6）flyway 显示已应用的所有Flyway数据库迁移。

7）env 返回当前的环境变量

8）health 健康状况

9）httptrace 查看HTTP请求响应明细（默认100条)

10）info 展示服务应用信息

11）loggers 查看日志配置，支持动态修改日志级别

12）liquibase 显示已应用的所有Liquibase数据库迁移

13）metrics 显示当前应用程序的“指标”信息

14）mappings 显示所有@RequestMapping请求列表

15）scheduledtasks 显示服务中的所有定时任务

16）sessions 允许从Spring Session支持的会话存储中检索和删除用户会话

17）shutdown 可以使服务优雅的关闭，默认没有开启

18）threaddump 执行线程转储
```

              参考：Spring Boot官网之Actuator部分

注意：**Spring Boot Actuator** 也支持 **prometheus**、**jolokia** 和 **integrationgraph**，但若要使用它们，分别需要引入对应的依赖，才能正常使用

## 03. 如何实现

**1）引入依赖**

简单的使用呢，只需要引入以下依赖，启动服务即可

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

**2）重启服务，访问页面**

访问地址 **http://localhost:8888/actuator**，便可以看到各个监控 **url**,访问对应的 **url**，就可以看到对应的监控参数

需要注意的是，为了安全起见，**http** 默认只暴露 **health** 和 **info** 两个节点

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/1-两个节点.png)

## 04. 常用配置项

我们可以根据需要，通过配置项进行节点的开启与关闭设置

**1）开启单个 endPoint 端点**

```
• management.endpoint.<id>.enabled=true

• management.endpoints.enabled-by-default=false
```

示例：

```
#1、只开启info端点的配置
#关闭所有端点
management.endpoints.enabled-by-default=false
#开启info端点
management.endpoint.info.enabled=true
```

这两个配置项的结合，表示只开启了 info 端点

**2）暴露、关闭 endpoint 端点**

```
management.endpoints.jmx.exposure.exclude=

management.endpoints.jmx.exposure.include=*



management.endpoints.web.exposure.exclude=

management.endpoints.web.exposure.include=info, health
```

如上所示，前两个配置项表示对 jmx 端点的关闭与开启的整体配置，后两项配置则是对 http 端点的关闭与开启的整体配置。

**3）对于监控访问地址特定配置**

从第 3 小节中，我们可以看到，监控默认的 HTTP 访问地址为

**服务 ip:服务端口/actuator/<id>**

在实际生产中，我们也可以通过配置，将监控地址配置为与原服务不同的某个特定地址。

这样的话，我们就可以只给外部提供我们的服务地址，而不需要担心他们也监控我们的服务的作用了。

经常使用到的配置项如下所示：

```
management.server.address=监控服务ip

management.server.port=监控服务端口

management.endpoints.web.base-path=访问的baseUrl

management.endpoints.web.path-mapping.<id>=<id>访问路径
```

我们在项目中添加如下配置

```
#监控服务ip
management.server.address=127.0.0.1
#监控服务端口
management.server.port=8008
#访问的baseUrl（默认为/actuator）
management.endpoints.web.base-path=/myActuator
#对某个特定端点的路径配置
management.endpoints.web.path-mapping.health=/monitor/health
```

页面访问，原来的请求地址已无法访问，而我们配置的地址已经生效

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/2-已经生效.png)

## 05. 源码分析

总得来说，**Spring Boot Actuator** 是通过定制不同的 **endPoint**，进行各个监控项的监控的

具体点的话，我们跟着源码走一波~

如下所示，会通过 **EndpointDiscoverer.discoverEndpoints**()方法进行各个监控项的加载

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/3-的加载.png)

**1）加载 endPointBean**

通过调用 createEndpointBeans()方法，加载所有带有@EndPoint 注解的 bean

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/4.png)

我们可以看到，该方法时是从上下文中获取带有@EndPoint 注解的 bean，检查是否有重复 id 的 endPointBean，若未出现，则将结果返回

**2）加载 EndPointExtensionBean**

通过调用 addExtensionBeans()方法，加载所有带有@EndPointExtension 注解的 bean，同样检查是否出现重复 id 的 EndPointExtensionBean，若未出现，则将结果返回

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/5.png)

需要说明的是，该 EndpointExtension 类是对对应的 Endpoint 类的扩展。

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/6.png)

目前有三个这样的 Bean:

```
CachesEndpointWebExtension
HealthEndpointWebExtension
EnvironmentEndpointWebExtension
```

其中，**HealthEndpointWebExtension** 的定义如下

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/7.png)

我们可以看到，**HealthEndpointWebExtension** 通过标签 **endpint** 指向 **HealthEndPoint** 类

接着上一步，进入 **addExtensionBean**()方法,我们看到，该方法会分别对 **EndpointExtension** 和 **extensionBean** 进行判断

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/8.png)

**a 判断 endpointExtension**

进入扩展 bean 判断，只有该扩展类对应的 filter 与对应的 endpointBean 保持一致，并且对应的 endpointBean 并没有被关闭掉，才能将其加入到 endpointBean 中

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/9.png)

**b 判断 endpointbean**

是否匹配对应的 filter，此 bean 是否被 exclude 了！(配置文件配置)，同时判断 bean 上的注解是否匹配当前 supplier

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/10.png)

**3)处理获取到的 endPointBeans**

这个过程通过调用 convertToEndPoints()方法，分别加载各个 endPoint 类的操作方法，并将各个 endPoint 的 id 以/actuator 为前缀的形式映射为 URL（若有特定配置，前缀为配置项的值）

例如，默认情况下，health 端点映射为

/actuator/health

**4）特定端点特定处理**

当收到某个端点的请求时，再分别进行各个监控点的判断，并展示结果。

如，当请求 health 节点，
即调用 url：
http://localhost:8888/actuator/health 时，会按照以下步骤执行

**a 首先会调用 invoke()方法，获取到对应的 url 映射地址**

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/11.png)

**b 然后通过反射机制获取到对应节点的操作类与方法**

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/12.png)

通过上图，我们发现，获取到的需要检查健康状况的分别为磁盘空间、数据库连接状态以及 redis 连接状态

**c 分别执行各个操作方法，返回结果**

那么接下来，就会分别进入各个指示器，调用对应的 doHealthCheck() 方法进行相应检查处理

举个例子，以下是磁盘的健康检查类的 doHealthCheck() 方法，它主要获取磁盘可用空间等信息，并返回结果

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/13.png)

好了，以上就是我们对于 **Spring Boot Actuator** 的监控原理进行了跟踪梳理。其他的端点，也基本类似，大家不妨也跟踪源码，了解一下他们的具体处理机制

当然，对于一些监控项，也支持响应式，就是说，只要某个监控项的值发生了变化，就会实时更新数据。本节作为了解监控特性的第一篇，就先不介绍了，以后有机会可以专门再来一篇

# 二 实战环节

## 01. 快速实现

很简单，请参照第一节中 03 如何实现的操作步骤快速实现一个具有监控功能的应用程序，分为以下三步：

1）引入依赖
2）添加配置
3）运行服务，查看节点

## 02. 功能扩展

**Spring Boot Actuator** 为我们提供的诸多节点，已可以支持我们的大部分的监控功能了，但若是我们还想要扩展的话，我们可以分别通过添加自定义 **endpoint**、自定义健康状况检查项以及自定义健康状态进行监控项等进行扩展

**1）自定义 endpoint**

如果需要自定义节点，只需要定义一个带有注解 @Endpoint 的类，然后结分别合 @ReadOperation, @WriteOperation, or @DeleteOperation 定义具体的方法即可

如下所示

```
@Endpoint(id = "hello")
@Component
public class MyEndPoint {

        @ReadOperation
        public String getHello(){
            return "Hello~ World";
        }
        /**
         * 定义带有参数的
         * @param name
         * @return
         */
        @ReadOperation
        public String getHelloWithName(@Selector String name){
            return "Hello~"+name;
        }

        @WriteOperation
        public String postHello(){
            return "post Hello";
        }
        @DeleteOperation
        public String deleteHello(){
            return "delete Hello";
        }
    }
```

页面访问，你会发现，便多了我们新增的两个节点

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/14.png)

**2）自定义健康状况检查项**

**Spring Boot Actuator** 默认支持以下各项的健康检查

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/15.png)

它们均通过直接或间接实现 **HealthIndicator** 接口来处理各个服务项进行健康状况检查请求

那么，我们若要自定义自己健康状况检查项的话，同样只需要实现 **HealthIndicator** 接口，并重写对应的 health() 方法即可

比如我们现在需要增加一个监控库存中是否还有足够的苹果的时候，我们就可以定义如下所示的监控类

```
/**
 * 监控库存中是否还有足够的苹果
 */
@Component
public class MyIndicator implements HealthIndicator {

    private long count = 50; //实际中生产中，可以从数据库中获取苹果的数量
    @Override
    public Health health() {
        Health health;
        if (count<=5){
            health = Health.up()
                    .withDetail("count",count)
                    .withDetail("message","糟糕，苹果库存不足啦~需要备货啦~")
                    .build();
        }else {
            health = Health.down()
                    .withDetail("count",count)
                    .withDetail("message","我们还有足够的苹果哟~")
                    .build();
        }
        return health;
    }
}
```

重启服务，页面请求 health 的 url

![](https://cdn.jsdelivr.net/gh/helemile/notes@main/img_sp8/15.png)

健康检查项中就多了我们自定义的监控项

**3）自定义健康状况类型**

关于健康状态类型，actuator 内置以下几种 HTTP 状态码：

```
Status	Mapping
DOWN	503
OUT_OF_SERVICE	503
UP	 200
UNKNOWN	 200
```

我们也可以通过如下配置添加自定义状态：

```
#添加状态FATAL
management.health.status.order=FATAL, DOWN, OUT_OF_SERVICE, UNKNOWN, UP

#映射状态的状态码（即FATAL对应的状态码绑定为503）
management.health.status.http-mapping.FATAL=503
```

如果你需要更多的控制，也可以通过定义自己的 **HealthStatusHttpMapper bean** 进行实现

# 三 总结 总而言之

今天主要聊了聊作为 **Spring Boot** 的四大法宝之一的 监控特性

主要分别从认识 **Spring Boot Actuator**、如何使用使用它实现监控功能，进而跟踪源码探索底层原理，最后通过自定义节点或者监控项等方式进行功能扩展

**spring-boot-starter-actuator** 模块的实现对于实施微服务的中小团队来说，可以有效地减少监控系统在采集应用指标时的开发量。通常我们会使用它所提供的 **http** 的 **API** 进行监控端或者管理端的功能集成开发，从而实现服务各项指标的监控，尤其方便

当然，我们也可以通过它的各个端点的实现机制，定义我们自己的 **endPoint** 或者对于已有的 **endPoint** 功能进行进行相应的扩展，实现我们自身系统个性化的监控需求

因为监控项中暴露了很多服务参数，实际过程中，我们通常是借助 **Spring Boot Security** 来进行安全配置。留给大家进行探索咯~

看完文章，我想你肯定掌握了以下几点：

**1、Spring Boot Actuator 的监控项**

**2、如何使用 Spring Boot Actuator 进行应用程序的监控**

**3、Spring Boot Actuator 的监控原理**

**4、如何扩展监控项，实现定制化监控功能**

嗯，就这样。每天学习一点，时间会见证你的强大~

下期预告：

Spring Boot（九）：注册中心 Eureka-服务端视角

本期项目代码已上传到 github~有需要的可以参考
**https://github.com/wangjie0919/Spring-Boot-Notes**

往期精彩回顾

[`Spring Boot（七）`：你不能不知道的 Mybatis 缓存机制！](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483890&idx=1&sn=ceb10a1619ef4270e8a8d38b1e91af25&chksm=96eb03e3a19c8af5b5846f72b578ee9fdb207603d5658a79d721a97e456ee1f0b9ca1e946a61&scene=21#wechat_redirect)

[`Spring Boot`（六）：Spring Boot（六）：那些好用的数据库连接池们](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483845&idx=1&sn=b604205c54cbf132a0a3a0d9fd035bba&chksm=96eb03d4a19c8ac207645109ed636d89d781daa27bf5e1290393f04ef45aa9453073d4fa9c76&token=1019255373&lang=zh_CN#rd)

[`Spring Boot`（五）：春眠不觉晓，Mybatis 知多少](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483821&idx=1&sn=7c5ed87610b93589266106d1b07191d1&chksm=96eb03bca19c8aaa5967520fe03381481020450834e7b4e518dd729dc38b71c4cb50154d7e8d&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（四）：让人又爱又恨的 JPA](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483799&idx=1&sn=f0a553daf412aefeb2f7877970b04aca&chksm=96eb0386a19c8a90c15b1dae13d39546a2d01875d3f3489c2e2f709a97f8f8ca87bee4ec58e0&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（三）：操作数据库-Spring JDBC](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483779&idx=1&sn=6ce5bbd2d8028b176ecf3b1b7fa8f3ea&chksm=96eb0392a19c8a84f1e7356a36012a2ef2f17f4d70f60c83ea42c1e72ed5a561c18d1782577e&scene=21#wechat_redirect)

[`Spring Boot`（二）：第一个`Spring Boot`项目](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483751&idx=1&sn=90bb32f21dbfd6793b899ac4cb04d52f&chksm=96eb0376a19c8a60eb9ea0b8227b3e34d55efb0feb6e1aa944d90514333f2be70635b8bc820c&scene=21#wechat_redirect)

[`Spring Boot`（一）：特性概览](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483725&idx=1&sn=c1399fcb817d5a434f272b93abd36798&chksm=96eb035ca19c8a4ac715cf62f4cfe301e0124e7b800c87b3ae78503190bbb8ec5e2754ce3fd4&scene=21#wechat_redirect)
