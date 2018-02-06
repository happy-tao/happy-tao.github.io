---
layout: post
title: "事务与并发"
date: 2017-03-03 15:28:11 +0800
comments: true
tags:
    - Java
    - 事务
keywords: 事务，并发
---
## 事务是什么
<https://www.wikiwand.com/zh-hans/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1>

Wiki的介绍很清晰，这里就不再粘贴复制了。
<!--more-->

## 事务的隔离级别
MySQL的事务隔离级别基本和Oracle一致，此处以MySQL为例讲解。

参考文档：

<https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html>

<http://xstarcd.github.io/wiki/MySQL/mysql_isolation_level.html>

MySQL有四种隔离级别

* Read Uncommitted（读未提交）

在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。

* Read Committed（读已提交）

这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这种隔离级别 也支持所谓的不可重复读（Nonrepeatable Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

* Repeatable Read（可重读）

这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。不过理论上，这会导致另一个棘手的问题：幻读 （Phantom Read）。简单的说，幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB和Falcon存储引擎通过多版本并发控制（MVCC，Multiversion Concurrency Control）机制解决了该问题。

* Serializable（可串行化）

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。

## 隔离级别与一致性
这四种隔离级别采取不同的锁类型来实现，若读取的是同一个数据的话，就容易发生问题。例如：

脏读(Drity Read)：某个事务已更新一份数据，另一个事务在此时读取了同一份数据，由于某些原因，前一个RollBack了操作，则后一个事务所读取的数据就会是不正确的。

不可重复读(Non-repeatable read):在一个事务的两次查询之中数据不一致，这可能是两次查询过程中间插入了一个事务更新的原有的数据。

幻读(Phantom Read):在一个事务的两次查询中数据笔数不一致，例如有一个事务查询了几列(Row)数据，而另一个事务却在此时插入了新的几列数据，先前的事务在接下来的查询中，就会发现有几列数据是它先前所没有的。

在MySQL中，实现了这四种隔离级别，分别有可能产生问题如下所示：

|隔离级别|脏读|不可重复读|幻读|
|:---|:--:|:--:|:--:|
|读未提交(Read Uncommitted)|YES|YES|YES|
|读已提交(Read Committed)|NO|YES|YES|
|可重复读(Repeatable Read)|NO|NO|YES|
|可串行化(Serializable)|NO|NO|NO|

示例见SQL：

``` sql
-- 事务与并发
-- 查看MySQL事务隔离级别，分为全局和当前回话
-- 取消autocommit
set autocommit=0;
show variables like "%autocommit%";

-- 查看隔离级别
SELECT @@global.tx_isolation;
SELECT @@session.tx_isolation;
SELECT @@tx_isolation;

show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+

show global variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+

-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL read uncommitted;
SET SESSION TRANSACTION ISOLATION LEVEL read committed;
SET SESSION TRANSACTION ISOLATION LEVEL repeatable read;
SET SESSION TRANSACTION ISOLATION LEVEL serializable;

-- 事务操作

start transaction;
rollback;
commit;
-- 测试表语句
create database test;
use test;
DROP TABLE IF EXISTS `test`;
CREATE TABLE `test` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `type` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
INSERT INTO `test` VALUES (1, '吴瑞涛', '1');
INSERT INTO `test` VALUES (2, '李亚宁', '2');
INSERT INTO `test` VALUES (3, '邓景波', '1');
INSERT INTO `test` VALUES (4, '孙振磊', '2');
-- 脏读示例
-- 两个窗口执行以下内容
use test;
set autocommit=0;
SET SESSION TRANSACTION ISOLATION LEVEL read uncommitted;
set names gbk;
start transaction;
-- 窗口1
select * from test where id = 1 \G;
--窗口2
select * from test where id = 1 \G;
-- 窗口1
update test set type = 2 where id = 1;
-- 窗口2
select * from test where id = 1 \G;
--窗口1,2
commit;

-- 不可重复读示例
use test;
set autocommit=0;
SET SESSION TRANSACTION ISOLATION LEVEL read committed;
set names gbk;
start transaction;
-- 窗口1
select * from test where id = 1 \G;
--窗口2
select * from test where id = 1 \G;
-- 窗口1
update test set type = 2 where id = 1;
-- 窗口2
select * from test where id = 1 \G;
--窗口1
commit;
--窗口2
select * from test where id = 1 \G;
-- 窗口2
commit;

-- 幻读示例
use test;
set autocommit=0;
SET SESSION TRANSACTION ISOLATION LEVEL repeatable read;
set names gbk;
start transaction;
-- 窗口1
select * from test where type = 2 \G;
--窗口2
select * from test where type = 2 \G;
-- 窗口1
update test set type = 2 where id = 3;
commit;
-- 窗口2
select * from test where type = 2 \G;
--窗口2
commit;
```

## Spring事务

参考文档：

<https://www.ibm.com/developerworks/cn/education/opensource/os-cn-spring-trans/>

<http://blog.csdn.net/hjm4702192/article/details/17277669>

Spring事务通过AOP实现，允许指定隔离级别，传播属性，只读属性等。

Spring事务传播行为：

所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。在TransactionDefinition定义中包括了如下几个表示传播行为的常量：

* TransactionDefinition.PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
* TransactionDefinition.PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
* TransactionDefinition.PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
* TransactionDefinition.PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
* TransactionDefinition.PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。
* TransactionDefinition.PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
* TransactionDefinition.PROPAGATION_NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

这里需要指出的是，前面的六种事务传播行为是 Spring 从 EJB 中引入的，他们共享相同的概念。而 PROPAGATION_NESTED是 Spring 所特有的。以 PROPAGATION_NESTED 启动的事务内嵌于外部事务中（如果存在外部事务的话），此时，内嵌事务并不是一个独立的事务，它依赖于外部事务的存在，只有通过外部的事务提交，才能引起内部事务的提交，嵌套的子事务不能单独提交。如果熟悉 JDBC 中的保存点（SavePoint）的概念，那嵌套事务就很容易理解了，其实嵌套的子事务就是保存点的一个应用，一个事务中可以包括多个保存点，每一个嵌套子事务。另外，外部事务的回滚也会导致嵌套子事务的回滚。

事务的只读属性：

事务的只读属性是指，对事务性资源进行只读操作或者是读写操作。所谓事务性资源就是指那些被事务管理的资源，比如数据源、 JMS 资源，以及自定义的事务性资源等等。如果确定只对事务性资源进行只读操作，那么我们可以将事务标志为只读的，以提高事务处理的性能。在 TransactionDefinition 中以 boolean 类型来表示该事务是否只读。

参考配置：

``` xml
<!--数据源配置 -->
<bean id="base.druid" abstract="true">
    <property name="initialSize" value="${jdbc.initialSize}" />
    <property name="maxIdle" value="${jdbc.maxIdle}" />
    <property name="minIdle" value="${jdbc.minIdle}" />
    <property name="maxActive" value="${jdbc.maxActive}" />
    <property name="removeAbandoned" value="${jdbc.removeAbandoned}" />
    <property name="removeAbandonedTimeout" value="${jdbc.removeAbandonedTimeout}" />
    <property name="maxWait" value="${jdbc.maxWait}" />
    <property name="validationQuery" value="${jdbc.validationQuery}" />
    <property name="testOnBorrow" value="${jdbc.testOnBorrow}" />
</bean>
<!-- 主库配置 -->
<bean id="base.datasource.M" parent="base.druid" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    <property name="filters" value="slf4j"/>
    <property name="proxyFilters">
        <list>
            <ref bean="zmc.cat.filter" />
        </list>
    </property>
    <property name="driverClassName" value="${jdbc.oracle.driverClassName}" />
    <property name="url" value="${jdbc.oracle.base.url.master}" />
    <property name="username" value="${jdbc.oracle.base.username.master}" />
    <property name="password" value="${jdbc.oracle.base.password.master}" />
</bean>
<!-- 从库配置 -->
<bean id="base.datasource.S" parent="base.druid" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    <property name="filters" value="slf4j"/>
    <property name="proxyFilters">
        <list>
            <ref bean="zmc.cat.filter" />
        </list>
    </property>
    <property name="driverClassName" value="${jdbc.oracle.driverClassName}" />
    <property name="url" value="${jdbc.oracle.base.url.slave}" />
    <property name="username" value="${jdbc.oracle.base.username.slave}" />
    <property name="password" value="${jdbc.oracle.base.password.slave}" />
</bean>


<bean id="base.sqlSessionFactory.M" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="base.datasource.M"/>
    <property name="mapperLocations" value="classpath*:com/ziroom/zmc/base/dao/map/*.xml"/>
    <property name="configLocation" value="classpath:mybatis-config.xml"></property>
</bean>

 <bean id="base.sqlSessionFactory.S" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="base.datasource.S"/>
    <property name="mapperLocations" value="classpath*:com/ziroom/zmc/base/dao/map/*.xml"/>
    <property name="configLocation" value="classpath:mybatis-config.xml"></property>
</bean>


 <bean id="base.writeSqlSessionTemplate.M" class="org.mybatis.spring.SqlSessionTemplate" scope="prototype">
    <constructor-arg index="0" ref="base.sqlSessionFactory.M"/>
</bean>

 <bean id="base.writeSqlSessionTemplate.S" class="org.mybatis.spring.SqlSessionTemplate" scope="prototype">
    <constructor-arg index="0" ref="base.sqlSessionFactory.S"/>
</bean>

<!-- Mybatis主从支持 -->
<bean id="base.MybatisDaoContext" class="com.asura.framework.dao.mybatis.base.MybatisDaoContext">
    <property name="writeSqlSessionTemplate" ref="base.writeSqlSessionTemplate.M"></property>
    <property name="readSqlSessionTemplate" ref="base.writeSqlSessionTemplate.S"></property>
</bean>

<!-- 事务管理器 -->
<bean id="base.transactionManager"
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="base.datasource.M" />
</bean>


 <tx:advice id="base.txAdvice" transaction-manager="base.transactionManager">
    <tx:attributes>
        <tx:method name="save*" propagation="REQUIRED" />
        <tx:method name="add*" propagation="REQUIRED" />
        <tx:method name="create*" propagation="REQUIRED" />
        <tx:method name="insert*" propagation="REQUIRED" />
        <tx:method name="update*" propagation="REQUIRED" />
        <tx:method name="merge*" propagation="REQUIRED" />
        <tx:method name="del*" propagation="REQUIRED" />
        <tx:method name="remove*" propagation="REQUIRED" />
        <tx:method name="get*" read-only="true" />
        <tx:method name="find*" read-only="true" />
        <tx:method name="*" propagation="REQUIRED" />
    </tx:attributes>
</tx:advice>

<aop:config expose-proxy="true">
    <!-- 只对业务逻辑层实施事务 -->
    <aop:pointcut id="txPointcut" expression="execution(* com..service..*.*(..))" />
    <aop:advisor advice-ref="base.txAdvice" pointcut-ref="txPointcut" />
</aop:config>
```

## 使用事务的注意事项

事务的目的是：

1. 为数据库操作序列提供了一个从失败中恢复到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。
2. 当多个应用程序在并发访问数据库时，可以在这些应用程序之间提供一个隔离方法，以防止彼此的操作互相干扰。

所以我们定义事务注意事项如下：

* 需要考虑业务的最小粒度，一些附加信息是否要放在事物内，例如循环插入100个员工数据，每个员工的成功保存是应该考虑的，而不是我们需要保证100个同时成功；
* 严禁事务中夹杂HTTP调用，发邮件等拖慢事务的操作；事务是数据库概念，数据库只需考虑增删改查的成功即可，其他业务放在外层做；

## 并发

并发会带来什么问题：

1. 并发量大的情况下，会拖慢网络速度，例如年会的Gogoda活动；
2. 并发量大的情况下，用户的数据可能会乱掉，例如多个用户同时兑换一张优惠券，两个亲戚同时在柜台给一张银行卡的卡主存钱。

在此处，我们只讨论第二种情况带来的数据一致性问题。

如何解决并发的数据安全问题：

### 数据库锁

Oracle和MySQL（InnoDB引擎下）的数据库都支持数据库的行级锁，行级锁可以在尽量保证并发的情况下保证数据安全性。

参考链接：

<http://chenzhou123520.iteye.com/blog/1860954>

<http://solitary.iteye.com/blog/1396667>

MySQL行级锁示例：

``` sql
-- 行级锁示例
use test;
set autocommit=0;
SET SESSION TRANSACTION ISOLATION LEVEL read committed;
set names gbk;
start transaction;
-- 窗口1
select * from test where id = 1 \G;
-- 窗口2
select * from test where id = 1 \G;
-- 窗口1
select * from test where id = 1 for update;
-- 窗口2
select * from test where id = 1 \G;
select * from test where id = 2 for update;
select * from test where id = 1 for update;
-- 窗口1
update test set type = 2 where id = 1;
commit;
-- 观察窗口2，已获取数据信息
commit;
```

使用数据库行级锁的注意事项：

* 查询列的条件必须包含索引或主键，建议使用主键作为锁条件；
* 如果业务数据中有判断，请放在加锁之后做，放在加锁之前是没有意义的，例如银行卡扣款；
* 使用时尽量确保查询结果有一条或没有数据，多行锁可能导致死锁。
* 注意主从，确保从主库查询需要做判断的数据。

### 分布式锁

如果保证数据一致性有多条，那么此处可以使用分布式锁来解决，例如保洁阿姨的日程。

使用Redis实现分布式锁示例代码：

``` java
 /*
 * Copyright (c) 2015. ziroom.
 */
package com.asura.framework.cache.redisOne;

import com.asura.framework.utils.LogUtil;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

/**
 * <p>分布式锁工具</p>
 * <p>
 * <PRE>
 * <BR>    修改记录
 * <BR>-----------------------------------------------
 * <BR>    修改日期         修改人          修改内容
 * </PRE>
 *
 * @author sunzhenlei
 * @version 1.0
 * @date 2015/9/7 10:42
 * @since 1.0
 */
public class RedisSentinelLockHandler {

    private final static Logger LOGGER = LoggerFactory.getLogger(RedisSentinelLockHandler.class);

    /**
     * 获取锁重试次数
     */
    private static final int WAIT_COUNT = 50;

    /**
     * 获取锁重试间隔，毫秒
     */
    private static final int RETRY_INTERVAL = 50;

    /**
     * 锁前缀
     */
    private static final String LOCK_KEY_PREFIX = "lock_key_";

    RedisOperations redisOperations;

    /**
     * 获取分布式锁
     *
     * @param lockKey
     * @return
     */
    public boolean getLock(String lockKey) {
        return getLock(lockKey, WAIT_COUNT * RETRY_INTERVAL);
    }

    /**
     * 获取分布式锁
     *
     * @param lockKey 锁key
     * @param timeout 超时时间，毫秒
     * @return
     */
    public boolean getLock(String lockKey, int timeout) {
        String key = LOCK_KEY_PREFIX + lockKey;
        try {
            for (int i = 0; i < timeout / RETRY_INTERVAL; i++) {
                if (redisOperations.setnx(key, String.valueOf(1), timeout)) {
                    LogUtil.debug(LOGGER, "get lock success : key is {}", key);
                    return true;
                } else {
                    LogUtil.debug(LOGGER, "get lock failure : key is {}, count is {}", key, i);
                    Thread.sleep(RETRY_INTERVAL);
                }
            }
        } catch (Exception e) {
            LogUtil.debug(LOGGER, "get lock throw an exception key is {}, exception : {}", key, e);
        }
        LogUtil.debug(LOGGER, "get lock failure : key is {}", key);
        return false;
    }

    /**
     * 释放分布式锁
     *
     * @param lockKey
     */
    public void releaseLock(String lockKey) {
        String key = LOCK_KEY_PREFIX + lockKey;
        redisOperations.del(key);
        LogUtil.debug(LOGGER, "delete lock success : key is {}", key);
    }

    public RedisOperations getRedisOperations() {
        return redisOperations;
    }

    public void setRedisOperations(RedisOperations redisOperations) {
        this.redisOperations = redisOperations;
    }
}
```

使用分布式锁注意事项

* 判断数据的条件放入锁之内，防止数据异常（通数据库锁第一条，可做两重判断，锁外，锁内）；
* 锁应该确保数据提交完毕后释放，即分布式锁应该在事务之外；
* 确保锁条件单一性，不能同时锁住多个数据片信息，例如同时锁定保洁阿姨A和B和C的日程，会造成死锁。
* 锁一定要有超时时间，否则会导致死锁。

### 其他未在代码中实现的方案：

队列：确保数据的更新顺序。

待补充。