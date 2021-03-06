---
title: 基于Curator和Zookeeper实现分布式锁节点泄漏的问题
date: 2020-01-21 16:42:43
tags:
  - Zookeeper
  - Curator
  - 分布式锁
---
#### 发现问题
运维组巡查发现Zookeeper集群每一台实例CPU偶尔都会飙高，而且随着时间推进，CPU飙高现象也越来越频繁。具体如下图

![ZookeeperCpuRate](https://github.com/hezudaopp/hexo/raw/master/source/_posts/_v_images/20200121165616199_1170516494.png)

由于彼时Zookeeper集群耦合了太多的功能，包括Dubbo服务注册，Canal集群协调，配置中心，业务方的协调，分布式锁等等，所以难以判断是什么原因导致CPU飙高。
Zookeeper使用JAVA实现，CPU飙高大概率是由于GC导致的；再通过观察CPU飙高持续时间（如下图），再对比ZookeeperFullGC次数指标，可以断定是FullGC导致的CPU指标飙高。检查下Zookeeper Server集群的启动参数，发现只给JVM堆内存分配512M的内存；因此有理由怀疑是因为随着业务增长带来的Zookeeper节点数量增加导致更频繁的FullGC影响了CPU指标。

![ZookeeperCpu](https://github.com/hezudaopp/hexo/raw/master/source/_posts/_v_images/20200121164854006_2103878463.png)

#### 排查问题
- 第一阶段
 加大Zookeeper的JVM堆内存到4G，继续观察指标；发现内存加大后，CPU飙高现象确实没有之前那么频繁。
- 第二阶段
 大概一个月之后，CPU飙高又开始频繁了。加上Prometheus监控后我们发现，Znode缓慢增长，但是已经积累了500多万。
 通过`stat`命令，可以拿到Znode的子节点数量信息`numChildren`，发现是分布式锁相关节点下有大量的子节点，这些子节点都是没有值的。
 
![ZnodeCount](https://github.com/hezudaopp/hexo/raw/master/source/_posts/_v_images/20200121165011458_1567079786.png)

#### 解决问题
- 问题为何产生
1. Curator的InterProcessMutex实现分布式锁，会在某个固定的节点下重建`临时顺序`子节点，当锁释放的时候，这个固定的节点并不会释放。
2. 当我们需要更细粒度的锁时，一般会根据业务来控制锁的粒度。比如扣减库存的场景，我们一般会选用商品id作为锁节点名称的一部分，然后再在这个节点下创建子节点。每次锁释放之后，这个节点都会被留下。随着业务的增长，就会留下很多的没有子节点也没有值的Znode。这些Znode就用占用大量的堆内存空间，影响JVM FullGC的频率以及CPU的使用率。
- 如何解决问题
1. 升级Zookeeper和Curator版本，Zookeeper 3.5以及之后版本支持CONTAINER类型节点，主要特性为：当子节点为空时，能够自动删除父节点（A CONTAINER node is the same as a PERSISTENT node with the additional property that when its last child is deleted, it is deleted (and CONTAINER nodes recursively up the tree are deleted if empty)）。
2. 自定义简单的分布式互斥锁，拿锁的时候创建`临时节点`，释放的锁的时候删除该临时节点即可。
#### 如何删除大量如此大量的节点
1. 直接执行命令`rm`删除：Zookeeper直接报错（超时）。
2. 新建一个集群，复制非分布式锁数据到这个集群下，然后迁移到新集群：需要重启所有的应用服务，而且操作流程繁琐。
3. 这些节点ID都是UUID作为Znode名称，而这些UUID都是业务相关的ID，因此可以从数据库中取这些UUID然后一批一批删除。