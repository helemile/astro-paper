---
title: Spring Boot（五）：春眠不觉晓，Mybatis知多少
author: shumile
pubDatetime: 2023-07-29T20:08:06.130Z
postSlug: spring-boot-5
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - Mybatis
description: 在 JavaWeb 项目开发中，我们使用最多的 ORM 框架可能就是 Mybatis 了，那么对于常用的 Mybatis，你究竟了解多少呢？
---

## 目录

在 **JavaWeb** 项目开发中，我们使用最多的 **ORM** 框架可能就是 **Mybatis** 了，那么对于常用的 **Mybatis**，你究竟了解多少呢？

# 一 原理:战前磨刀

## 01.Mybatis 是什么

**Mybatis** 是**支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架**。**Mybatis** 避免了几乎所有的 JDBC 代码，手动设置参数以及获取结果集

**Mybatis** 可以对配置和原生 Map 使用简单的 **XML** 或注解，将接口和 **Java** 的 **POJOs** (Plain Old Java Objects,普通的 Java 对象)映射成数据库中的记录

## 02. Mybatis 如何操作数据库

支持三种操作方式

1）注解

2）语句构建器

3）xml 文件与接口映射方式

## 03. Mybatis 的缓存机制

缓存的重要性是不言而喻的。

考虑到性能，几乎所有与性能相关工具都会涉及到使用缓存， 我们可以避免频繁的与数据库进行交互， 尤其是在查询越多、缓存命中率越高的情况下， 使用缓存对性能的提高更明显。这个不在本篇细讲，到时候会专门作为一期内容呈现

## 04. Mybatis 的执行流程

![](https://files.mdnice.com/user/13208/c528dcd7-bbb2-4bff-aee7-b37abc9b539e.png)

1） **加载配置**

配置来源于两个地方，一处是**配置文件**，一处是 **Java** 代码的注解，将 SQL 的配置信息加载成为一个个 **MappedStatement** 对象（包括了传入参数映射配置、执行的 **SQL** 语句、结果映射配置），存储在内存中

2） **SQL 解析**

当 API 接口层接收到调用请求时，会接收到传入 SQL 的 ID 和传入对象（可以是 Map、JavaBean 或者基本数据类型），**Mybatis** 会根据 SQL 的 ID 找到对应的 **MappedStatement**，然后根据传入参数对象对 **MappedStatement** 进行解析，解析后可以得到最终要执行的 **SQL** 语句和参数

3） **SQL 执行**

将最终得到的 SQL 和参数拿到数据库进行执行，得到操作数据库的结果

4） **结果映射**

将操作数据库的结果按照映射的配置进行转换，可以转换成 **HashMap**、**JavaBean** 或者**基本数据类型**，并将最终结果返回
--摘自《**Mybatis**》百度百科

## 05. 执行流程-源码分析

**Mybatis** 的对数据库的底层操作，如 **Spring** **JDBC** 一样，实际上也是对 **JDBC** 的一个封装，它的底层原理依旧是 **JDBC**。所以依然遵从 **JDBC** 的 **6** 个步骤

a 加载数据库驱动

b 建立链接

c 创建 statement

d 执行 SQL 语句

e 处理结果集

f 关闭数据库

通过跟踪使用 **xml** 方式的查询方法源码，我们一路走下来是这样的：

1）建立连接

![](https://files.mdnice.com/user/13208/a00bf80c-76cd-4c58-a95f-98ec61e95978.png)

2）获取 **xml** 文件与接口中的对应方法

![](https://files.mdnice.com/user/13208/cd348dd1-ad1a-497f-9bb8-fc064c9faf20.png)

3）创建 **statement**，并执行 **sql** 语句，返回结果集

![](https://files.mdnice.com/user/13208/94fc59f5-2f70-42b2-a02c-11c9607ee9b9.png)

![](https://files.mdnice.com/user/13208/80ee2fd0-5611-4059-8747-0e6591d49494.png)

4）在这几步结束之后，会关闭 **statement**

5）关闭连接

![](https://files.mdnice.com/user/13208/22877bf4-7d76-47c0-b630-83ce614096b1.png)

其中，我们需要知道的是，**Spring** 会根据我们引入的数据库类型，预先加载数据库的驱动。因此 **Mybatis** 执行数据库的流程，实际上就是 **JDBC** 的 **6** 大步骤

## 06. 得力助手

针对 **Mybatis** 的一些较为繁琐的方面，也分别有几个好用的工具

- mybatis-代码生成器

- 分页插件

- mybatis-plus

使用它们，可以大大提高 **Mybatis** 的开发效率，感兴趣可以了解一下。

嗯，怎么说呢，它们的好，谁用谁知道呢~

![](https://files.mdnice.com/user/13208/2bd7cdeb-1932-48d8-93e1-611c686dfe78.png)

# 二 实战:实战环节

对原理有个大概的了解，接下来，就进入今天的实战环节啦~

本次实战依旧演示操作 **mysql** 数据库-完成 **Spring Boot** 使用 **Mybatis** 操作数据库的例子。

## 01. 引入依赖

只需要引入 **mybtis** 依赖和 **mysql** 驱动两个主要依赖

```
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.2</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

## 02. 属性配置

分别进行数据库连接信息以及 **mybatis** 的相关信息配置

以下分别为大家演示了表字段与实体类属性的映射支持下划线对应驼峰的方式机制、指定 **mapper.xml** 文件位置、以及配置日志输出这 **3** 个常用的配置项

### 1) 数据库连接信息配置

```
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/test

?characterEncoding=utf8

spring.datasource.username=root
spring.datasource.password=123456
### 2）开启下划线的表字段映射为驼峰格式的实体类属性
#mybatis.configuration.map-underscore-to-camel-case=true
### 3）用于指定xml文件的位置（使用xml执行sql语句的方式时）
mybatis.mapper-locations=classpath:mapper/*.xml
### 4）打印sql语句
mybatis.configuration.log-impl=org.apache.ibatis.logging

.stdout.StdOutImpl
```

## 03. 创建实体类

使用之前文章中提到过的一个好用的开发工具 lombok（如果想要了解用法，可以参考文章 **SpringBoot（二）：第一个 Spring Boot 项目**），我们建立好的实体类 **User** 如下：

```
@Builder
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User implements Serializable {
    private long id;
    private String name;
    private int age;
    private Date createTime;
    private Date updateTime;
}
```

## 04. 创建接口

针对在第一节中提到的三种操作方法，我们分别进行讲解

### 1）注解

**Mybatis** 提供了很多操作注解，如下

![](https://files.mdnice.com/user/13208/4ab7283d-f68d-4df2-97e9-5e0b57cd7bba.png)

**Mybatis** 的注解方式，是使用这些操作注解，将 **sql** 语句写入这些注解中，进行数据库的操作的（其中结尾为 **Provider** 的注解，用于语句构建器方式）

我们只需要在接口上添加 **@Mapper** 注解，然后直接使用这些注解进行方法的定义

![](https://files.mdnice.com/user/13208/20a6c06f-5f57-446e-87fc-78ae712db7fa.png)

我们分别定义了数据库的新增和根据 id 查找记录这两个方法

### 2）语句构建器

**a 创建 UserBuilder**

实现接口 **ProviderMethodResolver**

```
public class UserBuilder  implements ProviderMethodResolver {

    public String deleteUserSql() {
        return new SQL() {{
            DELETE_FROM("user");
            WHERE("ID = ${id}");
        }}.toString();
    }
    public static String selectUserById() {
        return new SQL()
                .SELECT("name,age,create_time,update_time")
                .FROM("user")
                .WHERE("id = #{id}")
                .toString();
    }
}
```

**b 创建接口**

定义方法时，采用对应的 **Provider** 注解的方式，指定类型 **type** 为我们的语句构建类，然后定义与语句构建类中的方法名一致的方法

```
/**
 * 使用语句构造器方式定义数据库操作接口
 */
@Mapper
public interface UserMapperWithBuilder {

    @SelectProvider(type = UserBuilder.class)
    User selectUserById(long id);
}
```

### 3）xml 方式

**xml 方式**，使我们经常使用的方式，因为它对于项目开发来说，还是比较灵活的。

**a 建立与接口映射的 xml 文件 UserMapper.xml**

![](https://files.mdnice.com/user/13208/f623927f-a06f-4ee8-91b6-20987c9dd8e0.png)

**b 建立相应的映射接口**

```
/**
 * 使用xml方式定义数据库操作接口
 */
public interface UserMapperWithXml {

    List<User> getUserList(Map<String, Object> map);
}
```

温馨提示：需要注意以下几点，否则程序执行的时候会找不到

这个文件的放置位置应该与配置文件中所配置的路径一致

**mapper** 标签中的 **namespace** 属性要指定为对应的映射接口 **UserMapperWithXml**

各个方法的 **id** 要与接口中的法名保持一致，同时相应的输入输出参数保持一致

## 05. 调用接口方法

1）在启动类上添加注解，用于扫描我们所定义的数据库操作接口

```
@MapperScan("com.shumile.springbootmybatis.mapper")
```

接着，分别通过方法 **testBuilder**、**testByMapper** 以及 **testByXml** 调用以上所定义的各个数据库方法

```
@SpringBootApplication
@Slf4j
@MapperScan("com.shumile.springbootmybatis.mapper")
//@EnableTransactionManagement
public class SpringBootMybatisApplication implements

ApplicationRunner {


    @Autowired
    private UserMapperWithAnnotation userMapperWithAnnotation;
    @Autowired
    private UserMapperWithXml userMapperWithXml;
    @Autowired
    private UserMapperWithBuilder userMapperWithBuilder;
    public static void main(String[] args) {
        SpringApplication.

                run(SpringBootMybatisApplication.class, args);

}

    @Override
    @Transactional
    public void run(ApplicationArguments args) throws Exception {
        //testByMapper();
        testByXml();
        //testBuilder();
    }
    private void testBuilder(){
        userMapperWithBuilder.selectUserById(18l);
    }


    private void testByXml() {
        Map<String, Object> map = new HashMap<>();
        map.put("name","小米");
        map.put("age",17);
        List<User> userList = userMapperWithXml.getUserList(map);
        userList.forEach(user->log.info("user:{}",user));
    }

    private void testByMapper() {
        User c = User.builder().name("小米").age(17)
                .createTime(new Date())
                .updateTime(new Date()).build();
        int count = userMapperWithAnnotation.save(c);
        log.info("Save {} User: {}", count, c);
        c = userMapperWithAnnotation.findById(c.getId());
        log.info("Find User: {}", c);
    }


}
```

## 06. 运行项目

运行项目，你就可以看到各个功能纷纷都实现啦，演示一下通过 xml 方式实现的日志打印效果

![](https://files.mdnice.com/user/13208/a351e754-8214-4188-980a-171b6396c8e4.png)

我们可以看到，日志中不仅为我们打印出来了格式化以后的 sql 语句，还将输出了 **sqlSession** 的创建与关闭等信息。所以在平时采用配置文件中所配置的日志输出方式还是挺好用的呢。

当然还有一种专门用于数据库监控的 **p6Spy** 工具，也挺好用的，感兴趣的话，可以搜索一下用法哦~

如果跟着文章的思路一路走到这里的话，那么恭喜你， **Mybatis** 的三种操作数据库的方式，你已经全部学会啦~

![](https://files.mdnice.com/user/13208/e10592ea-7b77-491b-a5db-f0f7a67584a0.png)

# 三 总结：总而言之

总体来说，使用语句构建器的方式操作数据库，采用具体的 sql 语句对应的方法来替换具体 sql 关键词，可以简化代码中直接书写的动态 sql 语句，并且还可以支持随意更换数据库，但总得来说，还是不够灵活

使用 **XML** 或**注解**的方式来配置 **ORM**，把 **SQL** 用标签管理起来，不关心，也不干涉实际 **SQL** 的书写。在这种思路下框架轻量，很容易集成，又因为我们可以使用 SQL 所有的特性，可以写存储过程，也可以写 SQL 方言（Dialect），所以灵活度相当高。

那么为什么我们更倾向于使用 xml 与接口映射的方式呢？

使用 **xml** 有以下**两大优点**：

- **SQL 与代码分离**。可以把所有的 sql 写在 xml 文件中，要是有什么修改，甚至是换数据库的操作，我们只需要更换 xml 文件就行了

- **复杂语句的实现方式灵活**。针对各种复杂的 sql、以及 sql 可能实现起来比较麻烦的情况，我们也可以直接使用存储过程等多种方式来实现

虽然步骤相对比较麻烦，但是它最为灵活，也最为简洁，代码看起来更清爽，使用久了，也并不会觉得麻烦了。

![](https://files.mdnice.com/user/13208/4abce0e9-6477-4595-bb0d-a1abb3375d8b.png)

今天一起重新学习了 **Mybatis** ，总体来说，有以下几点：

**1、Spring Boot 集成 Mybatis 的加载流程**

**2、Mybatis 操作数据库的三种方式**

**3、三种操作方式各自的特点**

嗯，就这样。每天学习一点，时间会见证你的成长

下期预告：

**Spring Boot(六)：那些好用的连接池们~**

本期项目代码已上传到 github~ 有需要的可以参考
**https://github.com/wangjie0919/Spring-Boot-Notes**

`Spring Boot`往期系列文章回顾：

[`Spring Boot`（四）：让人又爱又恨的 JPA](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483799&idx=1&sn=f0a553daf412aefeb2f7877970b04aca&chksm=96eb0386a19c8a90c15b1dae13d39546a2d01875d3f3489c2e2f709a97f8f8ca87bee4ec58e0&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（三）：操作数据库-Spring JDBC](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483779&idx=1&sn=6ce5bbd2d8028b176ecf3b1b7fa8f3ea&chksm=96eb0392a19c8a84f1e7356a36012a2ef2f17f4d70f60c83ea42c1e72ed5a561c18d1782577e&scene=21#wechat_redirect)

[`Spring Boot`（二）：第一个`Spring Boot`项目](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483751&idx=1&sn=90bb32f21dbfd6793b899ac4cb04d52f&chksm=96eb0376a19c8a60eb9ea0b8227b3e34d55efb0feb6e1aa944d90514333f2be70635b8bc820c&scene=21#wechat_redirect)

[`Spring Boot`（一）：特性概览](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483725&idx=1&sn=c1399fcb817d5a434f272b93abd36798&chksm=96eb035ca19c8a4ac715cf62f4cfe301e0124e7b800c87b3ae78503190bbb8ec5e2754ce3fd4&scene=21#wechat_redirect)
