---
title: Spring Boot（七）：你不能不知道的Mybatis缓存机制！
author: 杰哥
pubDatetime: 2023-08-21T16:08:06.130Z
postSlug: spring-boot-7
featured: false
draft: false
tags:
  - Spring Boot
  - Java
  - 微服务
  - 源码解析
  - 连接池
  - Mysql
  - MyBatis

description: 缓存的重要性是不言而喻的。使用缓存， 我们可以避免频繁的与数据库进行交互， 尤其是在查询越多、缓存命中率越高的情况下， 使用缓存对性能的提高更明显。同样地，**MyBatis** 作为 **ORM** 框架，也必然会支持缓存
---

缓存的重要性是不言而喻的。使用缓存， 我们可以避免频繁的与数据库进行交互， 尤其是在查询越多、缓存命中率越高的情况下， 使用缓存对性能的提高更明显。

同样地，**MyBatis** 作为 **ORM** 框架，也必然会支持缓存

它分别支持**一级缓存**和**二级缓存** 。其中一级缓存是 **sqlSession** 级的缓存，而二级缓存则可以实现多个 **sqlSession** 间的缓存

什么意思？往下看喽~

# 一 一级缓存

## 01. 什么是一级缓存

之所以说 **MyBatis** 的一级缓存是 **sqlSession** 级的，是因为它只支持同一个 **sqlSession** 下的缓存，也就是说缓存只在同一个 **sqlSession** 之间共享

## 02.如何实现一级缓存

一级缓存分为两个范围：**statement** 和 **session**

**statement**：说白了就是一个 sql 语句

**session**：有数据库产生的一个连接，即一个 **sqlsession**

而 **MyBatis** 默认支持一级缓存，不需要专门进行配置，并且它支持 **session** 范围的一级缓存。

针对缓存属性，**MyBatis** 通过类 **org.apache.ibatis.sessionConfiuration** 进行了配置，我们可以看到 **localCacheScope** 的默认级别为 **SESSION**（并且二级缓存的也是默认开启的）

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/image.png)

注意：**Configuration** 类中的 **cacheEnabled** 属性是针对**二级缓存**的开关控制，而不是针对**一级缓存**的。**一级缓存**完全不需要进行配置，它并没有开关，是 **MyBatis** 默认支持的

那么，也就是说，我直接运行服务，一级缓存就生效了？

试试看，进入一级缓存测试环节~

## 03. 一级缓存测试

**1）在同一个方法中调用两次相同的方法**

```
public void testCache(){
    User user = userMapperWithAnnotation.findById(33L);
    log.info("Find User: {}", user);
    User user2 = userMapperWithAnnotation.findById(33L);
    log.info("Find User2: {}", user2);
}
```

**2）直接在 run 方法中进行调用**

```
@SpringBootApplication
@Slf4j
@MapperScan("com.shumile.springbootmybatis.mapper")
public class SpringBootMybatisApplication

implements ApplicationRunner {


    @Autowired
    private UserMapperWithAnnotation userMapperWithAnnotation;

    public static void main(String[] args) {
        SpringApplication.run(SpringBootMybatisApplication.class, args);
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
          testCache();
    }
```

我们预期的效果是，既然 **MyBatis** 默认已经支持一级缓存，那么我执行两个一模一样的方法，肯定只需要查询一次数据库了，第二次就应该直接从缓存中取结果了

**3）运行代码，如下图所示**

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/11.png)

哈？什么情况？骗人呢？打印了两条 sql 语句，这不还是查询了两次吗？

说实话，这个问题曾经困扰了我好几个小时，我在想，难道网上说的都有问题吗？是不是对于一级缓存，还专门有什么特殊配置呢？

最后呢，通过源码的跟踪，终于豁然开朗~

## 04. 源码分析

我们跟踪 **findById**() 方法的执行，步骤如下

1）首先会进入 **DefaultSqlSession** 的 **selectOne**()方法

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/22.png)

2）接着，会进入 **selectList**()方法

我们知道在 **Confifuration** 类中已经默认配置了一级缓存的支持范围，在执行查询语句的时候，我们拿到的对应配置对象 就是默认支持 **session** 范围内的缓存（依旧先不用管 **cacheEnabled** 属性）。

那么，这块是没有问题的

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/没有问题.png)

3）往下走，进入 **CachingExecutor** 的 **query**()方法

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/querY 方法.png)

根据参数，获取到对应的可执行 **sql** 语句之后，进入创建缓存 key 的方法 **createCacheKey**()，并将获取到的 key 作为参数传给下面的 **query**()方法

在这里，先暂停，我们进入 **createCacheKey**() 方法内部，看看这个 Key 是如何确定的

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/如何确定.png)

**4）createCacheKey()实现探究**

它通过分别执行 cacheKey.update() 方法，将 Statement.id、Offset、Limmit、Sql 以及 Params，这五个属性分别，放入 cacheKey 中的 updateList 中。其中 update()方法如下

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/如何确定.png)

再来看看，**cacheKey.equals**()方法

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/等于方法.png)

很显然，除去 hashcode，checksum 和 count 的比较外，只要 updatelist 中的元素对应相等了，就可以认为是 CacheKey 相等。也就是说，只要两条 Sql 的 Statement.id、Offset、Limmit、Sql 以及 Params 这五个值相同，即可以认为是相同的 Sql

看完了对于 key 的组装环节，接着继续往下走~

**5）在缓存中找 key**

往下执行，在 query()方法中，会首先通过 key 值去缓存中取，如果缓存中没有，即获取到的 list 为空，则需要去数据库中查询

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/中查询.png)

**6）执行具体的数据库查询方法**

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/查询方法.png)

**7）将结果存入缓存**

执行以后，会通过 localCache.putObject(key,list) 将当前执行的结果放在本地缓存中

**8）提交结果**

这一步结束以后，便进入到了 **commit**()方法

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/提交方法.png)

我们看到，在执行 commit()方法时，会清空本地缓存。那么以后再次查询时，缓存中总是找不到对应的 key 值，就会出现每次都重新执行 sql 语句，去数据库中查询的现象了

那么，我们便很容易就知道了，为什么会不支持一级缓存了。

原来，要想支持一级缓存，就得要保证在这些 sql 语句全部执行完以后，再去执行 commit()方法，也就是说，我们的方法必须要在同一个事务内，才会支持

那就一起来验证一下，开启了事务的情况~

**1）开启事务**

接下来，我们对方法开启事务，在启动类添加 @EnableTransactionManagement 注解，并在 run（）方法上添加 @Transactional 注解

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/注解.png)

然后进入验证环节

**2）验证环节**

第一次查询的执行过程，跟上面的基本一样。同样是从数据库中查询得到结果，并将结果存放到缓存中

第二次查询

注意了，关键就在第二次查询

会继续判断从缓存中取对应 key 的值，这次我们可以取到 key 的 value 值，即它的查询结果，直接将这个结果集返回即可

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/即可.png)

直到执行完两条语句之后，进入 commit()方法，进行事务提交操作

看下执行结果图

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/结果图.png)

sql 语句只执行了一次，那么说明验证成功~

## 小结

为什么会出现不开启事务时，一级缓存不生效；开启了事务，一级缓存生效？

很显然，那是因为它每条语句执行结束以后，都会执行提交方法，而提交方法在每次都会清空本地缓存。而开启了事务的话，方法是在所有操作结束以后才会提交，因此就会支持一级缓存啦

# 二 二级缓存

## 01. 什么是二级缓存

一级缓存中，是一个 sqlSession 使用一个缓存，而 MyBatis 的二级缓存，则支持多个 SqlSession 之间共享缓存。MyBatis 默认开启二级缓存

## 02. 如何二级缓存

从第一节中对一级缓存的源码分析中，我们也提到了，在 Configuration 类中，已经默认开启了二级缓存（cacheEnabled=true）

除了这个参数，还需要在 mapper.xml 文件中，添加 cache 或者 cache-ref 标签，进行缓存配置

如下所示

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/所示.png)

cache 标签用于声明 namespace 使用二级缓存，并且可以通过以下属性自定义配置

- **type**：cache 使用的类型，默认是 PerpetualCache

- **eviction**：定义回收的策略，常见的有 FIFO，LRU

- **flushInterval**：配置一定时间自动刷新缓存

- **size**：最多缓存对象的个数

- **readOnly**：是否只读，若配置可读写，则需要对应的实体类能够序列化

- **blocking**：配置若缓存中找不到对应的 key，是否会一直 blocking，直到有对应的数据进入缓存

cache-ref 代表引用别的命名空间的 Cache 配置，两个命名空间的操作使用的是同一个 Cache

要想实现两个命名空间共享缓存，那么可以 cache-ref 标签的 namespace 属性引入另一个命名空间，如：

```
<cache-ref namespace="mapper.OrderMapper"/>
```

--标签说明参考自：

https://tech.meituan.com/2018/01/19/mybatis-cache.html

## 03. 二级缓存测试

要想验证 MyBatis 的二级缓存，就需要构造多个不同 sqlsession 的操作，验证在这些 sqlsession 之间能够实现缓存共享

**1）创建 mybatis-configuration.xml 文件**

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="logImpl" value="STDOUT_LOGGING"/>
        <!--二级缓存开关-->
        <setting name="cacheEnabled" value="true" />
<!--        <setting name="localCacheScope" value="false"/>-->
    </settings>
    <!--环境配置，连接的数据库，-->
    <environments default="mysql">
        <environment id="mysql">
            <!--指定事务管理的类型-->
            <transactionManager type="JDBC"></transactionManager>
            <!--dataSource 指连接源配置，POOLED是JDBC连接对象的数据源连接池的实现-->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"></property>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/test"></property>
                <property name="username" value="root"></property>
                <property name="password" value="123456"></property>
            </dataSource>
        </environment>
    </environments>
    <mappers>

        <mapper resource="mapper/UserMapper.xml"></mapper>
    </mappers>
</configuration>
```

**2）我们通过这个配置文件，分别定义两个 sqlsession,进行同样的查询操作**

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/查询操作.png)

调用方法执行结果如下

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/查询操作.png)

我们看到，并没有实现缓存的共享，即二级缓存失效了

**3）第二次测试**

我们在执行第二个 sqlsession 的操作之前，先执行 sqlSession1 的提交

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/sqlsession1的提交.png)

执行代码，如下所示

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img_sp7/end.png)

我们看到，二级缓存成功实现了

咦~这又是为什么呢？

这是因为呢，只有第一个 sqlSession 执行了提交操作，第二个 sqlSession 才能感应到，然后才能获取到这个缓存

那么，如果我们的数据发生了变化，它肯定就不能再去取缓存中的数据了，否则我们在页面中看到的数据就不是最新的了

这一点，MyBatis 也为我们考虑到了，我们来看看如果我们在操作中间，执行了更新表的操作，情况是怎样的

**4）第三次测试**

如下所示，在两个查询操作之间，我们执行了另一个 sqlSession 的 update()方法

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/update方法.png)

执行结果，如下所示

![](https://cdn.jsdelivr.net/gh/helemile/Spring-Boot-Notes@dependabot/maven/activiti/activiti_demo/junit-junit-4.13.1/img/end2.png)

在最后一次查询操作时，同样执行了 sql 语句，那么，这种情况下，二级缓存就失效了

此外，如果我们在执行过程中，执行了多表操作，即如果 A 表和 B 表相关联，若对 A 表执行了更新操作，B 表并不能够感知到，从而会拿到脏数据，影响正常的业务逻辑

二级缓存的实现原理，大家可以参照一级缓存中跟踪源码的方式，自己跟着代码再进行更进一步的探究，有任何问题也可以随时与我留言沟通哦~

## 小结

mybatis 通过 **cacheEnabled** 来进行二级缓存的开启与关闭配置，默认是**开启**状态

使用时，还需要在 mapper.xml 中通过 **cache** 标签开启命名空间内部的缓存。当然也可以通过 **cache-ref** 加入其它命名空间，进行二级缓存的共享

MyBatis 对多表查询有局限性，容易出现读到脏数据。建议使用第三方缓存实现

# 三 总结 总而言之

今天主要聊了聊 **MyBatis** 的**一级缓存**和**二级缓存**

mybatis 默认支持一级缓存，不需要开关设置，但必须是在事务开启的情况下才会生效，具体原因，我们也通过跟踪查询语句的执行过程理解了

而二级缓存，必须在前面的 sqlSession 提交事务之后，才能够支持，并且使用具有局限性，每一个 sqlSession 执行完之后，必须进行提交操作，其他 sqlSession 才 能感应到变化，对于多表操作容易拿到脏数据等缺陷。

个人更建议在项目使用集中式缓存，比如使用 redis 进行数据的存储，对于分布式的场景，我们也能够保证缓存不失效，并且不会读到脏数据，从而保证业务一致性

技能总结

**1、MyBatis 一级缓存的实现方式与实现原理**

**2、MyBatis 二级缓存的实现方式与失效场景**

**3、两级缓存使用情况总结与建议**

温馨提示

项目中如果出现一些不符合认知的代码逻辑问题，除了四处搜索之外，也不妨试试跟踪源码，一步步分析解决。这样不仅了解了其中的原理，同时还会极大锻炼你解决问题的能力哦~建议试试，相信我，时间久了，你会觉得代码其实也是香的呢

嗯，就这样。每天学习一点，时间会见证你的强大。

下期预告：

Spring Boot（八）：Spring Boot 的监控法宝：Actuator

本期项目代码已上传到 github~有需要的可以参考
https://github.com/wangjie0919/Spring-Boot-Notes

往期精彩回顾

[`Spring Boot`（六）：Spring Boot（六）：那些好用的数据库连接池们](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483845&idx=1&sn=b604205c54cbf132a0a3a0d9fd035bba&chksm=96eb03d4a19c8ac207645109ed636d89d781daa27bf5e1290393f04ef45aa9453073d4fa9c76&token=1019255373&lang=zh_CN#rd)

[`Spring Boot`（五）：春眠不觉晓，Mybatis 知多少](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483821&idx=1&sn=7c5ed87610b93589266106d1b07191d1&chksm=96eb03bca19c8aaa5967520fe03381481020450834e7b4e518dd729dc38b71c4cb50154d7e8d&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（四）：让人又爱又恨的 JPA](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483799&idx=1&sn=f0a553daf412aefeb2f7877970b04aca&chksm=96eb0386a19c8a90c15b1dae13d39546a2d01875d3f3489c2e2f709a97f8f8ca87bee4ec58e0&token=1060450670&lang=zh_CN#rd)

[`Spring Boot`（三）：操作数据库-Spring JDBC](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483779&idx=1&sn=6ce5bbd2d8028b176ecf3b1b7fa8f3ea&chksm=96eb0392a19c8a84f1e7356a36012a2ef2f17f4d70f60c83ea42c1e72ed5a561c18d1782577e&scene=21#wechat_redirect)

[`Spring Boot`（二）：第一个`Spring Boot`项目](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483751&idx=1&sn=90bb32f21dbfd6793b899ac4cb04d52f&chksm=96eb0376a19c8a60eb9ea0b8227b3e34d55efb0feb6e1aa944d90514333f2be70635b8bc820c&scene=21#wechat_redirect)

[`Spring Boot`（一）：特性概览](https://mp.weixin.qq.com/s?__biz=MzIwMTY3NjY3MA==&mid=2247483725&idx=1&sn=c1399fcb817d5a434f272b93abd36798&chksm=96eb035ca19c8a4ac715cf62f4cfe301e0124e7b800c87b3ae78503190bbb8ec5e2754ce3fd4&scene=21#wechat_redirect)
