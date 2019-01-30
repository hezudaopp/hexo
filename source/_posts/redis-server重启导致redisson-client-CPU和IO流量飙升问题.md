---
title: redis server重启导致redisson client CPU和IO流量飙升问题
date: 2019-01-16 17:49:02
tags:
- redis
- netty
---


#### 现象：

java1, java2, java4 cpu上升并且与redis的IO流量飙升，飙升之后没有下降趋势且趋于稳定

java3指标正常是因为java3的dubbo服务被禁用。

#### 分析：

- 出问题的时间点scheduled在启动（应该是巧合，第二天晚上重新尝试重启scheduled无法重现，而且从业务上说也解释不通。）

- 出问题的时间点同时出现redis connect timeout错误（需确认那个时间点是否有操作redis server dump？）

- java1,java2,java4 cpu指标差不多，但是input/output流量却相差较多，并且input/output比率也相差较大(以此推断，这三个机子执行的redis命令可能并不相同)

- docker stats发现，出问题的三台机子product-service cpu都达到200%以上

- arthas ```shell thread -n 5```，发现redisson-netty-1-*这类线程占用cpu资源比较多，而且调用堆栈信息都是：
```Java
"redisson-netty-1-13" Id=51 cpuUsage=7% RUNNABLE (in native)
    at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
    at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
    at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
    at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
    -  locked io.netty.channel.nio.SelectedSelectionKeySet@31f3c444
    -  locked java.util.Collections$UnmodifiableSet@15dc8f75
    -  locked sun.nio.ch.EPollSelectorImpl@405d0006
    at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
    at io.netty.channel.nio.SelectedSelectionKeySetSelector.select(SelectedSelectionKeySetSelector.java:62)
    at io.netty.channel.nio.NioEventLoop.select(NioEventLoop.java:753)
    at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:409)
    at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:858)
    at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:138)
    at java.lang.Thread.run(Thread.java:745)
```

- 在java2上执行packet peat抓包如图，但是java1, java4却没有抓包（主要考虑java1，java4 io较java2高）
![](https://github.com/hezudaopp/hexo/blob/master/source/_posts/_v_images/20190116175455944_668045513.png?raw=true)

- watch com.pupu.product.service.impl.ActivityEntityServiceImpl findActivityIdByStoreProduct "{params,returnObj}" -x 2并没有发现被4903这个key被频繁调用（应该是redisson-netty-1-*这类线程在底层频繁调用）

- java2: jmap -histo 信息
```Java
root@prod-java-2:/# jmap -histo 1 | head -50

 num     #instances         #bytes  class name
----------------------------------------------
   1:       6058560      663009504  [C
   2:       1116359      388823592  [I
   3:       5718247      182983904  io.netty.util.Recycler$DefaultHandle
   4:       5684865      181915680  io.netty.buffer.PoolThreadCache$MemoryRegionCache$Entry
   5:        101042      169851384  [J
   6:       2029037      156812456  [Ljava.lang.Object;
   7:       6411509      153876216  java.lang.String
   8:        181216       75385856  rx.internal.util.unsafe.SpscLinkedQueue
   9:       2157774       69048768  java.util.HashMap$Node
  10:        719111       63281768  org.apache.catalina.session.StandardSession
  11:        722043       60966008  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  12:        521270       60829712  [B
  13:       1672912       53533184  java.util.concurrent.ConcurrentHashMap$Node
  14:        629204       50322752  [S
  15:        285276       50208576  rx.internal.util.unsafe.SpscUnboundedArrayQueue
  16:        724410       47073520  [Ljava.util.Hashtable$Entry;
  17:        735279       47057856  java.util.concurrent.ConcurrentHashMap
  18:        724250       34764000  java.util.Hashtable
  19:        550168       26408064  org.hibernate.hql.internal.ast.tree.Node
  20:       1005209       24125016  java.util.ArrayList
  21:        719446       23022272  com.pupu.core.dto.web.JwtUser
  22:        719446       23022272  org.springframework.security.authentication.UsernamePasswordAuthenticationToken
  23:        148872       22774848  [Ljava.util.HashMap$Node;
  24:         56908       22307936  com.pupu.product.domain.Product
  25:       1246184       19938944  java.lang.Integer
  26:        616861       19739552  java.util.LinkedList
  27:        719446       17266704  org.springframework.security.web.authentication.WebAuthenticationDetails
  28:        719124       17258976  java.beans.PropertyChangeSupport
  29:        285294       15976464  rx.internal.operators.OperatorMerge$InnerSubscriber
  30:        635026       15240624  java.util.LinkedList$Node
  31:        235563       15076032  org.hibernate.hql.internal.ast.tree.ParameterNode
  32:        621381       14913144  java.util.concurrent.ConcurrentHashMap$KeySetView
  33:        580886       13941264  rx.internal.util.SubscriptionList
  34:        285276       13693248  rx.subjects.UnicastSubject$State
  35:        201222       12878208  java.nio.DirectByteBuffer
  36:        263905       12667440  java.util.HashMap
  37:        719124       11505984  java.beans.PropertyChangeSupport$PropertyChangeListenerMap
  38:        719113       11505808  org.springframework.security.core.context.SecurityContextImpl
  39:        719111       11505776  org.apache.catalina.session.StandardSessionFacade
  40:        445994       10703856  java.lang.Long
  41:        115756       10186528  java.lang.reflect.Method
  42:        561581        8985296  java.util.concurrent.atomic.AtomicReference
  43:        181216        8698368  rx.internal.operators.OperatorScan$3
  44:        181216        8698368  rx.internal.operators.OperatorScan$InitialProducer
  45:        181216        8698368  rx.internal.operators.OperatorSkip$1
  46:        195256        7810240  java.util.LinkedHashMap$Entry
  47:        235808        7545856  org.hibernate.engine.query.spi.NamedParameterDescriptor
```

#### 重现
压测搜索接口，然后执行redis dump操作，执行过程接口压测不要停。

#### 测试结果
压测搜索接口，执行redis sleep操作，无法重现问题；执行docker restart redis，可以重现。即使停掉压测接口，流量还是没有下降。
![](https://github.com/hezudaopp/hexo/blob/master/source/_posts/_v_images/20190116180244685_796430475.png?raw=true)

#### 确认问题
使用RBatch时候才有如上问题，具体参考[issue](https://github.com/redisson/redisson/issues/1567)