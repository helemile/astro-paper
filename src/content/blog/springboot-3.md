---
title: Spring Boot（三）： 操作数据库-Spring JDBC
author: shumile
pubDatetime: 2023-07-26T20:11:06.130Z
postSlug: spring-boot-3
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - JDBC
description: Spring Boot 系列文章第二弹开始啦~上一篇文章中我们概述了 Spring Boot 特性、优缺点等，相信你对它有了一定印象。今天，让我们一起动手开始第一个 SpringBoot 项目吧
---

## 简介

`Spring Boot`访问数据库，常用的方式有 Mybaits、Hibernate 以及`Spring Boot`提供的 JDBC 这三种方式。其中，`Spring JDBC`，是`Spring`中最基本、最底层的访问数据库的实现方式。

我将会分三次内容对每种操作数据库的方式进行分别说明，感兴趣的话，继续关注后续文章更新哦~

![](https://p.ipic.vip/anp0jk.jpg)

今天，我们先一起来看`Spring JDBC`是如何操作数据库的。希望大家通过本篇文章的阅读，可以了解到

- 超级好用的`lombok`

- `Spring Boot`的常用 bean 注解

- 如何使用`Spring JDBC`操作`mysql`数据库

## 实战原理

### 1 超级好用的开发辅助工具-`lombok`

项目演示中会使用到一个超级好用的开发辅助工具`lombok`。使用它，不仅会节省我们的时间，还会大大减少代码量。

使用它开发，只需要在 IDEA 中添加`lombok`插件，并引入`lombok`依赖即可（本文不做具体介绍，网上资料很多，大家随便搜搜就出来啦）。
常用注解

- @Getter 和@Setter ：使用在属性上，生成的 getter 和 setter 方法。

- @ToString ：使用在类上，为对应类实现 toString 方法。

- @EqualsAndHashCode ：使用在类上，生成 hashCode 和 equals 方法。

- @NoArgsConstructor ：使用在类上，生成无参的构造方法。

- @AllArgsConstructor ：使用在类上，生成包含类中所有字段的构造方法。

- @Data ：使用在类上，生成 setter/getter、equals、canEqual、hashCode、toString 方法，如为 final 属性，则不会为该属性生成 setter 方法。

- @Slf4j ：使用在类上，生成 log 常量。演示中会用到。

### 2 Spring 常用的 bean 注解

- @Component ---一个通用的注解，可以定义一个通用的 Bean
- @Repository ---数据操作仓库，即用于定义数据库操作 dao 层
- @Service - -业务服务，用于服务层
- @Controller --- 用于 controller 层
- @RestController---`Spring Boot`针对 Rest 服务定制的@Controller 注解

### 3 使用的依赖

```java
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### 4 简单的 JDBC 操作方法

`JDBCTemplate`为我们提供了增删改查数据库的方法。

```
query
queryForObject
queryForList
queryForMap
update
execute
```

查询，除了基本查询，还提供了返回值分别为 Object、list、map 等查询方法，update 方法可以分别对数据进行增删改操作，execute 则为基本的数据库执行方法。

### 5 方法原理

`Spring JDBC`,即 Spring 对 JDBC 的整合，使我们的使用更为便捷而已。各个操作方法的背后，其实也是执行了 JDBC 的那 6 个步骤：
a 加载数据库驱动

b 建立链接

c 创建 statement

d 执行 SQL 语句

e 处理结果集

f 关闭数据库

`Spring Boot`会根据引入的依赖，加载数据库驱动。而 JdbcTemplate 操作方法，则包含了后 5 个步骤

![](https://p.ipic.vip/1qfvw4.jpg)

其中第 4、5 步骤，是包含在调用的具体操作类中的。而代码中是使用 action.doInStatement(stmt)去调用具体的操作类的。如 query 方法

![](https://p.ipic.vip/srv5bm.jpg)

## 实战环节

好了，做了这么多准备工作，接下来，让我们正式进入实战环节吧~

由于工作、学习中`mysql`数据库使用比较普遍，因此特地在此为大家演示`Spring JDBC`如何实现操作`mysql`数据库。其他数据库的方式也很类似。只需要更换对应的数据库连接包以及地址配置即可。

那么有的同学估计要问了，它都支持哪些数据库呢？

答案就是，支持 JDBC 的所有数据库喽~也就是说它支持几乎所有关系型数据库的操作。

### 1 引入依赖

需要分别引入以下三个依赖：
spring-jdbc 依赖：spring-boot-starter-jdbc
`mysql`连接类：`mysql-connector-java`
`lombok`开发辅助类:`lombok`

![](https://p.ipic.vip/j2whby.jpg)

### 2 数据库信息配置

在配置文件 application.properties 中分别配置数据库的 url、username、password。

![](https://p.ipic.vip/237o1b.jpg)

> Tips1:
> `Spring Boot` 会根据我们引入的数据库连接依赖类型，自动配置数据库的驱动，因此我们可以不需要配置数据库的驱动项。

同时还要保证数据库中已经存在了我们后续要用到的 User 表哦~我创建的表是这样的

![](https://p.ipic.vip/y272x4.jpg)

### 3 新建实体类

![](https://p.ipic.vip/2m0xcm.jpg)

我们可以看到，实体类 User 中就是用了 `lombok`的两个注解 @Data 和@Builder,有了这两个注解，我们的实体类，就变得清爽很多了~

### 4 新建 dao 层

1)定义类。使用@Repository 注解，声明该类为 dao 层的 bean,通过`lombok`的@Slf4j 注解，进行快捷的日志输出 2)注入 JdbcTemplate

3)具体操作。使用 JdbcTemplate 中的操作类，分别新增、查询操作

![](https://p.ipic.vip/xeaykx.jpg)

查询，分别演示了返回结果为对象和返回为 List 的两种情况，大家感兴趣可以自己再尝试一下其他情况。

![](https://p.ipic.vip/76fkj6.jpg)

![](https://p.ipic.vip/vlfgfg.jpg)

### 5 调用 dao 层方法

`Spring Boot`的启动类，实现了 ApplicationRunner 接口，并覆盖其 run 方法。服务启动时，就会运行 run 方法。在 run 方法中分别调用添加用户、查询用户列表的方法。

![](https://p.ipic.vip/f3a7ob.jpg)

> Tips2：与 ApplicationRunner 类似的接口还有 CommandLineRunner，大家有兴趣可以先自己了解一下，以后有机会可以给大家专门讲解。在这个地方，大家只需要了解 ApplicationRunner 接口的用法即可。

## 6 启动项目

![](https://p.ipic.vip/3x636l.jpg)

我们可以看到，日志中显示，添加了一条数据，并查询到一条记录好了，相信大家对简单的`Spring JDBC`操作数据库方法已经掌握了~

![](https://p.ipic.vip/c1s0ie.jpg)

如果想了解更多的操作方法，包括批量操作，查询结果类型为 Map 的，以及使用 NamedSql 的方式，大家可以查看操作类`JDBCTemplate`，结合官网进行进一步的学习。

当然，有任何问题，也欢迎大家留言与我沟通哦~

## 总 结

1 强大的开发辅助工具`lombok`可以简化我们的开发
2 `Spring Boot` 常用声明 bean 的注解
3 `Spring JDBC`操作数据方式

今天演示了`Spring JDBC`操作数据库的原理及方式，并在同时也提到了`Spring Boot`常用声明 bean 的注解、`lombok`工具的使用、JDBC 操作数据库的原理等内容。

以后，我会继续以这种原理+实战的方式，来讲述`Spring Boot`的功能点。这样学习效率可能更高。建议大家实战结束以后，再返回看一遍原理，效果会更佳哦

如果你感觉到自己学到了很多，或者复习到了很多技能点，就快来关注我，一起继续学下去吧~
