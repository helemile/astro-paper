---
title: Spring Boot（六）：那些好用的数据库连接池们
author: 杰哥
pubDatetime: 2023-07-31T16:08:06.130Z
postSlug: spring-boot-6
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - 连接池
  - Mysql
  - HikariCP
  - Druid
description: 数据库连接池，在我们与数据库操作工程中所起的作用可谓是巨大的，尤其是需要频繁操作数据库的情况下
---

数据库连接池，在我们与数据库操作工程中所起的作用可谓是巨大的，尤其是需要频繁操作数据库的情况下

简单来说，它作为一个连接池，来管理数据库的连接，避免了数据库每次执行 **sql** 语句，都要去重新创建连接导致的性能灾难

目前最热门的数据库连接池，就要属阿里巴巴的 **Druid** 以及 **HikariCP** 了，它们也分别是 **Spring Boot 1.x** 和 **Spring Boot 2.x** 默认的数据库连接池。因此，今天我们主要来聊聊这两种好用的数据库连接池

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/dog.jpg)

# 一 理论: HikariCP

官网地址：**https://github.com/brettwooldridge/HikariCP**

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/光.png)

图一

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/光2.jpg)

图二

这两个图分别为官网中的内容，图一主要表明了 **HicariCP** 连接池的特点，快如光速

图二，则表示与其他各个连接池的比较。连接的创建与关闭和 statement 的创建和关闭时的效率比较。用测试结果再次说明了它的闪光点：快速

## 01.特点

显而易见：快速，简单，可靠

## 02.为什么快

1）字节码级改变（很多方法通过 **JavaAssist** 生成）

**HikariCP** 利用了一个第三方的 **Java** 字节码修改类库 **Javassist** 来生成委托实现动态代理

动态代理的实现在 **ProxyFactory** 类，源码如下

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/3-源码如下.jpg)

我们发现这些代理方法中只有一行抛异常的处理代码，注释写着“**Body is replaced (injected) by JavassistProxyFactory**”，其实方法 **body** 中的代码是在编译时调用 **JavassistProxyFactory** 才生成的，编译后的 **JavassistProxyFactory.class** 类代码如下图

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/4-如下图.jpg)

之所以使用 **Javassist** 生成动态代理，是因为其速度更快，相比于 **JDK Proxy** 生成的字节码更少，精简了很多不必要的字节码

### 2）大量小改进

**a 用 FastList 替代 ArrayList**

我们知道，**JDBC** 连接数据库的 6 大步骤中，到了最后一步需要关闭资源时，关闭 **Statement** 时，必须将其从此集合中删除；关闭 **Connection** 时，必须迭代该集合并关闭所有打开的 **Statement** 实例，最后必须清除该集合

**ArrayList** 本来用于在 **ProxyConnection** 类中跟踪、**管理打开的 Statement** 实例。**HikariCP** 则使用定制化修改的 **FastList**<**Statement**> 替换 **ArrayList**<**Statement**>实例

而为何这样就会加快效率呢？

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/5-效率呢？.jpg)

**ArrayList** 执行 **get（int index）**方法时，对每个对象都要检查。而使用 **FastList** 则不需要

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/6-则不需要.png)

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/6-则不需要2.png)

可以看到，使用 **FastList** 时，由于直接定义的就是一个固定长度的数组，那么范围就是可以保证的，因此在 **get** 方法中，直接使用元素索引找到特定元素，就不需要对每个对象都进行检查了

此外，**ArrayList** 的 **remove**（**Object**）是从头到尾进行扫描，但是 **JDBC** 编程中的常见模式是在使用后立即关闭 **Statement**，或者以打开的相反顺序关闭 **Statement**。对于这些情况，其实从尾部开始的扫描会更好

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/7-会更好.png)

我们可以看到，**FastList** 改造了 **remove**(**Object**) 方法，采用从后往前的顺序进行对象的删除

因此，将 **ArrayList** <**Statement**> 替换为自定义类 **FastList**，该类消除了范围检查并执行了从尾到头的删除扫描，大大提升了执行效率

**b 无锁集合** **ConcurrentBag**

**HikariCP** 包含一个名为 **ConcurrentBag** 的自定义无锁集合

这是一个专门的并发包，可以为连接池实现 **LinkedBlockingQueue** 和 **LinkedTransferQueue** 的卓越性能。它尽可能使用 **ThreadLocal** 存储来避免锁定，但如果 **ThreadLocal** 列表中没有可用的项目，则会使用它来扫描公共集合

当借用线程没有自己的东西时，**ThreadLocal** 列表中的未使用项可能被“窃取”。它是一个“无锁”实现，使用专门的 **AbstractQueuedLongSynchronizer** 来管理跨线程信令

**c 代理类的优化**

比如用 **invokestatic** 代替 **invokevirtual**。此更改从堆栈中删除了静态字段访问，推入和弹出操作，并因为保证了呼叫站点不会更改，使得 **JIT** 可以更轻松地优化调用

## 03.Spring Boot 对它的支持

它是 **Spring Boot 2.x** 默认数据连接池。因此，要想使用它来操作数据库，可以无需配置数据库连接池，**Spring Boot** 默认就进行了配置

那么 **Spring Boot** 是如何实现自动使用 **HikariCP** 数据库连接池的呢？

1）首先，**Spring Boot** 的 **starter-jdbc** 依赖中，就自动引入了 **HikariCP** 连接池的依赖

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/8-的以来.png)

2）查看数据库配置类 **DataSourceConfiguration** 对 **HicariCP** 的配置

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/9-的配置.jpg)

当定义了 **HikariDataSource** 类（存在于第一步中自动引入的 HicariCP 依赖中），未定义 **DataSource** 类，并且有 **spring.datasource.type** 属性为 **com.zaxxer.hikari.HikariDataSource**，当然 **matchIfMissing = true** 表示即使不设置，也符合条件

使用前缀为 **spring.datasource.hikari**，对 **DataSourceProperties** 中的各个属性进行默认配置，然后创建配置数据源（如果 **application.properties** 中先需要覆盖这些属性的值，则需要配置的名称表示为前缀名+**DataSourceProperties** 中的属性名称），如：**spring.datasource.hikari.url**

就这样，两个步骤的配合，就实现了 **Spring Boot 2.x** 对 **HicariCP** 的默认支持

## 04.监控功能

**HikariCP** 提供了一些监控指标，他的监控指标都是基于 **MicroMeter** 提供出来的，然后支持 **Prometheus** 和 **Dropwizard**。具体如何实现，本次暂不做讲解。如有疑问，可随时沟通

# 二 理论: Druid

**Druid** 连接池是阿里巴巴开源的数据库连接池项目。**Druid** 连接池为监控而生，内置强大的监控功能，监控特性不影响性能。功能强大，能防 **SQL** 注入，内置 **Loging** 能诊断 Hack 应用行为

## 01. 稳定特性

扛得住双十一，稳定性你想呢？

## 02. 为监控而生

**Druid** 增加 **StatFilter** 之后，能采集大量统计信息，同时对性能基本没有影响。**StatFilter** 对 **CPU** 和**内存**的消耗都极小，对系统的影响可以忽略不计

监控不影响性能是 **Druid** 连接池的重要特性

**1）执行次数、返回行数、更新行数和并发监控**

**StatFilter** 能采集到每个 **SQL** 的执行次数、返回行数总和、更新行数总和、执行中次数和和最大并发。并发监控的统计是在 **SQL** 执行开始对计数器加一，结束后对计数器减一实现的。可以采集到每个 **SQL** 的当前并发和采集期间的最大并发

**2）慢查监控**

通过如下所示配置慢 **sql** 打印，可以打印出慢 **sql** 的执行时间，具体 **sql** 语句的打印输出

```
#慢sql打印配置
spring.datasource.druid.filter.stat.log-slow-sql=true
spring.datasource.druid.filter.stat.slow-sql-millis=100
```

## 03. 众多扩展点

众多扩展点，方便进行定制

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/10-进行定制.png)

各个扩展点分别通过继承类 **FilterEventAdapter**，分别进行不同功能的扩展。通过源码可以看到，包括建立连接之前，**statement** 执行 sql 语句之前、之后可以做什么等

其中, **config** 负责进行加解密相关工作，**stat** 负责监控，**slf4j** 则负责与 **log4j** 配合进行日志输出

如果我们要实现功能扩展的话，也只需要同样继承 **FilterEventAdapter**，实战环节会有扩展示例哦~

## 04. 防 sql 注入

通过一些配置项，可以轻易实现防 sql 注入功能

如以下配置实现 **mysql** 数据库中禁止删表、删除数据操作

```
#sql防注入配置
spring.datasource.druid.filter.wall.enabled=true
spring.datasource.druid.filter.wall.db-type=mysql



#设置数据库不可以执行删除、删表操作

spring.datasource.druid.filter.wall.config.delete-allow=false
spring.datasource.druid.filter.wall.config.drop-table-allow=false
```

## 05. ExceptionSorter

**ExceptionSorter**-对各种主流数据库的返回码都有定制

下面是 **ExceptionSorter** 接口，各个数据库实现类通过实现这个接口，进行方法的定制化覆盖

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/11-定制化覆盖.jpg)

Spring Boot 会根据我们引入的数据库驱动依赖，来对应不同数据库的返回码处理

## 06. 内置加密配置

# 三 实战：实战环节

相信大家对两种数据源已经有了一定的了解了，接下来就分别看看在 **Spring Boot（2.x）**中分别如何使用这它们吧

## 01. Spring Boot 使用 HicariCP

由于在 **Spring Boot 2.x** 中，只要需要操作数据库，就需要引入 **JDBC** 依赖（ **Mybatis** 和 **JPA** 对应的依赖中都包含了 **JDBC** 依赖），而我们在理论环节都已经发现了，**JDBC** 中已经引入了 **HicariCP** 依赖

即 **Spring Boot 2.x** 已经默认支持了 **HicariCP**，我们需要做的就只需要关心数据库操作相关功能即可，完成这些，直接运行项目即可

注意：关于 **Spring Boot** 连接数据库的具体操作，可分别参考前几次文章。由于使用这两种连接池的区别主要是依赖和配置文件中配置项名称的区别，因此本次实战只做依赖、配置演示

### 1）引入依赖

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/12-引入以来.png)

### 2) 属性配置

```
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/test?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=123456
#可以不用配置数据库驱动，SpringBoot 会根据引入的依赖进行自动配置
#spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

3）项目启动

日志打印如下：

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/13-打印如下.png)

如图所示，项目默认使用的是 HicariCP 连接池

## 02. Spring Boot 使用 Druid

1）引入依赖

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/14-引入以来.png)

引入 **druid** 数据源，并排除掉 jdbc 中的 **HicariCP** 依赖。

**Tips:**

虽然经测试发现，不排除 **HicariCP** 依赖，**Spring Boot** 也会直接使用主动引入的 **Druid**，但是为了项目的简洁，以及可管理性，尽量排除掉比较好

2）属性配置

```
spring.datasource.druid.initial-size=5
spring.datasource.druid.max-active=5
spring.datasource.druid.min-idle=5
spring.datasource.druid.filter.config.enabled=true
#其中conn为自己扩展的
spring.datasource.druid.filters=conn,config,stat,slf4j
#spring.datasource.druid.connection-properties=config.decrypt=true;config.decrypt.key=${public-key}
spring.datasource.druid.test-on-borrow=true
spring.datasource.druid.test-on-return=true
spring.datasource.druid.test-while-idle=true
#慢sql打印配置
spring.datasource.druid.filter.stat.log-slow-sql=true
spring.datasource.druid.filter.stat.slow-sql-millis=1
#public-key=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALS8ng1XvgHrdOgm4pxrnUdt3sXtu/E8My9KzX8sXlz+mXRZQCop7NVQLne25pXHtZoDYuMh3bzoGj6v5HvvAQ8CAwEAAQ==
#sql防注入配置
spring.datasource.druid.filter.wall.enabled=true
spring.datasource.druid.filter.wall.db-type=mysql
spring.datasource.druid.filter.wall.config.delete-allow=false
spring.datasource.druid.filter.wall.config.drop-table-allow=false
```

### 3）功能扩展

**a 自定义扩展类 ConnectionLogFilter**

继承 **FilterEventAdapter**，并分别实现建立连接前、后的操作

```
@Slf4j
public class ConnectionLogFilter extends FilterEventAdapter {

    @Override
    public void connection_connectBefore(FilterChain chain,
                                         Properties info) {
        log.info("BEFORE CONNECTION!");
    }

    @Override
    public void connection_connectAfter(ConnectionProxy connection) {
        log.info("AFTER CONNECTION!");
    }
}
```

**b 创建 druid-filter.properties 文件，配置该扩展类**

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/15-扩展类.png)

**c 配置类中进行配置**

```
spring.datasource.druid.filters=conn,config,stat,slf4j
```

即在 **spring.datasource.druid.filters** 项中，添加 **conn**。

### 4）项目启动

如下所示，项目已经切换到了 **druid** 连接池。并实现了扩展类

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp6/16-实现了扩展类.jpg)

# 四 总结：总而言之

今天主要聊了一下 **HicariCP** 和 **Druid** 这两个常用的数据库连接池。其实两个作为连接池还是挺不错的，**HikariCP** 的功能比较纯粹一些，对性能有极大地积极作用，**Druid** 内置的功能会更丰富，可扩展的点比较多。

建议如果只是纯粹拿来做连接池，**HikariCP** 就挺好的。如果有自己定制各种功能的需求，比如密码加密、监控功能等，**Druid** 会有更好扩展。

本人实际开发中使用的 **Druid** 连接池会更多一些，你呢？

总结：

**1、HicariCP 数据库连接池的特点-为什么快**

**2、Druid 数据库连接池的特点-为监控而生**

**3、Spring Boot 对两种连接池的支持**

嗯，就这样。每天学习一点，时间会见证你的强大。

下期预告：

**Spring Boot（七）：你不能不知道的 Mybatis 缓存机制！**
