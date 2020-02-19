---
title: Dubbo consumer端优雅停机的不足以及改进
date: 2020-02-18 15:43:04
tags:
  - Dubbo
  - 优雅停机
---
基于Dubbo 2.6.0版本
## 问题抛出
### provider方的优雅停机
Dubbo服务提供方主要通过以下流程实现优雅停机：
1. 先注销provider服务在注册中心的注册信息，这样新的consumer流量就无法调用到该机器的provider服务
2. 等待已经在运行的provider服务线程执行完任务返回给consumer方（最大允许等待超时时间通过dubbo.service.shutdown.wait来配置，默认为10000ms）
3. 继续停机响应的操作

### consumer方的优雅停机
Dubbo官方对于consumer方实现优雅停机的描述是这样的
  - 停止时，不再发起新的调用请求，所有新的调用在客户端即报错。
  - 然后，检测有没有请求的响应还没有返回，等待响应返回，除非超时，则强制关闭。
 关键问题在于第一点，如果保证停止时不再有新的调用请求。
 
#### web服务
对于web类服务，我们通过网关（如nginx，kong等）切换流量来实现停机前不再发起新的Dubbo调用请求。

#### 消息中间件消费服务
如果是消息中间件消费服务，如果避免在停止Dubbo容器时，不再有新的Dubbo调用请求以及如何如果保证有的Dubbo调用结束后再关闭容器。

##  实现方案
如上分析，我们可以判断如果服务在停机的时候还有新的请求，那么这些请求会由于Dubbo容器的停止而调用失败。
### 验证问题存在
我们使用RabbitMQ消息中间件做模拟，先往RabbitMQ broker的queue中推送大量的消息等待消费应用来消费，消费的逻辑很简单，调用Dubbo接口，该Dubbo接口会sleep 1s后返回。
当停止消费应用Dubbo容器的时候，会报如下的错误信息：
```java
Caused by: com.alibaba.dubbo.rpc.RpcException: Rpc invoker for service interface com.pupu.product.api.sleep -> dubbo://192.168.1.111:28087/com.pupu.product.api.sleep?anyhost=true&application=product-consumer&check=false&dubbo=2.6.0&generic=false&interface=com.pupu.product.api.sleep&loadbalance=consistenthash&methods=sleep&pid=1&register.ip=172.16.16.121&remote.timestamp=1581085333479&revision=0.0.140&side=consumer&timeout=10000&timestamp=1581086035422&version=1.0.0 on consumer 192.168.1.111 use dubbo version 2.6.0 is DESTROYED, can not be invoked any more!
	at com.alibaba.dubbo.rpc.protocol.AbstractInvoker.invoke(AbstractInvoker.java:122) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.pupu.core.config.dubbo.HystrixFilter.invoke(HystrixFilter.java:22) ~[core-common-0.0.235.jar!/:0.0.235]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:68) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.monitor.support.MonitorFilter.invoke(MonitorFilter.java:74) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:68) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.protocol.dubbo.filter.FutureFilter.invoke(FutureFilter.java:53) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:68) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.filter.ConsumerContextFilter.invoke(ConsumerContextFilter.java:47) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:68) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(ListenerInvokerWrapper.java:73) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:52) ~[dubbo-2.6.0.jar!/:2.6.0]
	at com.alibaba.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:77) ~[dubbo-2.6.0.jar!/:2.6.0]
	... 8 common frames omitted
```

### 问题根源
分析错误日志，找到报错代码所在：
```java
public Result invoke(Invocation inv) throws RpcException {
        if (destroyed.get()) {
            throw new RpcException("Rpc invoker for service " + this + " on consumer " + NetUtils.getLocalHost()
                    + " use dubbo version " + Version.getVersion()
                    + " is DESTROYED, can not be invoked any more!");
        }
        ...
    }
```
很明显，问题是由于destroyed(AtomicBoolean类型)成员变量为true，导致Dubbo consumer方尝试调用provider方服务时报错。
正常情况destroyed成员变了为false，当Dubbo容易停止时，会将destroy设置为true。如下代码段说明了Dubbo容器停止时的流程：
1. AbstractConfig中注册ShutdownHook
```java
static {
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            public void run() {
                if (logger.isInfoEnabled()) {
                    logger.info("Run shutdown hook now.");
                }
                ProtocolConfig.destroyAll();
            }
        }, "DubboShutdownHook"));
    }
```
2. 当Dubbo容器停止时，调用链路：com.alibaba.dubbo.config.AbstractConfig.destroyAll() -> com.alibaba.dubbo.config.ProtocolConfig.destroyAll() -> com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol.destroy() -> com.alibaba.dubbo.rpc.protocol.AbstractProtocol.destroy() -> com.alibaba.dubbo.rpc.protocol.AbstractInvoker.destroy()。
3. com.alibaba.dubbo.rpc.protocol.AbstractInvoker.destroy()：执行如下代码设置destroyed变量为true。
```java
public void destroy() {
        if (!destroyed.compareAndSet(false, true)) {
            return;
        }
        setAvailable(false);
    }
```
理解了Dubbo停服务的实现之后，通过分析不难发现，只要在Dubbo容器执行shutdown之前，停止消息中间件中新的消息进入到容器中，并且让已经进入到容器中的消息执行完成后再执行Dubbo容器的shutdown操作即可做到消费服务的优雅停机。

我们再来看看Runtime中执行shutdownhook线程的实现：
```java
class ApplicationShutdownHooks {

    private static IdentityHashMap<Thread, Thread> hooks;
    ...
    static void runHooks() {
        Collection<Thread> threads;
        synchronized(ApplicationShutdownHooks.class) {
            threads = hooks.keySet();
            hooks = null;
        }

        // 并发执行各个hook
        for (Thread hook : threads) {
            hook.start();
        }
        for (Thread hook : threads) {
            while (true) {
                try {
                    // 等待各个hook执行完成
                    hook.join();
                    break;
                } catch (InterruptedException ignored) {
                }
            }
        }
    }
    ...
    }
```
从代码中可以看出，所有的shutdownhook线程是并发执行的，也就是说Dubbo容器的shutdownhook会和RabbitMQ consumer的shutdownhook线程并发执行。
这样就会出现RabbitMQ consumer的shutdownhook线程还没有执行完成的情况下，Dubbo容器的shutdownhook已经执行执行完毕了。
在Dubbo shutdownhook执行完成之后到RabbitMQ consumer的shutdownhook线程执行完成之后这期间的消息执行的任务都会异常。
因此，只要让RabbitMQ consumer不再接收新消息并且所有已经接收的消息已经执行完成之后再执行Dubbo shutdownhook线程，即可避免这个异常。

### 避免shutdownhook的线程并发执行
1. 注销Dubbo shutdownhook
2. 自定义实现shutdownhook，该shutdownhook实现和dubbo shutdownhook一样，不过在执行Protocol.destroyAll()方法执行前，sleep一定的时间等待其他shutdown线程执行完成。

## 实现细节
### 类加载和Spring bean加载顺序
因为需要注销Dubbo的shutdownhook，所以需要保证自定义的实现的shutdownhook处理需要在Dubbo成功添加shutdownhook之后，
而Dubbo添加shutdownhook的时候是在类加载期间，自定义的shutdownhook的处理在Spring bean（对象）加载的时候，因此可以保证注销Dubbo shutdownhook时它已经在ApplicationShutdownHooks中了。

### 反射实现shutdownhook.remove方法
ApplicationShutdownHooks的remove方法是包内才可见的，需要反射才能调用该方法。实现如下：
```java
private static void unregisterDubboShutdownHook() throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        Class clazz = Class.forName("java.lang.ApplicationShutdownHooks");
        Field field = clazz.getDeclaredField("hooks");
        field.setAccessible(true);
        IdentityHashMap<Thread, Thread> threadMap = (IdentityHashMap<Thread, Thread>) field.get(clazz);
        Thread dubboShutdownHookThread = null;
        for (Thread thread : threadMap.keySet()) {
            if ("DubboShutdownHook".equals(thread.getName())) {
                dubboShutdownHookThread = thread;
                break;
            }
        }
        if (dubboShutdownHookThread == null) {
            log.error("DubboShutdownHook thread not found");
            return;
        }
        Method method = clazz.getDeclaredMethod("remove", Thread.class);
        method.setAccessible(true);
        method.invoke(clazz, dubboShutdownHookThread);
    }
```