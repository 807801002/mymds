# [Java 面试题 —— 老梁 Java 面试通关要点汇总集](http://www.iocoder.cn/Interview/laoliang/)



 总阅读量:4491次

[自我表扬：《Dubbo 实现原理与源码解析 —— 精品合集》](http://www.iocoder.cn/Dubbo/good-collection/?title) 
[表扬自己：《D数据库实体设计合集》](http://www.iocoder.cn/Entity/good-collection/?title)

摘要: 原创出处 https://blog.720ui.com/2018/java_interview_final/ 「老梁」欢迎转载，保留摘要，谢谢！

- 基础篇
  - [基本功](http://www.iocoder.cn/Interview/laoliang/)
  - [集合](http://www.iocoder.cn/Interview/laoliang/)
  - [线程](http://www.iocoder.cn/Interview/laoliang/)
  - [锁机制](http://www.iocoder.cn/Interview/laoliang/)
- 核心篇
  - [数据存储](http://www.iocoder.cn/Interview/laoliang/)
  - [缓存使用](http://www.iocoder.cn/Interview/laoliang/)
  - [消息队列](http://www.iocoder.cn/Interview/laoliang/)
- 框架篇
  - [Spring](http://www.iocoder.cn/Interview/laoliang/)
  - [Netty](http://www.iocoder.cn/Interview/laoliang/)
- 微服务篇
  - [微服务](http://www.iocoder.cn/Interview/laoliang/)
  - [分布式](http://www.iocoder.cn/Interview/laoliang/)
- 安全&性能
  - [安全问题](http://www.iocoder.cn/Interview/laoliang/)
  - [性能优化](http://www.iocoder.cn/Interview/laoliang/)
- 工程篇
  - [需求分析](http://www.iocoder.cn/Interview/laoliang/)
  - [设计能力](http://www.iocoder.cn/Interview/laoliang/)
  - [设计模式](http://www.iocoder.cn/Interview/laoliang/)
  - [业务工程](http://www.iocoder.cn/Interview/laoliang/)
  - [软实力](http://www.iocoder.cn/Interview/laoliang/)
- [HR 篇](http://www.iocoder.cn/Interview/laoliang/)

------

------

首先，声明下，以下知识点并非阿里的面试题。这里，笔者结合自己过往的面试经验，整理了一些核心的知识清单，帮助读者更好地回顾与复习 Java 服务端核心技术。本文会以引出问题为主，后面有时间的话，笔者陆续会抽些重要的知识点进行详细的剖析与解答。

------

## 基础篇

### 基本功

- 面向对象的特征

  >三个基本特征
  >
  >* 封装  就是把客观事物封装成抽象的类，并且类可以把自己的数据和方法只让可信的类或者对象操作，对不可信的进行信息隐藏
  >* 继承 可以使用现有类的所有功能，并在无需重新编写原来的类的情况下对这些功能进行扩展
  >* 多态 允许将子类类型的指针赋值给父类类型的指针；实现多态有两种方式：覆盖，子类重新定义父类的虚拟函数的做法；重载，指允许存在多个同名函数，这些函数参数表不同
  >
  >五种设计原则
  >
  >* 单一职责原则
  >* 开放封闭原则
  >* Liskov替换原则
  >* 依赖倒置原则
  >* 接口隔离原则

- final, finally, finalize 的区别

- int 和 Integer 有什么区别

- 重载和重写的区别

- 抽象类和接口有什么区别

- 说说反射的用途及实现

  > 当程序运行时，允许改变程序结构或变量类型，这种语言称之为动态语言。我们认为Java并不是动态语言，但它却有一个非常突出的动态相关的机制——反射。
  >
  > Reflection是Java程序开发语言的特征之一，它允许运行中的Java程序获取自身的信息，并且可以操作类和对象的内部属性。
  >
  > 反射的核心：JVM运行时才动态加载的类或调用方法或属性，不需要事先3（写代码的时候或编译期）知道运行对象是谁。
  >
  > 反射的用途：反射最重要的用途就是开发各种通用框架。如下是struct.xml配置
  >
  > ```xml
  > <action name="login"
  >            class="org.ScZyhSoft.test.action.SimpleLoginAction"
  >            method="execute">
  >        <result>/shop/shop-index.jsp</result>
  >        <result name="error">login.jsp</result>
  >    </action>
  > ```
  >
  > 配置文件与Action建立了一种映射关系，当View层发出请求时，请求会被StrutsPrepareAndExecuteFilter拦截，然后StrutsPrepareAndExecuteFilter会去动态地创建Action实例。

- 说说自定义注解的场景及实现

- HTTP 请求的 GET 与 POST 方式的区别

  > GET和POST本质上没有区别你信吗？ 
  >
  > GET和POST能做的事情是一样一样的。你要给GET加上request body，给POST带上url参数，技术上是完全行的通的
  >
  > GET产生一个TCP数据包；POST产生两个TCP数据包。

- session 与 cookie 区别

  > session是存储在服务器端的，cookie是存储在客户端的，session依赖于cookie

- session 分布式处理

  https://my.oschina.net/u/1774673/blog/871912

  > 集群中存在A、B两台服务器，用户在第一次访问网站时，Nginx通过其负载均衡机制将用户请求转发到A服务器，这时A服务器就会给用户创建一个Session。当用户第二次发送请求时，Nginx将其负载均衡到B服务器，而这时候B服务器并不存在Session，所以就会将用户踢到登录页面。
  >
  > * 第一种：粘性session
  >
  > 是指将用户锁定到某一个服务器上。**优点**：简单，无需对session做任何处理。**缺点**：缺乏容错性，机器故障将被转移到另一台机器上，session信息都将失效。**使用场景：**发生故障对客户影响较小；服务器低概率故障事件。**实现方式**：以nginx为例，在upstream模块配置ip_hash属性即可实现粘性session
  >
  > * 第二种：服务器session复制
  >
  > 把session的所有内容序列化，广播所有节点，保证session同步。**优点**：可容错。**缺点**：对网络负荷造成一定压力，如果session量大可能造成网络堵塞，拖慢服务器性能。**实现方式**：1.设置tomcat，server.xml开启tomcat集群功能，填写本机IP寄客人，设置端口号，防止端口冲突。2.在应用里增加信息：通知应用当前处于集群环境中，支持分布式在web.xml中添加选项<distribuable/>
  >
  > * 第三种：session共享机制
  >
  > 使用分布式缓存方案如memcached、Redis，但是要求Memcached或Redis必须是集群。使用Session共享也分为两种机制
  >
  > * 第四种：session持久化到数据库
  > * 第五种:   terracotta实现session复制

- JDBC 流程

- MVC 设计思想

- equals 与 == 的区别

### 集合

- List 和 Set 区别
- List 和 Map 区别
- Arraylist 与 LinkedList 区别
- ArrayList 与 Vector 区别
- HashMap 和 Hashtable 的区别
- HashSet 和 HashMap 区别
- HashMap 和 ConcurrentHashMap 的区别
- HashMap 的工作原理及代码实现
- ConcurrentHashMap 的工作原理及代码实现

### 线程

- 创建线程的方式及实现
- sleep() 、join（）、yield（）有什么区别
- 说说 CountDownLatch 原理
- 说说 CyclicBarrier 原理
- 说说 Semaphore 原理
- 说说 Exchanger 原理
- 说说 CountDownLatch 与 CyclicBarrier 区别
- ThreadLocal 原理分析
- 讲讲线程池的实现原理
- 线程池的几种方式
- 线程的生命周期

### 锁机制

- 说说线程安全问题
- volatile 实现原理
- synchronize 实现原理
- synchronized 与 lock 的区别
- CAS 乐观锁
- ABA 问题
- 乐观锁的业务场景及实现方式

## 核心篇

### 数据存储

- MySQL 索引使用的注意事项
- 说说反模式设计
- 说说分库与分表设计
- 分库与分表带来的分布式困境与应对之策
- 说说 SQL 优化之道
- MySQL 遇到的死锁问题
- 存储引擎的 InnoDB 与 MyISAM
- 数据库索引的原理
- 为什么要用 B-tree
- 聚集索引与非聚集索引的区别
- limit 20000 加载很慢怎么解决
- 选择合适的分布式主键方案
- 选择合适的数据存储方案
- ObjectId 规则
- 聊聊 MongoDB 使用场景
- 倒排索引
- 聊聊 ElasticSearch 使用场景

### 缓存使用

- Redis 有哪些类型

  > Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

- Redis 内部结构

- 聊聊 Redis 使用场景

- Redis 持久化机制

- Redis 如何实现持久化

- Redis 集群方案与实现

- Redis 为什么是单线程的

- 缓存奔溃

- 缓存降级

- 使用缓存的合理性问题

### 消息队列

- 消息队列的使用场景
- 消息的重发补偿解决思路
- 消息的幂等性解决思路
- 消息的堆积解决思路
- 自己如何实现消息队列
- 如何保证消息的有序性

## 框架篇

### Spring

- BeanFactory 和 ApplicationContext 有什么区别
- Spring Bean 的生命周期
- Spring IOC 如何实现
- 说说 Spring AOP
- Spring AOP 实现原理
- 动态代理（cglib 与 JDK）
- Spring 事务实现方式
- Spring 事务底层原理
- 如何自定义注解实现功能
- Spring MVC 运行流程
- Spring MVC 启动流程
- Spring 的单例实现原理
- Spring 框架中用到了哪些设计模式
- Spring 其他产品（Srping Boot、Spring Cloud、Spring Secuirity、Spring Data、Spring AMQP 等）

### Netty

- 为什么选择 Netty

- 说说业务中，Netty 的使用场景

- 原生的 NIO 在 JDK 1.7 版本存在 epoll bug

  > epoll bug，它会导致Selector空轮询，最终导致CPU 100%。官方声称在JDK1.6版本的update18修复了该问题，但是直到JDK1.7版本该问题仍旧存在，只不过该BUG发生概率降低了一些而已，它并没有被根本解决。
  >
  > **Selector BUG出现的原因**
  >
  > 若Selector的轮询结果为空，也没有wakeup或新消息处理，则发生空轮询，CPU使用率100%，
  >
  > **Netty的解决办法**
  >
  > - 对Selector的select操作周期进行统计，每完成一次空的select操作进行一次计数，
  > - 若在某个周期内连续发生N次空轮询，则触发了epoll死循环bug。
  > - 重建Selector，判断是否是其他线程发起的重建请求，若不是则将原SocketChannel从旧的Selector上去除注册，重新注册到新的Selector上，并将原来的Selector关闭。

- 什么是TCP 粘包/拆包

- TCP粘包/拆包的解决办法

  > 1、定长包
  >
  > 2、包尾加\r\n（ftp）
  >
  > 3、包头加上包体长度
  >
  > 4、更复杂的应用层协议

- Netty 线程模型

- 说说 Netty 的零拷贝

- Netty 内部执行流程

- Netty 重连实现

## 微服务篇

### 微服务

- 前后端分离是如何做的
- 微服务哪些框架
- 你怎么理解 RPC 框架
- 说说 RPC 的实现原理
- 说说 Dubbo 的实现原理
- 你怎么理解 RESTful
- 说说如何设计一个良好的 API
- 如何理解 RESTful API 的幂等性
- 如何保证接口的幂等性
- 说说 CAP 定理、 BASE 理论
- 怎么考虑数据一致性问题
- 说说最终一致性的实现方案
- 你怎么看待微服务
- 微服务与 SOA 的区别
- 如何拆分服务
- 微服务如何进行数据库管理
- 如何应对微服务的链式调用异常
- 对于快速追踪与定位问题
- 微服务的安全

### 分布式

- 谈谈业务中使用分布式的场景
- Session 分布式方案
- 分布式锁的场景
- 分布是锁的实现方案
- 分布式事务
- 集群与负载均衡的算法与实现
- 说说分库与分表设计
- 分库与分表带来的分布式困境与应对之策

## 安全&性能

### 安全问题

- 安全要素与 STRIDE 威胁
- 防范常见的 Web 攻击
- 服务端通信安全攻防
- HTTPS 原理剖析
- HTTPS 降级攻击
- 授权与认证
- 基于角色的访问控制
- 基于数据的访问控制

### 性能优化

- 性能指标有哪些
- 如何发现性能瓶颈
- 性能调优的常见手段
- 说说你在项目中如何进行性能调优

## 工程篇

### 需求分析

- 你如何对需求原型进行理解和拆分
- 说说你对功能性需求的理解
- 说说你对非功能性需求的理解
- 你针对产品提出哪些交互和改进意见
- 你如何理解用户痛点

### 设计能力

- 说说你在项目中使用过的 UML 图
- 你如何考虑组件化
- 你如何考虑服务化
- 你如何进行领域建模
- 你如何划分领域边界
- 说说你项目中的领域建模
- 说说概要设计

### 设计模式

- 你项目中有使用哪些设计模式
- 说说常用开源框架中设计模式使用分析
- 说说你对设计原则的理解
- 23种设计模式的设计理念
- 设计模式之间的异同，例如策略模式与状态模式的区别
- 设计模式之间的结合，例如策略模式+简单工厂模式的实践
- 设计模式的性能，例如单例模式哪种性能更好。

### 业务工程

- 你系统中的前后端分离是如何做的
- 说说你的开发流程
- 你和团队是如何沟通的
- 你如何进行代码评审
- 说说你对技术与业务的理解
- 说说你在项目中经常遇到的 Exception
- 说说你在项目中遇到感觉最难Bug，怎么解决的
- 说说你在项目中遇到印象最深困难，怎么解决的
- 你觉得你们项目还有哪些不足的地方
- 你是否遇到过 CPU 100% ，如何排查与解决
- 你是否遇到过 内存 OOM ，如何排查与解决
- 说说你对敏捷开发的实践
- 说说你对开发运维的实践
- 介绍下工作中的一个对自己最有价值的项目，以及在这个过程中的角色

### 软实力

- 说说你的亮点
- 说说你最近在看什么书
- 说说你觉得最有意义的技术书籍
- 工作之余做什么事情
- 说说个人发展方向方面的思考
- 说说你认为的服务端开发工程师应该具备哪些能力
- 说说你认为的架构师是什么样的，架构师主要做什么
- 说说你所理解的技术专家

## HR 篇

- 你为什么离开之前的公司
- 你为什么要进我们公司
- 说说职业规划
- 你如何看待加班问题
- 谈一谈你的一次失败经历
- 你觉得你最大的优点是什么
- 你觉得你最大的缺点是什么
- 你在工作之余做什么事情
- 你为什么认为你适合这个职位
- 你觉得自己那方面能力最急需提高
- 你来我们公司最希望得到什么
- 你希望从这份工作中获得什么
- 你对现在应聘的职位有什么了解
- 您还有什么想问的
- 你怎么看待自己的职涯
- 谈谈你的家庭情况
- 你有什么业余爱好
- 你计划在公司工作多久

------

> 这里，特别感谢「寻找李先森」的提议与整理。

# 666. 彩蛋

如果你对 Java面试题 感兴趣，欢迎加入我的知识一起交流。