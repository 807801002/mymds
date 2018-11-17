# 为什么阅读

## 为什么阅读

* 深入了解MQ，知其然知其所以然，如何实现高性能、高可用
* 最终一致性，是如何通过MQ实现
* 了解Netty在分布式中间件如何实现网络通信以及各种异常场景的处理
* 了解**MQ消息存储**，特别是**磁盘IO**部分
* 通过阅读源码，突破技术

## 步骤

* nameserv启动
* broker启动
* producer启动
* consumer启动
* 消息模型
  * 消息唯一编号
* producer发消息
* broker收消息
* broker发消息
* consumer收消息
  * 多消费者
  * 重试消息
* consumer消息确认
* consumer负载均衡
* broker队列模型
* broker store消息存储
* 顺序消息
* 事务消息
* 定时（延迟）消息
* pub/sub模型
* namesrv集群
* broker主从
* filtersrv过滤消息
* remoting调用（server、client）
* 跨机房
* Hook机制
* Tool-Admin
* Tool-Command
* Tool-Monitor
* broker主备切换

# NameServ

## NameServ组件

* KVConfigManager：KV配置管理
  * key-value配置管理，增删改查
* RouteInfoManager，路由信息管理
  * 注册Broker，提供Broker信息（名字、角色编号、地址、集群名）
  * 注册Topic，提供Topic消息（Topic名、读写权限、队列情况）

# Topic

# Message

# 高可用

# 定时消息与消息重试

# Filtersrv

# 事务消息

# RPC通信