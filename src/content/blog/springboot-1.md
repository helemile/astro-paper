---
title: SpringBoot（一）：特性概览
author: shumile
pubDatetime: 2023-07-24T20:11:06.130Z
postSlug: spring-boot-1
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
description: 我们知道，IT 界使用 Java 做 Web 应用开发已有 20 年左右的历史，现如今已经成为一个成熟的语言。而最受 Java 开发者喜爱的框架当属 Spring，Spring 也随之成为了在 Java EE 开发中真正意义上的标准。
---

近两年，`Spring Boot`成为了 java web 开发主框架，日益风行。那么，究竟为什么要用`Spring Boot`呢？

## 1. SpringBoot 诞生

![](https://p.ipic.vip/0i140e.jpg)

我们知道，`IT` 界使用 `Java` 做 `Web` 应用开发已有`20`年左右的历史，现如今已经成为一个成熟的语言。而最受`Java`开发者喜爱的框架当属`Spring`，`Spring`也随之成为了在`Java EE`开发中真正意义上的标准。

但是随着新技术的发展，脚本语言大行其道的时代（Node JS，Ruby，Groovy，Scala 等），`Java EE`使用`Spring`逐渐变得笨重起来，大量的 XML 文件存在于项目中，所以项目本身繁琐的配置、整合第三方框架的配置问题，低下的开发效率和部署效率等等问题渐渐多了起来。在不断的社区反馈下，`Spring`团队便在`Spring`基础上，开发出了相应的框架：`Spring` Boot。

事实证明，`Spring Boot`确实是当之无愧的 javaEE 开发颠覆者～而`Spring`官方已经将 `Spring Boot` 作为公司最顶级的项目来推广，放到了官网上第一的位置，因此后续 `Spring Boot` 的持续发展也被看好。接下来让我们一起来聊聊 SpringBoot 的核心功能。

## 2.核心功能

`Spring Boot`的核心特点包含以下几项：

1）自动配置`-Auto Configuration`

2）起步依赖`-Starter Dependency`

3）命令行界面`-Spring Boot CLI`

4）运行监控`-Actuator`

其中最重要的，最常用的就是自动配置和起步依赖，先来说说自动配置

`Spring Boot`的自动配置，就是基于添加的`jar`依赖，自动依据类中的`Java`配置、配置文件`application.properties`的配置项以及环境上下文的配置进行自动配置。这些自动配置都是通过依赖`spring-boot-autoconfiguration`来实现的

乍一听，是不是感觉有点蒙？那就说得稍微细一点，跟把大象放进冰箱一样，它的加载流程分为以下 3 步：

### a. 加载配置文件

`Sring Boot`在启动时，会加载`spring.factories`里的属性`EnableAutoConfiguration`

属性中配置了`Sping Boot`支持的所有自动配置类(项目中通常会使用这种模式进行功能的扩展～)

![](https://p.ipic.vip/arioqd.jpg)

### b.声明相关类

进入类中，通过@Conditional 等一系列的注解来分别声明相应的类

### c. 加载配置项

通过对应的`propertis`文件里面的默认配置项进行加载相应的配置项。如果项目中通过`Java`或者配置文件的形式做了配置，那么就直接使用该配置的值，否则直接使用默认值

这也就是为什么 我们并没有配置 web 服务的端口，服务却会默认以 8080 端口启动；配置了以后，项目就会使用我们的端口号启动

如果还有一点点蒙，那就让我们一起进入源码，用例子来看看吧

### 举一个`Spring Boot`加载内嵌数据源的栗子

如果我们的项目里面引入了 redis 客户端的依赖（jedis 和 lettuce 任何一种客户端），`Spring Boot`就会根据 redis 的的自动配置类 RedisAutoConfiguration 进行后续配置的加载

![](https://p.ipic.vip/3jfhco.jpg)

1. 首先会根据`@ConditionalOnClass()`判断是否存在`RedisOperations`这个类；

2）如果存在，就会启用`RedisProperties`里面对应`redis`的所有属性配置；

![](https://p.ipic.vip/jdqwnu.jpg)

这个类，采用前缀加属性,来组合我们的属性，其中具体的属性支持级联形式,比如:对于`spring`的端口配置,就是`spring.redis.host`了。

3. 接着通过代码

`@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})`

分别引入`LettuceConnectionConfiguration、JedisConnectionConfiguration`

这两个客户端的类(感兴趣的话，可以进入这两个类，看一下，它们其实也是使用`@conditional`注解继续进行判断的),每个客户端会自行建立连接,进行相应的配置

4. 最后，分别建立了两个`bean:redisTemplate和stringRedisTemplate`，我们看到它们均使用了`@ConditionalOnMissingBean`注解，即我们可以直接注入它们，进行使用；也可以根据业务场景，自己定义即可覆盖它们。

这样下来，只要我们引入了 reids 的客户端依赖，`Spring Boot`就帮我们自动配置了`redis`的地址、端口号等信息，还为我们声明了`redisTemplate`。

如果是连接本地`redis`,端口也为默认端口，那么我们要做的就只是直接注入`redisTemplate`，使用它提供的各个方法去操作`redis`啦！是不是很神奇！

### 现在跟着源码走一遍，就会更容易弄清楚啦

到此为止，我们就摸清了`Spring Boot`自动配置的神秘面目～

啊哈～有木有很简单～我们下次会继续带领大家动手实现自己的自动配置。从而更深刻的了解自动配置原理，敬请期待哦～

好了，搞清楚了自动配置，接下来让我们再一起看看起步依赖,起步依赖是`Spring Boot`的另一大核心功能。体现在以下两点

1）直接面向功能

2）一站获得所有依赖，不再复制粘贴

也就是说使用`Spring Boot` 的这一特性，我们清楚我们要做什么，只要引入一个相应的`starter-denpendency`,`Spring Boot`就会根据我们的需要，帮助我们配置相应的依赖，而不用我们一个一个地去引入多个依赖，避免 pom 文件变得冗长，还容易出现依赖冲突的问题

比如说我们要实现 web 功能,直接引入`spring-boot-starter-web`,其中包含了 json、tomcat、web、webmvc 以及 hibernate 的验证依赖。如果没有特别的业务需求，那么我们使用这一套依赖就完全足够了

![](https://p.ipic.vip/p2eefw.jpg)

注意：所有的[starter](https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/using-boot-build-systems.html#using-boot-start)均可通过官网查看

> 还可以通过每个 starter 后面的 pom，查看该依赖对应引入了哪些依赖

## 3.优点

1）异构性

各个微服务可以根据业务情况，采用不同的语言，不同的数据库进行存储。

2）弹性

采用了微服务，如果一个组件出现了不可用，不会导致级联扩展。

3）扩展

单体服务不易扩展，多个较小的服务可以按需扩展。比如说要更新电商平台的收费规则，可能就只会跟订单服务、用户服务等个别服务有关系，那么只需要扩展这些受到影响的服务即可。

## 4.实施微服务的代价

当然，任何东西都不可能是完美的，使用`Spring Boot`，也要付出一定的代价

1）分布式系统的的复杂性

采用了`Spring Boot`微服务架构，我们就自然不得不考虑分布式系统带来的服务管理问题。

2）开发、测试等诸多研发过程中的复杂性

如果一个服务，依赖了其他服务，那么启动这个服务，还需要保证它依赖的所有服务都同样启动。比如，在微服务体系中，总是会存在 A 服务依赖 B 服务，依赖 C 服务等情况。导致开发测试会变得复杂。

3）部署、监控等诸多运维复杂性

同样的，部署、监控，需要关注的服务也会随之变得多起来，显得很复杂。

## 5. 总结

总得来说，`Spring Boot`其实是对`Spring Framework`做了二次封装。以便简化开发，让程序员将更多的精力和时间放到业务上去。规避了繁琐的配置操作，从而减少了遭遇`bug`的数量。而对于运维人员来说，虽然要关心的服务可能更多了，但每种服务的维护均可以采用简单的脚本来进行优雅地维护

因此，总体来讲，`Spring Boot`的出现，极大地降低了开发和运维的复杂度，也是软件开发发展的必然趋势。
