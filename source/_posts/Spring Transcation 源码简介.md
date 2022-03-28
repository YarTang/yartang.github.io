---
title: Spring Transaction 源码简介
layout: post
date: 2022-03-29 00:20:39
tags:
- Spring
- 事务
categoties:
- Spring
---


# 0. 引言

事务管理对系统应用来说必不可少，我们以此来保证数据的完整性和安全性，这点在金融系统中显得尤为重要。笔者在实践中仅仅了解 `@Transactional` 注解的大概使用方法，对 Spring 事务的实现原理也是一知半解，因此本文将对 Spring Transaction 进行简要分析，核心部分有以下两个方面：

- 源码的简要分析

- 声明式和编程式事务的分析

# 1. 事务回顾 && Spring Transaction 简介

事务（Transaction）是数据库区别于文件系统的重要特征之一，目前国际认可的数据库设计原则是 ACID 特性，用以保证数据库事务的正确执行。Mysql 的 innodb 引擎中的事务就完全符合 ACID 特性。

近年来分布式系统和微服务架构的盛行，催生了分布式事务，以保证大规模分布式环境下事务的最终一致性。本文主要通过单机事务对 Spring 事务原理进行分析，不涉及分布式事务原理。

Spring 对于事务的支持，分层概览图如 Fig. 1-1：

![Spring Transaction Layer](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647703785356spring-transaction-layer.png)

Fig. 1-1 Spring Transaction Layer

## 1.1 事务回顾

### 1.1.1 事务的 ACID 特性

- 原子性（Atomicity）
- 一致性（Consistency）
- 隔离性（Isolation）
- 持久性（Durability）

### 1.1.2 事务的隔离级别

#### 数据库读写的三个问题

- 脏读（Drity Read）：事务A更新记录但未提交，事务B查询出A未提交记录。

- 不可重复读（Non-repeatable read）：事务A读取一次，此时事务B对数据进行了更新或删除操作，事务A再次查询数据不一致。

- 幻读（Phantom Read）：事务A读取一次，此时事务B插入一条数据事务A再次查询，记录多了。

#### mysql 底层支持

在 MVCC 中存在两种读操作，快照读（Snapshot Read）和当前读（Current Read）

##### 快照读（一致性非锁定读）

一致性非锁定读(consistent nonlocking read)是指InnoDB存储引擎通过多版本控制(multi versionning)的方式来读取当前执行时间数据库中行的数据，如果读取的行正在执行DELETE或UPDATE操作，这是读取操作不会因此等待行上锁的释放。相反的，InnoDB会去读取行的一个快照数据。这种方式实际上就是普通的 select。

##### 当前读（锁定读）

- select * from table where ? lock in share mode; # 对读取的行加共享锁（S锁），其他事务可以加排他锁（X锁）
- select * from table where ? for update; # 对读取的行加 X 锁，其他事务不能再加任何锁
- insert, update, delete 操作前会先进行一次当前读，加 X 锁

其中前两种锁定读，需要用户自己显式使用，最后一种是自动添加的。需要注意的是这两种锁都必须处于事务中，事务 commit，锁释放。所以必须 begin 或者 start transaction 开启一个事务或者索性 `set autocommit=0` 把自动提交关掉（mysql默认是1，即执行完sql立即提交）

#### 分级处理策略

InnoDB使用不同的锁定策略支持每个事务隔离级别。对于关键数据的操作（遵从 ACID 原则），您可以使用强一致性（默认 Repeatable Read）。对于不是那么重要的数据操作，可以使用 Read Committed/Read Uncommitted 。Serializable 执行比可重读更严格的规则，用于特殊场景：XA 事务，并发性和死锁问题的故障排除。

**四种隔离级别**

- Read Uncommitted（读取未提交内容）

- Read Committed（读取提交内容）

- Repeatable Read（可重读）
 
- Serializable（可串行化）：这是最高的隔离级别，它是在每个读的数据行上加上共享锁（LOCK IN SHARE MODE）。在这个级别，可能导致大量的超时现象和锁竞争，主要用于分布式事务。

## 1.2 Spring Transaction 简介

### 1.2.1 Spring Transaction 属性

Spring Transaction 为了保证事务的 ACID 属性，定义了 6 个属性，对应 @Transaction(key1 = *, key2 = * ...) 注解中 6 个属性。

- 事务名称：用户可手动指定事务的名称，当多个事务的时候，可区分使用哪个事务。对应注解中的属性 value、transactionManager
- 隔离级别: 为了解决数据库容易出现的问题，分级加锁处理策略。对应注解中的属性 isolation
- 超时时间: 定义一个事务执行过程多久算超时，以便超时后回滚。可以防止长期运行的事务占用资源。对应注解中的属性 timeout
- 是否只读：表示这个事务只读取数据但不更新数据, 这样可以帮助数据库引擎优化事务。对应注解中的属性 readOnly
- 传播机制: 对事务的传播特性进行定义，共有7种类型。对应注解中的属性 propagation
- 回滚机制：定义遇到异常时回滚策略。对应注解中的属性 rollbackFor、noRollbackFor、rollbackForClassName、noRollbackForClassName

其中隔离级别即是上文理论的实现，下文将详细分析 Spring Transaction 的传播机制。

#### 传播机制

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。org.springframework.transaction 包下有一个事务定义接口 TransactionDefinition，定义了 7 种事务传播机制，官方文档翻译如下：

- PROPAGATION_REQUIRED
支持当前事务；如果不存在，创建一个新的。这通常是事务定义的默认设置，通常定义事务同步作用域。

- PROPAGATION_SUPPORTS
支持当前事务；如果不存在事务，则以非事务方式执行。

- PROPAGATION_MANDATORY
支持当前事务；如果当前事务不存在，抛出异常。

- PROPAGATION_REQUIRES_NEW
创建一个新事务，如果存在当前事务，则挂起当前事务。

- PROPAGATION_NOT_SUPPORTED
不支持当前事务，存在事务挂起当前事务;始终以非事务方式执行。类似于同名的EJB事务属性。

- PROPAGATION_NEVER
不支持当前事务；如果当前事务存在，抛出异常。

- PROPAGATION_NESTED
如果当前事务存在，则在嵌套事务中执行，如果当前没有事务，类似PROPAGATION_REQUIRED（创建一个新的）。

## 1.2.2 两种实现方式

在Spring中，事务有两种实现方式

- 声明式事务管理：添加 @Transactional 注解，并定义传播机制和回滚策略。基于Spring AOP 实现，本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。声明式有助于用户将操作与事务操作规则进行解耦，既起到事务管理的作用，又不影响业务代码的具体实现。

- 编程式事务管理：编程式事务管理使用底层源码可实现更细粒度的事务控制，允许用户在代码中精确定义事务的边界。Spring推荐使用 TransactionTemplate，典型的模板模式。

下面将分别使用两种方式，实现对账户金额的更新。cash_table 结构如下：

|     | id  | name     | cash   |
| --- | --- | -------- | ------ |
| 1   | 1   | mayun    | 2000   |
| 2   | 2   | mahuteng | 10000  |
| 3   | 3   | jianling | 111111 |
| 4   | 4   | huazi    | 10000  |

### 声明式事务管理

只需要在 service.impl 层，业务方法上添加 @Transactional 注解，定义事务的传播机制为 REQUIRED（不写这个参数，默认就是 REQUIRED ），遇到 Exception 异常就一起回滚。

```java
@Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
public Response annotationTransaction(Integer id, Integer cash) {
    Integer oldCash = cashDAO.selectById(id).getCash();
    int updateRes = cashDAO.updateById(id, oldCash + cash);
    // 模拟业务逻辑的错误
    if (oldCash + cash < 0) {
        throw new RuntimeException("cash is not enough! please check again.");
    }

    return updateRes == 1 ? Response.SUCCESS : Response.FAIL;
}
```

### 编程式事务管理

编程式事务管理，使用 Spring 推荐的 transactionTemplate。

```java
@Resource
TransactionTemplate transactionTemplate;

public Response programTransaction(Integer id, Integer cash) {
    transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    transactionTemplate.execute(status -> {
        try {
            Integer oldCash = cashDAO.selectById(id).getCash();

            int updateRes = cashDAO.updateById(id, oldCash + cash);
            // 模拟业务逻辑的错误
            if (oldCash + cash < 0) {
                throw new RuntimeException("cash is not enough! please check again.");
            }

            return updateRes == 1 ? Response.SUCCESS : Response.FAIL;
        } catch (Exception e) {
            status.setRollbackOnly();
            return Response.FAIL;
        }
    });

    return Response.FAIL;
}
```

注意：

- 可以不用try catch，transactionTemplate.execute自己会捕捉异常并回滚。

- 如果有业务异常需要特殊处理，记得：status.setRollbackOnly(); 标识为回滚。

### 运行结果

传入参数 id = 2， cash = -10000000，运行结果如下：
> **命令行输出**：
programResponse: FAIL
> **mysql 日志**：
Query    SELECT @@tx_isolation
Query    set autocommit=0
Query    select id, name, cash from table_cash where id = 2
Query    update table_cash set cash = -9990000 where id = 2
Query    ROLLBACK
Query    set autocommit=1
> **cash_table 结构**:
1,mayun,2000

可以从 mysql 的日志文件中发现，事务进行了 rollback，表中的数据修改不成功。

# 2. Spring Transaction 源码分析

## 2.1 初步理解

### 2.1.1 核心接口

Spring Transaction Manager 的实现有许多细节，如果对整个接口框架有个大体了解会非常有利于我们理解事务，下面通过讲解 Spring 的事务接口来了解Spring实现事务的具体策略。

![spring transaction interface](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647703871919spring-transaction-interface.png)

Fig 2.1.1 Spring Transaction 管理核心接口示意图

### 2.1.2 事务管理器 - PlatformTransactionManager

Spring 并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给Hibernate 或者 JTA 等持久化机制所提供的相关平台框架的事务来实现。

Spring 事务管理器的接口是 org.springframework.transaction.PlatformTransactionManager，通过这个接口，Spring 为各个平台如 JDBC、Hibernate 等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。此接口的内容如下：

```java
public interface PlatformTransactionManager extends TransactionManager {
    // 由 TransactionDefinition 获得 TransactionStatus
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
    // 提交
    void commit(TransactionStatus status) throws TransactionException;
    // 回滚
    void rollback(TransactionStatus status) throws TransactionException;
}
```

其类图如Fig 2.1.2：

![platform-transaction-manager-diagram](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647704019843platform-transaction-manager-diagram.png)

Fig 2.1.2 PlatformTransactionManager 类图 

具体的具体的事务管理机制对Spring来说是透明的，它并不关心那些，那些是对应各个平台需要关心的，所以Spring事务管理的一个优点就是为不同的事务API提供一致的编程模型，如JTA、JDBC、Hibernate、JPA。

- JDBC 事务：使用 org.springframework.jdbc.datasource.DatasourceTransactionManager 处理事务边界。实际上 DatasourceTransactionManager 是使用 java.sql.Connection 管理事务，在 Connection#commit() 和 Connection#rollback() 上做了一层封装，进行事务的提交与回滚。

- Hibernate 事务：HibernateTransactionManager的实现细节是它将事务管理的职责委托给org.hibernate.Transaction当事务成功完成时，调用Transaction#commit()，反之，将会调用rollback()方法。

- JPA 事务：JpaTransactionManager只需要装配一个JPA实体管理工厂（javax.persistence.EntityManagerFactory接口的任意实现）。JpaTransactionManager将与由工厂所产生的JPA EntityManager合作来构建事务。

- JTA 事务：如果你没有使用以上所述的事务管理，或者是跨越了多个事务管理源（比如两个或者是多个不同的数据源，分布式事务等），你就需要使用JtaTransactionManager。JtaTransactionManager将事务管理的责任委托给javax.transaction.UserTransaction和javax.transaction.TransactionManager，其中事务成功完成通过UserTransaction#commit()方法提交，事务失败通过UserTransaction#rollback()方法回滚。

以上几种事务管理器（包括 AbstractPlatformTransactionManager）的实现比较复杂，本文将在 2.4 节中详细介绍。

### 2.1.3 基本事务属性定义 - TransactionDefinition

PlatformTransactionManager#getTransaction(TransactionDefinition definition)方法得到事务，这个方法里面的参数是 TransactionDefinition 类，这个类就定义了 1.2.1 节中的事务属性。

```java
public interface TransactionDefinition {

    default int getPropagationBehavior()；

    default int getIsolationLevel()；

    default int getTimeout()；

    default boolean isReadOnly()；

    default String getName()；
}
```

### 2.1.4 事务状态 - TransactionStatus

PlatformTransactionManager#getTransaction()的方法得到的是TransactionStatus接口的一个实现，这个接口的内容如下：

```java
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
    // 是否有保存点
    boolean hasSavepoint();
}

public interface TransactionExecution {
    boolean isNewTransaction();
    void setRollbackOnly();
    boolean isRollbackOnly();
    boolean isCompleted();
}

public interface SavepointManager {
    Object createSavepoint() throws TransactionException;
    void rollbackToSavepoint(Object savepoint) throws TransactionException;
    void releaseSavepoint(Object savepoint) throws TransactionException;
}
```

TransactionStatus 描述的是一些处理事务提供简单的控制事务执行和查询事务状态的方法，在回滚或提交的时候需要应用对应的事务状态。

## 2.2 声明式事务源码分析

声明式事务整体调用过程，可以抽象为以下两条线：

- 使用代理模式，生成代理增强类。
这是 SpringBoot 自动配置的常规套路，此处不再赘述。

- 根据代理事务管理配置类，配置事务的织入，在业务方法前后进行环绕增强，增加一些事务的相关操作。例如获取事务属性、提交事务、回滚事务。主要介绍这条线上的实现过程。


### 2.2.1 代理事务的配置 - ProxyTransactionManagementConfiguration

```java
// org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration.java

@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    // 定义事务增强器
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(TransactionAttributeSource transactionAttributeSource, TransactionInterceptor transactionInterceptor) {
        // 定义 transactionAdvisor
        BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
        // 设置事务属性
		advisor.setTransactionAttributeSource(transactionAttributeSource);
        // 设置事务拦截器
		advisor.setAdvice(transactionInterceptor);
        // 设置切面的优先顺序
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
    }

    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    // 定义基于注解的事务属性资源
    public TransactionAttributeSource transactionAttributeSource();

    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    // 定义事务拦截器
    public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource);

}
```
transactionAdvisor() 定义了一个 advisor，设置事务属性、设置事务拦截器 TransactionInterceptor、设置切入的顺序等。而声明式事务的核心就是事务拦截器。


### 2.2.1 事务拦截器——事务的入口

TransactionInterceptor 是 Spring 实现声明式事务的拦截器，它实现了 AOP 联盟的 MethodInterceptor 接口，它的父类 TransactionAspectSupport 封装了一些用于实现事务切面对事务进行管理的基本代码。TransactionInterceptor 的继承关系如 Fig 2.2.1。

![transaction interceptor](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647704074844transaction-interceptor.png)

Fig 2.2.1 TransactionInterceptor 类图 

我们给一个bean的方法加上 @Transactional 注解后，Spring 容器给我们的是一个代理的bean。当我们对事务方法调用时，会进入 Spring 的 ReflectiveMethodInvocation#proceed 方法。这是AOP的主要实现，在进入业务方法前会调用各种方法拦截器，我们需要关注的拦截器是 org.springframework.transaction.interceptor.TransactionInterceptor。
TransactionInterceptor 的职责类似于一个“环绕切面”，在业务方法调用前根据情况开启事务，在业务方法调用完回到拦截器后进行善后清理。

事务切面在源码中具体的实现方法是 TransactionAspectSupport#invokeWithinTransaction。事务切面关注的是TransactionInfo(TxInfo)，TxInfo是一个“非常大局观”的东西（里面啥都有：TxMgr, TxAttr, TxStatus还有前一次进入事务切面的TransactionInfo)。

事务切面会调用 createTransactionIfNecessary 方法来创建事务并拿到一个 TxInfo（无论是否真的物理创建了一个事务）。如果事务块内的代码发生了异常，则会根据 TxInfo 里面的 TxAttr 配置的 rollback 规则看看这个异常是不是需要回滚，不需要回滚就尝试提交，否则就尝试回滚。如果未发生异常，则尝试提交。

#### TransactionAspectSupport#invokeWithinTransaction

```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
            throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    final TransactionManager tm = determineTransactionManager(txAttr);
    // 响应式事务，主要代表为 RDBC2/MongoDB 的事务
    if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager) {
        // doSomeThing()
    }
    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
　　 // 标准声明式事务：如果事务属性为空 或者 非回调偏向的事务管理器
    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
        Object retVal = null;
        try {
            // 这里就是一个环绕增强，在这个proceed前后可以自己定义增强实现
            // 方法执行
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // 根据事务定义的，该异常需要回滚就回滚，否则提交事务
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            // 清空当前事务信息，重置为老的
            cleanupTransactionInfo(txInfo);
        }
        // 返回结果之前提交事务
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
　　 // 编程式事务：（回调偏向）
    else {
        // doSomething()
    }
}
```
本节主要关注声明式事务，核心流程如下：

- createTransactionIfNecessary():如果有必要，创建事务

- InvocationCallback的proceedWithInvocation()：InvocationCallback是父类的内部回调接口，子类中实现该接口供父类调用，子类TransactionInterceptor中invocation.proceed()。回调方法执行

- 异常回滚completeTransactionAfterThrowing()

#### TransactionAspectSupport#createTransactionIfNecessary

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

    // If no name specified, apply method identification as transaction name.
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            status = tm.getTransaction(txAttr);
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                        "] because no transaction manager has been configured");
            }
        }
    }
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

createTransactionIfNecessary() 的核心：

- getTransaction()：通过调用 PlatformTransactionManager#getTransaction 获取事务状态

- prepareTransactionInfo()，构造一个 TransactionInfo 事务信息对象，绑定当前线程：ThreadLocal<TransactionInfo>。

#### invocation.proceedWithInvocation() 回调业务方法

该方法实际调用 ReflectiveMethodInvocation#proceed() 方法，其源码如下：

```java
public Object proceed() throws Throwable {
    // We start with an index of -1 and increment early.
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // Evaluate dynamic method matcher here: static part will already have
        // been evaluated and found to match.
        InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
        if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        else {
            // Dynamic matching failed.
            // Skip this interceptor and invoke the next in the chain.
            return proceed();
        }
    }
    else {
        // It's an interceptor, so we just invoke it: The pointcut will have
        // been evaluated statically before this object was constructed.
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

该方法进行一些动态校验后，拦截器使用 invoke() 方法调用了目标业务方法。

#### TransactionAspectSupport#completeTransactionAfterThrowing() && completeTransactionAfterReturning()

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    // 对事务属性进行检查，捕获异常后是否需要进行回滚
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
                    "] after exception: " + ex);
        }
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
        }
        else {
            // We don't roll back on this exception.
            // Will still roll back if TransactionStatus.isRollbackOnly() is true.
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                throw ex2;
            }
        }
    }
```

该方法最终调用了 AbstractPlatformTransactionManager#rollback() 方法，与之相似的是，completeTransactionAfterReturning() 最终调用了 commit()。

#### ThreadLocal 的使用

ThreadLocal<TransactionInfo> 使业务代码或者其他切面中，可以拿到 TxInfo，也能拿到 TxStatus。拿到 TxStatus 我们就可以调用 setRollbackOnly 来打标以手动控制事务必须回滚。基于此声明式就可以实现事务的嵌套。

TransactionSynchronizationManager 是 Spring 事务代码中对 ThreadLocal 使用最多的类，目前它内部含有 6 个 ThreadLocal，分别是：

- resources
类型为 Map<Object, Object> 用于保存事务相关资源，比如我们常用的DataSourceTransactionManager会在开启物理事务的时候把 <DataSource, ConnectionHolder> 绑定到线程。
这样在事务作用的业务代码中可以通过 Spring 的 DataSourceUtils 拿到绑定到线程的 ConnectionHolder 中的 Connection。事实上对于 MyBatis 来说与 Spring 集成时就是这样拿的。

- synchronizations
类型为 Set<TransactionSynchronization> 用于保存 transaction synchronization，这个可以理解为是回调钩子对象，内部含有 beforeCommitm，afterCommit，beforeCompletion等钩子方法。
我们自己如果需要的话也可以在业务方法或者切面中注册一些 transaction synchronization 对象用于追踪事务生命周期做一些自定义的事情。

- currentTransactionName
当前事务名

- currentTransactionReadOnly
当前事务是否只读

- currentTransactionIsolationLevel
当前事务隔离级别

- actualTransactionActive
是否存在物理事务，比如传播行为为 NOT_SUPPORTED 时就会是 false。

## 2.3 编程式事务管理源码分析

Spring 提供了模板类 org.springframework.transaction.support.TransactionTemplate 实现编程式事务管理，其类图如 Fig 2.3.1

![transaction-template-diagram](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647704108859transaction-template-diagram.png)

Fig 2.3.1 TransactionTemplate 类图

源码如下：
```java
public class TransactionTemplate extends DefaultTransactionDefinition
		implements TransactionOperations, InitializingBean {

    public TransactionTemplate();
    public TransactionTemplate(PlatformTransactionManager transactionManager);
    public TransactionTemplate(PlatformTransactionManager transactionManager, TransactionDefinition transactionDefinition);

    @Override
    public void afterPropertiesSet() {
        if (this.transactionManager == null) {
            throw new IllegalArgumentException("Property 'transactionManager' is required");
        }
    }

    @Override
    @Nullable
    public <T> T execute(TransactionCallback<T> action) throws TransactionException;

    private void rollbackOnException(TransactionStatus status, Throwable ex) throws TransactionException;
}

```
TransactionTemplate 实现了 TransactionOperations 和 InitializingBean，前者用来执行事务的回调方法，后者执行 bean 加载完毕后的方法。

execute() 源码如下： 

```java
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
    Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");

    if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
        return ((CallbackPreferringPlatformTransactionManager) this.transactionManager).execute(this, action);
    }
    else {
        // 1. 获取事务状态
        TransactionStatus status = this.transactionManager.getTransaction(this);
        T result;
        try {
            // 2. 执行业务逻辑
            result = action.doInTransaction(status);
        }
        // 3. 捕获异常/错误后进行回滚
        catch (RuntimeException | Error ex) {
            // Transactional code threw application exception -> rollback
            rollbackOnException(status, ex);
            throw ex;
        }
        catch (Throwable ex) {
            // Transactional code threw unexpected exception -> rollback
            rollbackOnException(status, ex);
            throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
        }
        // 4. 事务提交
        this.transactionManager.commit(status);
        return result;
    }
}
```
execute() 主要步骤：

- getTransaction() 获取事务。

- doInTransaction() 执行业务逻辑，这里就是用户自定义的业务代码。如果是没有返回值的，就是doInTransactionWithoutResult()。

- commit()事务提交：调用 AbstractPlatformTransactionManager#commit()；rollbackOnException() 异常回滚：调用 AbstractPlatformTransactionManager#rollback()。

## 2.4 AbstractPlatformTransactionManager 源码阅读 

不管是编程式事务，还是声明式事务，最终源码都是调用事务管理器的 PlatformTransactionManager 接口的3个方法：

- getTransaction()
- commit()
- rollback()

不同的事务管理器提供了以上三个接口的不同实现，笔者主要阅读了 Spring 提供的 AbstractPlatformTransactionManager。

### 2.4.1 getTransaction()

### 2.4.2 commit()

### 2.4.3 rollback()

# 3. 注解 @Transaction 利弊分析

正如前文介绍所说，声明式事务管理，对业务代码没有侵入性，我们可以专注于业务逻辑。然而声明式真的有这么好么？

## 3.1 声明式事务的粒度

- 声明式事务的最小粒度作用在方法上，如果想要给一部分代码块增加事务的话，那就需要把这个部分代码块单独独立出来作为一个方法。

- 声明式事务容易被开发者忽略
如果开发者没有注意到一个方法是被事务嵌套的，那么就可能会再方法中加入一些如RPC远程调用、消息发送、缓存更新、文件写入等操作，这会导致两个问题：
    - 这些操作自身是无法回滚的，这就会导致数据的不一致。
    - 在事务中有远程调用，就会拉长整个事务。那么久会导致本事务的数据库连接一直被占用，那么如果类似操作过多，就会导致数据库连接池耗尽。

## 3.2 声明式事务拦截器失效

以下几种场景就可能导致声明式事务失效：

- @Transactional 应用在非 public 修饰的方法上
- @Transactional 注解属性 propagation 设置错误
- @Transactional 注解属性 rollbackFor 设置错误
- 同一个类中方法调用，导致 @Transactional 失效
- 异常被 catch 捕获导致 @Transactional 失效
- 数据库引擎不支持事务

因为Spring的事务是基于AOP实现的，但是在代码中，有时候我们会有很多切面，不同的切面可能会来处理不同的事情，多个切面之间可能会有相互影响。比如目前我们的错误日志打印类，就是通过 AOP 对异常进行捕获。如果不设置 @Transactional 切面和日志打印切面的优先级，将会使事务失效。

## 3.3 阿里规约

>【参考】@Transactional事务不要滥用。事务会影响数据库的QPS，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等

# 5. Conclution

- 设计模式：
    - 模板模式：编程式事务源码
    - 代理模式：声明式事务源码
    
- 面向接口编程：事务管理器的高度抽象 PlatformTransactionManager，定义 getTransaction()、commit()、rollback() 等方法；AbstractPlatformTransactionManager 抽象类实现通用的获取事务、提交事务、回滚事务；DataSourceTransactionManager 等实现类，针对不同的数据源，实现了特性接口。

- Spring 的注释简单易懂且编码整洁，可读性极高。除此之外，其丰富的日志也让我们能更好的理解源码。
