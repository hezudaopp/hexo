---
title: RabbitMQ和Kafka作为事件分发消息中间件使用时的正确姿势
date: 2019-11-18 15:35:24
tags:
- RabbitMQ
- Kafka
- 事件消息
---
系统加入了消息中间件之后，开发人员一般会消息中间件做一些异步操作。但是如何正确使用消息队列进行业务解耦以及事件分发，多数开发人员并没有规范。以下分别罗列RabbitMQ和Kafka用作领域事件分发消息中间件的一些规范。

#### RabbitMQ
- 通过topic模式收发消息
- Exchange一般是领域名称：如ORDER_EXCHANGE
- RoutingKey一般是业务名，如ORDER.PAID
- Queue一般用是消费端业务名，如MARKETING_ACTIVITY_COUNT_INCREASE
- __Queue和Exchange,RoutingKey的绑定操作在消费端处理__
- Exchange和RoutingKey由__生产端__声明，同学需要对消费端可见

#### Kafka
- topic表示领域事件，类似RabbitMQ中的RoutingKey，topic名称一般是`领域.事件`
- 以业务消费逻辑为单元定义consumerGroup
- publish方往特定的topic下推送消息
- 一般以领域中的聚合id的hash值来做topic下patition的依据
- consumerGroup下可以定义小于等于patition数的consumer