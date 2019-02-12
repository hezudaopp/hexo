---
title: 记一次外键和并发引发的MySQL死锁
date: 2019-02-12 11:04:09
tags:
---

线上订单状态流转业务偶尔会报数据库MySQL错误
```java
org.springframework.dao.CannotAcquireLockException:couldnotexecutestatement;SQL[n/a];nestedexceptionisorg.hibernate.exception.LockAcquisitionException:couldnotexecutestatement
org.springframework.orm.jpa.vendor.HibernateJpaDialect.convertHibernateAccessException(HibernateJpaDialect.java:269)~[spring-orm-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:244)~[spring-orm-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.orm.jpa.JpaTransactionManager.doCommit(JpaTransactionManager.java:521)~[spring-orm-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:761)~[spring-tx-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:730)~[spring-tx-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:504)~[spring-tx-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:292)~[spring-tx-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:96)~[spring-tx-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.dao.support.PersistenceExceptionTranslationInterceptor.invoke(PersistenceExceptionTranslationInterceptor.java:136)~[spring-tx-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.data.jpa.repository.support.CrudMethodMetadataPostProcessor$CrudMethodMetadataPopulatingMethodInterceptor.invoke(CrudMethodMetadataPostProcessor.java:133)~[spring-data-jpa-1.11.1.RELEASE.jar!/:na]
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.data.repository.core.support.SurroundingTransactionDetectorMethodInterceptor.invoke(SurroundingTransactionDetectorMethodInterceptor.java:57)~[spring-data-commons-1.13.1.RELEASE.jar!/:na]
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:213)~[spring-aop-4.3.7.RELEASE.jar!/:4.3.7.RELEASE]
...
```
以上日志省略了业务代码那部分的错误，初步排查是由于数据库锁获取失败导致报错。

1. 我们根据错误日志的时间点去反查数据库情况，发现数据库短时间内有三条一样的业务日志产生。根据数据库业务日志再去查询nginx请求日志，发现短时间（ms级）内同一个骑手请求同一个订单流转接口四次（三次成功，一次超时），如下图：
![](https://github.com/hezudaopp/hexo/blob/master/source/_posts/_v_images/20190212113347689_1865658982.png?raw=true)

2. 既然是并发情况下才会有死锁的问题，我们尝试在测试环境重现。步骤如下：
  - 将某个订单的状态修改为125
  - 压力测试同一个订单的next_step接口
  - 因为next_step会将订单状态修改为130，因此不一定重现，所有保持压力的同时订单状态通过客服调度接口调回125，查看接口返回内容。
  通过如上方法，我们在测试环境重现了死锁问题。
3. 既然能够重现，那我们在出现死锁的时候执行MySQL的`show engine innodb status;`命令，查看死锁情况，结果如下：
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2019-02-01 08:38:29 2ab2adb11700
*** (1) TRANSACTION:
TRANSACTION 380581806, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 643231, OS thread handle 0x2ab28ea03700, query id 162329967 192.168.1.12 pupumall updating
update logisticstasks set xxx = "aaa" where id = 123;
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5338 page no 16 n bits 96 index `PRIMARY` of table `fresh_main_test`.`logisticstasks` trx id 380581806 lock_mode X locks rec but not gap waiting
Record lock, heap no 26 PHYSICAL RECORD: n_fields 35; compact format; info bits 0
 0: len 30; hex 30326561616330372d636237372d346637652d623363392d336337373438; asc 02eaac07-cb77-4f7e-b3c9-3c7748; (total 36 bytes);
 1: len 6; hex 000016af3790; asc     7 ;;
 2: len 7; hex 21000001c42a5b; asc !    *[;;
 3: len 18; hex 303030313037353438393233353934393132; asc 000107548923594912;;
 4: len 30; hex 31343731643431652d633330632d346666622d393062612d393866656262; asc 1471d41e-c30c-4ffb-90ba-98febb; (total 36 bytes);
 5: len 30; hex 61383361336662312d306365632d343863382d616130342d613465643662; asc a83a3fb1-0cec-48c8-aa04-a4ed6b; (total 36 bytes);
 6: len 8; hex 000000000052a140; asc      R @;;
 7: len 4; hex 8000000a; asc     ;;
 8: len 4; hex 80000000; asc     ;;
 9: len 30; hex 30326561616330372d636237372d346637652d623363392d336337373438; asc 02eaac07-cb77-4f7e-b3c9-3c7748; (total 36 bytes);
 10: len 16; hex 30313037353438393233353934393132; asc 0107548923594912;;
 11: len 30; hex 37393161353335632d336535362d343738332d383135622d386430633133; asc 791a535c-3e56-4783-815b-8d0c13; (total 36 bytes);
 12: len 4; hex 80000082; asc     ;;
 13: len 4; hex 80000000; asc     ;;
 14: len 8; hex 80000168a325e1f3; asc    h %  ;;
 15: len 1; hex 80; asc  ;;
 16: len 30; hex 30653038373938632d366264642d343938322d613237612d653332636331; asc 0e08798c-6bdd-4982-a27a-e32cc1; (total 36 bytes);
 17: len 3; hex 78636a; asc xcj;;
 18: len 20; hex 3232323220202020202020202020202020202020; asc 2222                ;;
 19: len 4; hex 8000000a; asc     ;;
 20: len 4; hex 80000078; asc    x;;
 21: len 0; hex ; asc ;;
 22: len 30; hex 30303030303030302d303030302d303030302d303030302d303030303030; asc 00000000-0000-0000-0000-000000; (total 36 bytes);
 23: len 8; hex 80000168a30a6ab6; asc    h  j ;;
 24: len 8; hex 80000168a8359250; asc    h 5 P;;
 25: len 30; hex 30303030303030302d303030302d303030302d303030302d303030303030; asc 00000000-0000-0000-0000-000000; (total 36 bytes);
 26: len 30; hex 37313562333963302d663336382d343865392d626161302d363337363964; asc 715b39c0-f368-48e9-baa0-63769d; (total 36 bytes);
 27: len 1; hex 81; asc  ;;
 28: len 4; hex 8000000a; asc     ;;
 29: len 7; hex 4e313633333933; asc N163393;;
 30: len 8; hex 80000168a325e1f3; asc    h %  ;;
 31: len 8; hex 80000168a325e1f3; asc    h %  ;;
 32: len 4; hex 80000023; asc    #;;
 33: len 1; hex 81; asc  ;;
 34: len 4; hex 80000000; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 380581807, ACTIVE 0 sec starting index read, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 643719, OS thread handle 0x2ab2adb11700, query id 162329968 192.168.1.12 pupumall updating
update logisticstasks set update logisticstasks set xxx = "aaa" where id = 123;
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 5338 page no 16 n bits 96 index `PRIMARY` of table `fresh_main_test`.`logisticstasks` trx id 380581807 lock mode S locks rec but not gap
Record lock, heap no 26 PHYSICAL RECORD: n_fields 35; compact format; info bits 0
 0: len 30; hex 30326561616330372d636237372d346637652d623363392d336337373438; asc 02eaac07-cb77-4f7e-b3c9-3c7748; (total 36 bytes);
 1: len 6; hex 000016af3790; asc     7 ;;
 2: len 7; hex 21000001c42a5b; asc !    *[;;
 3: len 18; hex 303030313037353438393233353934393132; asc 000107548923594912;;
 4: len 30; hex 31343731643431652d633330632d346666622d393062612d393866656262; asc 1471d41e-c30c-4ffb-90ba-98febb; (total 36 bytes);
 5: len 30; hex 61383361336662312d306365632d343863382d616130342d613465643662; asc a83a3fb1-0cec-48c8-aa04-a4ed6b; (total 36 bytes);
 6: len 8; hex 000000000052a140; asc      R @;;
 7: len 4; hex 8000000a; asc     ;;
 8: len 4; hex 80000000; asc     ;;
 9: len 30; hex 30326561616330372d636237372d346637652d623363392d336337373438; asc 02eaac07-cb77-4f7e-b3c9-3c7748; (total 36 bytes);
 10: len 16; hex 30313037353438393233353934393132; asc 0107548923594912;;
 11: len 30; hex 37393161353335632d336535362d343738332d383135622d386430633133; asc 791a535c-3e56-4783-815b-8d0c13; (total 36 bytes);
 12: len 4; hex 80000082; asc     ;;
 13: len 4; hex 80000000; asc     ;;
 14: len 8; hex 80000168a325e1f3; asc    h %  ;;
 15: len 1; hex 80; asc  ;;
 16: len 30; hex 30653038373938632d366264642d343938322d613237612d653332636331; asc 0e08798c-6bdd-4982-a27a-e32cc1; (total 36 bytes);
 17: len 3; hex 78636a; asc xcj;;
 18: len 20; hex 3232323220202020202020202020202020202020; asc 2222                ;;
 19: len 4; hex 8000000a; asc     ;;
 20: len 4; hex 80000078; asc    x;;
 21: len 0; hex ; asc ;;
 22: len 30; hex 30303030303030302d303030302d303030302d303030302d303030303030; asc 00000000-0000-0000-0000-000000; (total 36 bytes);
 23: len 8; hex 80000168a30a6ab6; asc    h  j ;;
 24: len 8; hex 80000168a8359250; asc    h 5 P;;
 25: len 30; hex 30303030303030302d303030302d303030302d303030302d303030303030; asc 00000000-0000-0000-0000-000000; (total 36 bytes);
 26: len 30; hex 37313562333963302d663336382d343865392d626161302d363337363964; asc 715b39c0-f368-48e9-baa0-63769d; (total 36 bytes);
 27: len 1; hex 81; asc  ;;
 28: len 4; hex 8000000a; asc     ;;
 29: len 7; hex 4e313633333933; asc N163393;;
 30: len 8; hex 80000168a325e1f3; asc    h %  ;;
 31: len 8; hex 80000168a325e1f3; asc    h %  ;;
 32: len 4; hex 80000023; asc    #;;
 33: len 1; hex 81; asc  ;;
 34: len 4; hex 80000000; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 5338 page no 16 n bits 96 index `PRIMARY` of table `fresh_main_test`.`logisticstasks` trx id 380581807 lock_mode X locks rec but not gap waiting
Record lock, heap no 26 PHYSICAL RECORD: n_fields 35; compact format; info bits 0
 0: len 30; hex 30326561616330372d636237372d346637652d623363392d336337373438; asc 02eaac07-cb77-4f7e-b3c9-3c7748; (total 36 bytes);
 1: len 6; hex 000016af3790; asc     7 ;;
 2: len 7; hex 21000001c42a5b; asc !    *[;;
 3: len 18; hex 303030313037353438393233353934393132; asc 000107548923594912;;
 4: len 30; hex 31343731643431652d633330632d346666622d393062612d393866656262; asc 1471d41e-c30c-4ffb-90ba-98febb; (total 36 bytes);
 5: len 30; hex 61383361336662312d306365632d343863382d616130342d613465643662; asc a83a3fb1-0cec-48c8-aa04-a4ed6b; (total 36 bytes);
 6: len 8; hex 000000000052a140; asc      R @;;
 7: len 4; hex 8000000a; asc     ;;
 8: len 4; hex 80000000; asc     ;;
 9: len 30; hex 30326561616330372d636237372d346637652d623363392d336337373438; asc 02eaac07-cb77-4f7e-b3c9-3c7748; (total 36 bytes);
 10: len 16; hex 30313037353438393233353934393132; asc 0107548923594912;;
 11: len 30; hex 37393161353335632d336535362d343738332d383135622d386430633133; asc 791a535c-3e56-4783-815b-8d0c13; (total 36 bytes);
 12: len 4; hex 80000082; asc     ;;
 13: len 4; hex 80000000; asc     ;;
 14: len 8; hex 80000168a325e1f3; asc    h %  ;;
 15: len 1; hex 80; asc  ;;
 16: len 30; hex 30653038373938632d366264642d343938322d613237612d653332636331; asc 0e08798c-6bdd-4982-a27a-e32cc1; (total 36 bytes);
 17: len 3; hex 78636a; asc xcj;;
 18: len 20; hex 3232323220202020202020202020202020202020; asc 2222                ;;
 19: len 4; hex 8000000a; asc     ;;
 20: len 4; hex 80000078; asc    x;;
 21: len 0; hex ; asc ;;
 22: len 30; hex 30303030303030302d303030302d303030302d303030302d303030303030; asc 00000000-0000-0000-0000-000000; (total 36 bytes);
 23: len 8; hex 80000168a30a6ab6; asc    h  j ;;
 24: len 8; hex 80000168a8359250; asc    h 5 P;;
 25: len 30; hex 30303030303030302d303030302d303030302d303030302d303030303030; asc 00000000-0000-0000-0000-000000; (total 36 bytes);
 26: len 30; hex 37313562333963302d663336382d343865392d626161302d363337363964; asc 715b39c0-f368-48e9-baa0-63769d; (total 36 bytes);
 27: len 1; hex 81; asc  ;;
 28: len 4; hex 8000000a; asc     ;;
 29: len 7; hex 4e313633333933; asc N163393;;
 30: len 8; hex 80000168a325e1f3; asc    h %  ;;
 31: len 8; hex 80000168a325e1f3; asc    h %  ;;
 32: len 4; hex 80000023; asc    #;;
 33: len 1; hex 81; asc  ;;
 34: len 4; hex 80000000; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
```
分析如下：
  - 事务1等待来自事务2的锁.
  - 事务2占有一个锁（RECORD LOCKS space id 5338 page no 16 n bits 96 index `PRIMARY` of table `fresh_main_test`.`logisticstasks` trx id 380581807 lock mode S locks rec but not gap
Record lock, heap no 26）但是它等待（RECORD LOCKS space id 5338 page no 16 n bits 96 index `PRIMARY` of table `fresh_main_test`.`logisticstasks` trx id 380581807 lock_mode X locks rec but not gap waiting Record lock, heap no 26)
  - 事务2被回滚
4. 打印业务sql:
```sql
set session transaction read only
SET autocommit=0
select * from logisticstasks logisticst0_ where logisticst0_.id=?
commit
SET autocommit=1
select @@session.tx_read_only
set session transaction read write
set session transaction read only
SET autocommit=0
select * from orders orders0_ where orders0_.id=?
commit
SET autocommit=1
select @@session.tx_read_only
set session transaction read write
SET autocommit=0
select * from logisticsroutes logisticsr0_ where logisticsr0_.id=?
insert into logisticsroutes () values ()
update logisticstasks set xxx=? where id=?
commit
SET autocommit=1
SET autocommit=0
commit
SET autocommit=1
SET autocommit=0
commit
SET autocommit=1
```
可能是由于外键导致死锁（`logisticsroutes`表`task_id`字段的外键是` logisticstasks`的`id`字段），主要分析`insert into logisticsroutes`和`update logisticstasks`这两个语句：
  - 事务1执行`insert into logisticsroutes`, 因为外键约束，持有`logisticstasks`表`id`为123行锁（share mode）
  - 事务2执行`insert into logisticsroutes`，因为外键约束，和事务1同时持有`logisticstasks`表`id`为123行锁（share mode）
  - 事务1执行`update logisticstasks`，尝试获取`logisticstasks`的`id`为123行锁（exclusive mode），因为事务2已经持有`logisticstasks`表`id`为123行锁（share mode），因此事务1等待事务2释放锁
  - 事务2执行`update logisticstasks`，尝试获取`logisticstasks`表`id`为123的行锁（exclusive mode），但是事务1已经持有`logisticstasks`的`id`为123行锁（share mode），因此事务2等待事务1释放锁
以上，产生死锁。
5. 如果上条评论的分析正确，那么在删除logisticsroute表外键约束之后，该死锁问题就不复存在。因此我们在压测环境将此外键约束删除，然后重新用之前的方法尝试重现死锁问题。
测试结果：
没有发现之前的死锁错误日志，因此可以断定是由于外键约束导致死锁问题。