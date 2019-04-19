---
title: RabbitMQ消息队列持久化导致消费速度慢问题深究
date: 2019-04-19 20:06:04
tags:
  - RabbitMQ
  - durable
---

最近限时抢购业务导致消息队列堆积严重，这种情况一般是消息消费速度无法跟上消息生产速度产生的。那么什么会导致消息消费速度降低呢？很明显的一点，就是在消费端处理重业务导致消费端耗时久导致。但是除了这一点，通过指标还观察到一些只处理简单业务逻辑的消息队列消费端也出现了消息堆积的情况。于是顺势再推倒，除了消费端的消费逻辑，还有什么会导致消息消费速度降低呢？查阅了RabbitMQ文档之后，消息队列的持久化这个参数引起了我的注意。

#### 持久化队列VS瞬时队列
我们知道磁盘的IO速度远低于内存的操作，如果一条消息，到达RabbitMQ server端队列之后需要持久化之后才能被消费，那么消费的速度自然就会受限于消息的持久化速度，消费速度自然提不起来。如何验证消息的持久化对消费速度的影响呢？我们声明两个消息队列：orders和orders_false_durable，这两个队列处理相同的代码，并且让它们同时监听同一个Exchange规则相同的RoutingKey，然后，Publish大量的消息到这个Exchange的RoutingKey，让这些同时被这两个队列接收消费。我们监控这两个队列的消费速度如下：

![orders](https://github.com/hezudaopp/hexo/raw/master/source/_posts/_v_images/20190419204807018_623498566.png)
![orders_false_durable](https://github.com/hezudaopp/hexo/raw/master/source/_posts/_v_images/20190419204842309_1567685049.png)

从两个队列消费速度对比我们可以发现
1. 持久化队列的消息消费速度在消息生产的前期一直处于低位，直到生产快结束之后，消息消费速度才大幅提升；消息堆积成线性上升趋势。
2. 非持久化队列（瞬时队列）的消息消费速度几乎和消息生产速度持平，虽然仍有消息堆积（短时间大量Publish导致），但是消息堆积的量趋于稳定，而且是持久化队列堆积量的1/10不到。

既然持久化队列会大幅降低消息消费性能，为何不将队列设置成Transient（瞬时）呢？主要是因为，broker如果重启，入队且未被消费的消息如果未持久化，那么这些消费会随队列一起丢失。（重启期间新产生的消息也会因为队列不存在而丢失？）

#### IOPS影响持久化速度
接下来你可能会问是什么限制了消息的持久化性能？对，RabbitMQ server的IOPS是你第一个想到的点。通过Grafana监控RabbitMQ集群的指标（Grafana没有监控机器的IOPS），我们发现CPU和Memory指标没有任何异常，但是集群机器中的rabbitmq1的Load指标引起了我的注意：

![Load指标](https://github.com/hezudaopp/hexo/row/master/source/_posts/_v_images/20190419212157133_605753500.png?row=true)

每天上午的Load指标一直正常，但是到下午后一直飙到高位没有下降，直到近凌晨几乎没有业务产生消息后Load才下降。

查看aws ec2实例挂载卷的IOPS监控信息，如下图：

![IOPS](https://github.com/hezudaopp/hexo/row/master/source/_posts/_v_images/20190419215155334_1826587720.png?row=true)

此卷最大每天允许IOPS是300，从图中可以看出，凌晨和上午该卷的IOPS远大于300，到下午和晚上的时候IOPS被限制再100多，于是就会有上图中Load下午和晚上指标飙高的情况。

由此分析，IOPS速度影响了RabbitMQ的持久化速度，间接影响了队列消费速度。

#### 解决方案：
1. 垂直升级：提高实例卷的IOPS（aws中可以通过增加卷容量来提供）
2. 水平升级：不同的队列放在不同的RabbitMQ集群。