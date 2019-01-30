---
title: 一个redisson客户端实现分布式锁的异常
date: 2019-01-30 14:29:02
tags:
---

线上使用redis分布式锁偶发如下错误信息：
```java
java.lang.IllegalMonitorStateException: attempt to unlock lock, not locked by current thread by node id: 16c58397-66df-448d-aa46-c962ba4360bd thread-id: 306
	at org.redisson.RedissonLock.unlock(RedissonLock.java:367) ~[redisson-3.5.7.jar!/:na]
```
通过错误日志我们可以推断是由于lock和unlock不是同一个线程导致，那什么时候会出现lock和unlock不是同一个线程呢？
以下便是一种场景：
1. 线程1先拿到锁，但是业务执行时间超过锁自动超时时间 ，锁超时被自动释放
2. 线程2因为锁超时被释放后拿到了锁，开始执行业务
3. 线程1业务执行完成，尝试释放锁，但是这时候锁被线程2持有，于是报锁释放失败错误。( attempt to unlock lock, not locked by current thread)
4. 线程2正常释放自己持有的锁。

使用redis实现分布锁释放锁时需要锁保护，只有持有锁的线程方可释放锁。
redis文档中推荐使用长随机的字符串（如UUID）来标志持有锁的client，redisson则使用nodeId（UUID）+threadId标志。