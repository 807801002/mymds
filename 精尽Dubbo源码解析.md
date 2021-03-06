# 项目结构一览

## dubbo-common

提供工具类和通用模型

## dubbo-remoting

远程通信模块，提供通用的客户端和服务端的通讯功能

## dubbo-rpc

远程调用模块，抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理

集群相关的管理，有`dubbo-cluster`提供特性

## dubbo-cluster

集群模块：将多个服务提供方伪装为一个提供方，包括:负载均衡、集群容错、路由、分组聚合等。集群的地址列表可以是静态配置的，也可以由注册中心下发

* 容错
* 目录
* 路由
* 配置
* 负载均衡
* 合并结果

## dubbo-registry

注册中心模块：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象

## dubbo-monitor

监控模块：统计服务调用次数，调用时间的，调用链路跟踪的服务

## dubbo-config

配置模块：是Dubbo对外的API，用户通过Config使用Dubbo，隐藏Dubbo所有细节

## dubbo-container

容器模块：是一个Standlone的容器，以简单的Main加载Spring启动，因为服务通常不需要Tomcat/JBoss等Web容器的特性，没必要用Web容器去加载服务

## dubbo-filter

过滤器模块，提供了内置过滤器

## dubbo-plugin

提供了内置插件

## hessian-lite

Dubbo对Hessian2的序列化部分的精简、改进、Bugfix

## dubbo-demo

## dubbo-test

## Maven POM



# API配置

## 1.概述

Dubbo的配置是非常“灵活”的

四种配置方式

* API配置
* 属性配置
* XML配置
* 注解配置

所有配置项分三大类

* 服务发现：表示该配置项用于服务的注册于发现，目的是让消费方找到提供方
* 服务治理：表示该配置项用于治理服务间的关系，或为开发测试提供和便利条件
* 性能调优：表示该配置项用于调优性能，不同的选项对性能产生影响

# 核心流程

# 扩展机制SPI

# 线程池

# 服务暴露

# 服务引用

# 注册中心

# 动态编译

# 动态代理

# 服务调用

# 调用特性

# 过滤器

# NIO服务器

# HTTP服务器

# 序列化

# 服务容错

# 集群容错

