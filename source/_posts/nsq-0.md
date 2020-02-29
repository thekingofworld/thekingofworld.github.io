---
title: 消息队列NSQ源码分析（一）：概述
categories:
    - 消息队列
tags:
    - nsq
---
本篇为NSQ源码分析的第一篇，主要讲述NSQ的一些基础概念以及整体架构、各组件功能

<!-- more -->

## 基础概念

### topic、channel

topic和channel是nsq中比较重要的两个概念，topic代表了某种类型的数据流，一个topic通常有1个或多个channel，每个channel都会收到发往这个topic的消息拷贝。同时，一个channel可以注册多个消费者，channel的消息会随机发送给其中一个消费者。如下图：
![](/img/nsq.gif)

### nsq的消息投递模型和语义

nsq采用的消息投递模型是***push***的方式，即消息会主动推送给消费者，同时通过超时重传和确认机制实现了***最少一次***的消息投递语义

## 整体架构
![](/img/nsq-architecture.png)

### 各组件功能

#### nsqd

nsq消息队列实例，用来实现消息队列的核心功能，即消息的接收、发送、存储、topic、channel等关系的维护

#### nsqlookupd

用来实现消费者动态发现nsqd实例，消费者会定时轮询这个服务来获取当前生产指定topic的nsqd实例地址，同时nsqd实例也会定时上报实例信息到此服务。通过这种方式实现了消费者与生产者的解耦，消费者只需配置nsqlookupd实例的地址即可动态的发现指定topic的nsqd实例

#### nsqadmin

nsq的web管理后台，用来查看当前nsq集群的统计信息并执行一些管理任务

----

除了上面3个组件外，nsq官方还提供了一些命令行工具
   + nsq_stat：将当前nsq的统计信息输出到标准输出
   + nsq_tail：将指定topic/channel中的消息消费掉并输出到标准输出
   + nsq_to_file：将指定topic/channel中的消息消费掉并输出到文件
   + nsq_to_http：将指定topic/channel中的消息消费掉并向指定的地址发送http请求
   + nsq_to_nsq：将指定topic/channel中的消息消费掉并重新发布到指定的nsq实例
   + to_nsq：从标准输入获取数据并发送到nsq实例中