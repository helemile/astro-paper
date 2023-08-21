---
title: SpringBoot（二）：第一个Spring Boot项目
author: shumile
pubDatetime: 2023-07-25T20:11:06.130Z
postSlug: spring-boot-2
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - 第一个Spring Boot项目
description: Spring Boot 系列文章第二弹开始啦~上一篇文章中我们概述了 Spring Boot 特性、优缺点等，相信你对它有了一定印象。今天，让我们一起动手开始第一个 SpringBoot 项目吧
---

`Spring Boot` 系列文章第二弹开始啦~
上一篇文章中我们概述了`Spring Boot`特性、优缺点等，相信你对它有了一定印象。今天，让我们一起动手开始第一个 SpringBoot 项目吧

## 环境准备

- 编译器 `IDEA`
- `JDK`版本：1.8
- 构建工具：`Maven`

## 一 新建项目

建议大家使用`IDEA`创建项目，操作方便简单快捷。在日常的编码中，能够起到事半功倍的效果。

- 第 1 步：打开`IDEA`,点击`Create New Project`，进入第二步

- 第 2 步：左侧选择`Sping Initializer`，右侧分别配置 JDK 以及初始化器的地址，使用默认地址`https://start.spring.io/`，点击`Next`

- 第 3 步：核对系统自动生成的项目信息，点击 Next

- 第 4 步：选择依赖。因为第二步中我们选择了项目初始化的服务地址https://start.spring.io/，``IDEA``就会为我们加载这个地址中的所有依赖

我们只选择`Web`和经常会用到的`lombook`这两个依赖即可,点击`Next`

- 第 5 步：点击 finish,就完成了项目的创建

> Tips:大家会常常采用另一种方式创建项目。在https://start.spring.io/页面直接进行`Spring Boot`的项目名称和依赖配置，配置好之后，再下载到本地，导入`IDEA`

其实两种方式如出一辙，个人更习惯采用直接在`IDEA`新建的这种方式，`IDEA`直接帮我们加载了网页中的所有依赖，还省了解压，再打开项目的步骤。更为方便快捷

## 二 项目目录

创建好的项目目录如下所示

即

```java
demo
 +-src
  +-main
    +-java
      +- com.shumile.demo
        +- DemoApplication.java
  +-resources
     +-static
     +-templates
     +-application.properties
 +-test
 +-pom.xml
```

主要目录：

- src/main/java：程序开发以及主程序入口
- src/main/resources：配置文件
- src/test/java：测试程序

主要文件：
1 `DemoApplication.java` -项目唯一启动类（带有@SpringBootApplication 注解）
2 `static` 存放`web`访问的静态资

3 `templates` 存放模板文件
4 `application.propertie`s 项目配置文件
5 `pom.xml` 配置 maven 依赖的文件
查看`pom`文件，我们可以看到我们刚刚选择的两个依赖，`IDEA`结合`Spring Initializer`都帮我们配置在了`pom.xml`中

并自动配置了`test`依赖，可以用于快速单元测试。

项目打开以后，首先需要分别配置`maven`和`JDK`。配置成功以后，点击弹出的`import Changes`。即可下载依赖

等待依赖下载完成以后，就可以进入启动项目环节啦~

## 三 启动项目

右键`DemoApplication`启动类，直接启动项目

查看日志：显示

```java
Started DemoApplication in2.444 seconds (JVM running for3.307)
```

表示启动成功

看吧，我们就这么轻易的完成了项目的成功创建和启动~

想要更直观一点的话，再来创建一个 web 接口，直接在页面访问我们的项目。

## 四 构建 web 服务

新建`Controller`类，如下所示：

```
import org.springframework.web.bind.annotation.GetMapping;

import org.springframework.web.bind.annotation.RequestMapping;

import org.springframework.web.bind.annotation.RestController;

@RestController

@RequestMapping("/")

public class HelloController {

@GetMapping("/hello")

public String helloWorld(){

return "hello,world~";

}}
```

重新启动服务，在浏览器输入地址：`http://localhost:8080/hello`,得到结果

当当当当，第一个`Spring Boot`项目就轻轻松松建好啦~是不是比起 Spring 要搭建一个框架来说，简单了不止十倍呢？

为什么这么简单呢？想了解的话，继续往下看。

## 五 `Spring Boot` 帮我们做了什么

这主要还是归功于`Spring Boot` 的“约定大于配置”以及起步依赖（`starter-dependency`）这两大无敌特性。

### 1 自动配置

比如，服务端口，`Spring Boot` 默认配置成了 8080；接口请求前缀配置项`server.servlet.context-path`，默认配置成了“/”。如果我们将这个配置项，配置为`/shumile`

```
server.servlet.context-path=/shumile
```

这个时候，请求页面，就会返回 404

加上前缀`/shumile`，再次访问，就成功了~说明这一配置项生效了。

### 2 起步依赖

`Spring Boot` (一)：特性概览中有提到，项目中引入了 spring-boot-starter-web 依赖，即引入了`tomcat`和`webMvc`等相关依赖，那么我们的服务器就直接会使用`tomcat`服务器启动，并支持 webMvc 功能。

关于这两点，`Spring Boot` (一)：特性概览中

有提到具体的原理，在以后的文章里，我也会继续提到~ 大家会在以后的更多功能实现中，体会的越来越明显。

## 六 总结

今天主要带大家完成了第一个`Spring Boot`项目的搭建，如果什么问题的话，可以随时留言提问，我会耐心解答哦~想了解`Spring Boot`或者`java`的其他内容，也都可以留言与我互动哦~

以后，我会每周发两篇关于`Spring Boot` 的系列文章，内容会涉及`Spring Boot`的常用的大部分功能。

如果你一直对软件感兴趣，平时很少有机会学习；如果你对 java 感兴趣、如果你有过 Spring 或者`Spring Boot`相关开发经验，想要再系统地回顾一遍的话，欢迎关注我，一起轻松学习`Spring Boot` 吧~
