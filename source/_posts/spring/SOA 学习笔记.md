---
title: SOA 学习笔记
date: 2018-04-14 13:43:05
tags: SOA
categories: spring
---

## SOA 学习笔记

### SOA项目搭建篇

**主要目标是：完全清楚为什么要这么搭建，如何体现解耦、复用**

1. 大致的继承关系

![Alt text](./1534138305704.png)

* SOATopPom
>dependency parent是spring-boot-starter-parent
最基础的项目只包含一个pom.xml文件，里边主要是：
1、第三方的依赖
2、伺服地址配置
3、部署构件配置
4、对于SP项目来说，spring-boot-starter等aop、web包没有包含在这个pom.xml文件中

* SOAParent
>SOAParent是聚合项目（一个prj包含多个module）
主项目也只包含一个pom.xml文件，里边是业务层所需的依赖
module：
	IService - jar：只写具体的接口，不写实现
	bean - jar
	dao - jar

* SOAProvider
>dependency parent为SOAParent，主要作用是实现IService，规范的情况下，provider中只有serviceImpl，并把实现了的service注册到zookeeper

* SOAConsumer
>consumer的dependency parent不需要为SOAParent（正常情况下消费者是与后台无关的，除非是一个系统既是consumer，又是provider），它只要引入对应的IService就行。


### Dubbo篇


### zookeeper篇
1. zookeeper默认的三个端口
```java
//对client端提供服务，在provider和consumer中设置registry时使用
2181
//服务器与集群中的 Leader 服务器交换信息的端口
2888
//一旦集群中的 Leader 服务器挂了，需要一个端口重新进行选举，选出一个新的 Leader
3888
```
2. application.properties中spring-dubbo的配置
```java
//应用名称
spring.dubbo.application.name=dubbo_consumer
//注册中心地址
//两台服务器，三个zookeeper注册中心（算是伪集群）
spring.dubbo.registry.address=zookeeper://172.16.103.146:2281?backup=172.16.103.19:2281,172.16.103.136:2381
//协议名称
spring.dubbo.protocol.name=dubbo
//协议端口
spring.dubbo.protocol.port=20800
//
spring.dubbo.registry.timeout=6000
```

### Dubbo-Admin搭建

2. 出现的问题
```java
1. It is probably not running
	删除data下面除myid之外的其它文件
	保证dataDir、dataDirLog的路径是正确的
2. 实验证明，当provider写了版本号后，consumer必须也要写版本号，不然引不到
```
----
>后台学习推荐书籍
spring in action 第四版
springboot 揭秘  配合博客
dubbo 官网
《大型网站技术架构：核心原理与案例分析》
《大型网站系统与Java中间件实践》
大型分布式网站架构设计与实践
SOA理解
redis
redis开发与运维

