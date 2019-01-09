---
title: Executor Service内存泄漏
date: 2018-12-28 20:50:22
tags:
- Thread
- Executor Service
---

线上某台机的内存吃满，通过docker stats --no-stream查看该主机微服务的情况，发现其中一个微服务的内存竟然吃掉了7g多，但是该微服务配置的Xms和Xmx都是4g，初步断定是堆外内存泄漏，因为堆外内存JVM是不负责回收。

JVM的NIO的Channel和Buffer就用到堆外内存，这部分内存并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，如果使用不当，可能会导致OutOfMemoryError异常。

查看错误日志，发现如下信息
```Java
java.lang.OutOfMemoryError    ERROR    java.lang.OutOfMemoryError: unable to create new native thread
```
看来调用Native函数分配线程内存时出错，回想昨晚新发布的代码，跟线程有关的就是有个方法使用了局部变量引用ThreadPoolExecutor分配的线程池，而该线程池在方法结束之前没有调用ExecutorServie的shutdown方法释放线程池导致内存泄漏。

*其实线程池的用法应该是常驻内存，而不应该作为局部变量使用，否则会频繁的申请释放线程池而影响程序性能。*

*另外，由于Docker容器没有做内存限制，如果某个Docker服务出现内存泄漏，就有可能吃掉整台主机的内存。*