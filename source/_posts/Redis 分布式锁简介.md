---
title: Redis分布式锁简介
layout: post
date: 2022-03-28 02:39:53
tags:
- redis
- 分布式锁
- 分布式算法
categories:
- redis
---

# 1. 分布式锁简介

在分布式场景下，为了保持数据的最终一致性，可以通过**分布式锁**、**分布式事务**来满足业务需求。并发场景下，我们一般需要一个方法在某个时间段内只被一个线程执行。单机环境中，JUC 提供的 API 可以解决并发问题。分布式环境与单机环境有以下区别：

- 分布式场景不是多线程并发而是多进程并发。
- 单机场景多线程共享堆内存，可以直接使用内存作为标记的存储位置；分布式场景进程之间可能不在一台物理机上，需要使标记对所有进程可见。

## 1.1 分布式锁的应用场景

Matin Kleppmann 是剑桥大学的分布式系统研究员，他认为分布式锁的使用场景一般是为了效率或者正确性。

- 效率：加锁可以避免重复工作，节约资源（例如一些昂贵的计算），避免一些代价较小的问题。
- 正确性：加锁可以避免破坏正确性操作，如果锁失败，两个节点对同一条数据进行操作，可能导致文件损坏、数据丢失、永久性不一致等代价高昂的问题。

## 1.2 分布式锁的特点

- 互斥性：和我们本地锁一样互斥性是最基本，但是分布式锁需要保证在不同节点的不同线程的互斥。
- 可重入性：同一个节点上的同一个线程如果获取了锁之后那么也可以再次获取这个锁。
- 锁超时：和本地锁一样支持锁超时，防止死锁。
- 高效，高可用：加锁和解锁需要高效，同时也需要保证高可用防止分布式锁失效，可以增加降级。
- 支持阻塞和非阻塞：和 ReentrantLock 一样支持 lock 和 trylock 以及 tryLock（long timeout）。
- 支持公平锁和非公平锁（可选）：公平锁的意思是按照请求加锁的顺序获得锁，非公平锁就相反是无序的。这个一般来说实现的比较少。

# 2. RedLock 算法

## 2.1 单 Redis 实例分布式锁实现分析

使用 Redis 锁定资源最简单的方法是客户端请求锁时在单个 Redis 实例中创建 key，并加上过期时间（TTL），防止死锁。当客户端释放锁，删除 key。此时我们的架构中只有一个 Redis 实例，无法应对单点失败的场景。如果使用主从结构，由于 Redis 的备份异步进行，该情况下我们无法实现锁的互斥性。

一个简单的竞争场景（主从结构）如下：

1. 客户端 A 在 master 节点中获得锁；
2. 在将 key 备份到 slave 节点前，master 节点间宕机；
3. slave 节点被提为 master 节点；
4. 客户端 B 请求与 A 相同的资源并成功获得锁，此时 A 并未释放该资源。

### 单 Redis 实例分布式锁的正确实现

可以通过增设随机值实现锁的互斥性，安全地获取锁和释放锁，该算法是实现 Redis 分布式锁的基础。

> set resource_name my_random_value NX PX 30000

**my_random_value** 是一个随机值，该锁的所有竞争者的随机值都不能一样。释放锁时，只有 key 存在且存储的值和设置的值相同才能删除成功。可以通过以下 lua 脚本实现：
```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```
通过 lua 脚本保证了 get-check-delete 是一个原子操作。

基于Redis单实例，假设这个单实例总是可用，这种方法已经足够安全。

## 2.2 Redlock 算法概述

在 Redis 分布式环境中，假设存在 N 个 Redis master，这些节点完全独立，且不存在主从复制或者其他集群协调机制，并且这些节点使用单节点的方法安全地请求和释放锁。

接下来介绍几个重要环节。

### 2.2.1 请求锁

1. 获取当前 Unix 时间 **startTime**，单位为 ms。
2. 依次尝试 N 个实例，使用相同的 key 和 随机值获得锁。在向 Redis 实例请求锁时，需要设置一个网络连接和响应超时的时间，避免实例挂掉客户端继续等待的情况。如果响应时间超时，客户端尝试下一个实例。
3. 使用当前时间减去 **startTime** 得到锁获得时间 **getKeyDuration**，当且仅当从大多数 Redis 节点（N / 2 + 1）获取到锁，并且 max{**getKeyDuration**} 小于锁失效时间，锁才获得成功。
4. key 的真正有效时长 VT（Valid Time）= expireTime - getKeyDuration。
5. 当获取锁失败时，客户端应该在所有获取锁成功的实例上解锁。同时应该在一个随机时延后再重试，防止多个客户端同时竞争同一个锁（导致脑裂，即不同客户端不能完全获得所有实例的锁）。

### 2.2.2 释放锁

向所有 Redis 实例发送释放锁命令。这个点看起来比较简单，笔者当时觉得这只是一种类 roll back 操作，且仅仅针对获取锁成功的实例。实际上从分布式系统的角度上来看，作为一个异步通信模型，不仅仅需要考虑客户端到服务器的通信，也要考虑服务器到客户端是否正常。

设想这样一个场景：客户端申请锁的请求到达某个 Redis 节点，且该节点也执行了 SET 操作，但是由于网络问题，节点返回的响应包丢失。在 Redis 看来加锁成功，在客户端看来请求锁的操作由于超时失败。因此为了避免脑裂的发生，需要对所有 Redis 实例发送释放锁命令。

### 2.2.3 Redlock 安全性讨论

Redlock 通过设置两个必要条件（获得大多数锁且实际VT大于0）保证锁的互斥性。然而在某些节点崩溃重启的场景下，Redlock 真的互斥吗？

假设一共有 5 个 Redis 实例 A、B、C、D、E，客户端 1、2 在请求锁时发生以下事件序列：
1. 客户端成功锁住实例 ABC，由于网络延迟等原因没有锁住 DE。
2. 节点 C 崩溃重启，客户端 1 在实例 C 上设置的 key 没有完成持久化。
3. 节点 C 重启，客户端 2 成功锁住实例 CDE。
4. 客户端 1,2 都认为自己持有锁。

以上场景两个客户端同时获得了锁，破坏了锁的互斥性。默认情况下，Redis 的 AOF 持久化方式是 `appendfsync everysec`，最坏的情况是丢失一秒的数据，为了安全性可以将其设置为 `appednfsync always`。除此以外，为了应对这一情况，Redlock 提出了一种**延迟重启**的方案，即 Redis 实例宕机后的重启时间应该大于客户端的有效时间，使节点对客户端不可用，以在改动 AOF 策略情况下保证锁的安全性。

# 3. Martin Kleppmann 与 antirez 的争论

2016年2月8日 Kleppmann 在博客上发表了一篇文章 《[How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)》 对 Redlock 的安全性进行讨论。后来 antirez 在个人网站上发表了文章 《[Is Redlock safe?](http://antirez.com/news/101)》 回复了 Kleppmann 的质疑。

## 3.1 Kleppmann 的观点

Kleppmann 的文章大致可以分为两部分：

- 与 Redlock 无关。主要是分布式锁的相关问题，他分析得出即使存在一个完美实现的分布式锁（且实现了自动过期时间功能），如果服务端不提供 fencing 等机制，分布式锁也是不安全的。
- 对 Redlock 本身的批评。由于 Redlock 本质上是建立在一个同步模型之上，对系统的记时假设（timing assumption）有很强的要求，因此本身的安全性不够。

### 3.1.1 与 Redlock 无关的部分

Martin 给出了如下的时序图 Fig.1：

![Fig.1. unsafe lock](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647704258856unsafe-lock.png)

_Fig.1. 不安全的分布式锁时序图_

在上面的时序图中，假设锁服务本身是没有问题的，它总是能保证任一时刻最多只有一个客户端获得锁。`lease` 这个词等同于一个带有自动过期功能的锁。客户端 1 在获得锁之后发生了很长时间的 GC pause，在此期间，它获得的锁过期了，而客户端2获得了锁。当客户 1 从 GC pause 中恢复过来的时候，它不知道自己持有的锁已经过期了，它依然向共享资源（上图中是一个存储服务）发起了写数据请求，而这时锁实际上被客户端 2 持有，因此两个客户端的写请求就有可能冲突（锁的互斥作用失效了）。

但是如果使用没有 GC 的语言来实现客户端呢？Martin 认为系统环境非常复杂，仍然有很多原因导致进程的 pause，比如虚存造成的缺页故障（page fault），再比如CPU资源的竞争等。即使不考虑进程 pause，网络延迟也会导致类似的结果。总结来说即是**客户端在获得锁后进程长期阻塞导致锁过期**会不会影响锁的安全性。

Martin 给出的解决方案是 fencing token，简单来说是通过一个单调递增的数字来检查客户端访问共享资源时的顺序，其时序图如 Fig.2：

![Fig.2. fencing tokens](https://yartang-blogs.oss-cn-beijing.aliyuncs.com/blog-imgs/1647704280843fencing-tokens.png)

_Fig.2. Fencing Tokens 时序图_

Fig.2 中，客户端 1 先获取到锁，其 fencing token = 33。分布式锁失效，客户端 2 取得锁，其 fencing token = 34，操作共享资源。客户端 1 从 GC 中恢复，请求操作共享资源，存储服务发现其 token 小于 34，拒绝该操作，从而避免冲突。

**Note**：由于此处 Martin 并没有深入讲解 fencing token 的实现机制，因此笔者对此处也有很多疑问。比如两个客户端同时因为 GC 等原因阻塞进程，然而他们以相同的时间到达存储服务器，同时对资源进行操作，这也会破坏算法的安全性。

### 3.1.2 Redlock 部分

文章后半部分主要说明 Redlock 是一个同步模型，通过构造一些事件序列说明 Redlock 对系统计时过分依赖。首先构造的事件序列如下：

1. 客户端成功锁住实例 ABC，由于网络延迟等原因没有锁住 DE。
2. 节点 C 系统计时向前跳跃，客户端 1 在实例 C 上设置的 key 快速过期。
3. 客户端 2 成功锁住实例 CDE。
4. 客户端 1,2 都认为自己持有锁。

上述情况的发生本质上是由于 Redis 的时间获取的是 Unix 时间，底层使用 `gettimeofday()`而不是单调时钟`clock_gettime()`，会受到时间不连续跳跃的影响（管理员手动修改时间、ntp 服务器同步时间等）。一旦系统时间不准确，算法的安全性也得不到保证。
随后 Martin 继续设想了一个更加极端的场景：

1. 客户端 1 请求锁定节点 ABCDE。
2. 当对客户端 1 的响应在进行中时，客户端 1 进入长时间 GC。
3. 所有 Redis 节点上的锁都会过期。
4. 客户端 2 获取节点 ABCDE 上的锁。
5. 客户端 1 完成 GC，并接收来自 Redis 节点的响应，表明它已成功获取锁（在进程暂停时，它们保存在客户端 1 的内核网络缓冲区中）。
6. 客户端 1 和 2 现在都相信他们持有锁。

Martin 认为即使不考虑 GC 的影响，长时间的网络延迟也会导致上述情况的出现。实际上 antirez 在设计 Redlock 时就考虑了该问题，客户端 1 认为自己持有锁的必要条件之一就是真正有效时间（valid time）大于 0（antirez 在他的反驳文章中也对这个问题进行了反驳）。 

我们知道好的分布式算法应该基于异步模型（Asynchronous Model），算法的安全性不应该依赖于任何计时假设。在异步模型中：进程可能 pause 任意长的时间，消息可能在网络中延迟任意长的时间，甚至丢失，系统时钟也可能以任意方式出错。一个好的分布式算法，这些因素不应该影响它的安全性（safety property），只可能影响到它的活性（liveness property），也就是说，即使在非常极端的情况下（比如系统时钟严重错误），算法顶多是不能在有限的时间内给出结果而已，而不应该给出错误的结果。显然按照此标准，Redlock 的安全性是达不到的。

本文开头对分布式锁的应用场景的叙述就是 Kleppmann 的观点，他根据对 Redlock 的分析得出如下结论：
- 效率：如果使用分布式锁的目的是为了提高系统效率，直接使用单实例的 Redis 进行锁操作，而不是使用多个 Redis 节点构成的集群，浪费资源。
- 正确性：如果使用分布式锁的目的是保证正确性，那么不要使用 Redlock，建议使用类似 Zookeeper 的方案，或者是支持事务的数据库。

以及他认为选择 Redlock 是 `poor choice`，因为它 `neither fish nor fowl（非鱼非驴）`。

## 3.2 antirez 的反驳
antirez 也简单概括了 Kleppmann 的观点，即文章的前半部分和后半部分。他也就这两方面进行了反驳。

### 3.2.1 fencing token 机制

1. antirez 认为既然锁失效的情况下，fencing token 机制就能保证多进程对共享资源进行安全地访问，那为什么还需要分布式锁呢？原文如下：
> It is not clear why you would use a lock with strong properties at all if you can resolve races in a different way.

2. 如果使用一个单调递增的 token，那么共享资源就是一个线性存储系统。然而大多数场景下，对共享资源写入并不是线性化的。Redlock 虽然没有提供单调递增的 fencing token，但是提供了唯一的随机字符串（unique token），也可以达到同样的效果。原文中提到，可以使用 unique token 完成一个 CAS操作：假设存在一个共享资源 X

> 1. 设置 X.curlock = token。
> 2. 读出资源 X 以及附带的 curlock。
> 3. 按照”write-if-currlock == token”的逻辑，修改资源X的值。

同时 antirez 认为第一部分的批评适用于所有具有超时释放功能且没有提供单调递增 token 的分布式锁。

### 3.2.2 系统模型

Kleppmann 认为 Redlock 失败的情况主要有以下两种：
- 系统计时发生不连续跳跃
- 长时间的 GC 或长时间的网络延迟（虽然造成两种情况的成因不同，但是笔者认为逻辑上对分布式锁系统造成的影响是类似的，即客户端认为他持有该锁时，锁在 Redis 节点处已经失效）。

对于第一种情况，Kleppmann 举了两个系统时钟跳跃的例子：

1. 管理员手动调整
2. NTP 服务器调整

antirez 认为 1 是不可避免的，如果有人对 raft 日志进行修改，那么 raft 也无法正常工作。对于 2 可以通过配置 nptd 慢慢修改时间，而不是大跨度的修改系统时间。Redlock 由于使用相对时间，即使存在误差，在实际情况中也是可以接受的。

对于后一种情况来说，antirez 在设计 Redlock 时已经考虑到了，并设置了真正有效时间 valid time 作为必要条件之一。他在博客中也重新分析了这个问题，我们先回顾一下客户端[请求锁](#2.2.1)的过程。

我们可以知道由于 valid time 的设置，在第一步和第三步之间发生 GC 或是网络延迟都不会影响 Redlock 的安全性，都能被第四步检测出来。只有发生在第三步之后的 GC 或者延迟会影响锁服务的安全性。antirez 有一段比较有意思的论述：
> The delay can only happen after steps 3, resulting into the lock to be considered ok while actually expired, that is, we are back at the first problem Martin identified of distributed locks where the client fails to stop working to the shared resource before the lock validity expires. Let me tell again how this problem is common with **all the distributed locks implementations**, and how the token as a solution is both unrealistic and can be used with Redlock as well.
译文：只有发生在第三步之后的延时会导致，客户端认为锁服务是正常的但实际上已经过期。我们回到 Martin 的第一个问题，即是客户端没有在锁失效之前完成对共享资源的操作，这是 **所有分布式锁（这里特指带自动过期功能的锁）** 的共同问题。而且基于 token 这种方案是不切实际的，但是也能和 Redlock 一起使用。

笔者认为只要认同 antirez 对时钟的论断，他的分析看起来是无懈可击的。

# 4. Redission 简介

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括(`BitSet`, `Set`, `Multimap`, `SortedSet`, `Map`, `List`, `Queue`, `BlockingQueue`, `Deque`, `BlockingDeque`, `Semaphore`, `Lock`, `AtomicLong`, `CountDownLatch`, `Publish / Subscribe`, `Bloom filter`, `Remote service`, `Spring cache`, `Executor service`, `Live Object service`, `Scheduler service`) Redisson提供了使用Redis的最简单和最便捷的方法。本文主要介绍分布式锁（Distributed lock）中可重入锁（Reentrant Lock）和 Redlock 的实现。

## 4.1 Reentrant Lock 实现

[官方 WIKI](https://github.com/redisson/redisson/wiki/8.-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E5%92%8C%E5%90%8C%E6%AD%A5%E5%99%A8)说明可重入锁 `RLock` Java对象实现了 `java.util.concurrent.locks.Lock` 接口，用法类似 juc 中的 `lock`。

通过下面的例程简单说明一下可重入锁的实现。

```java
public class App {

    public static void main(String[] args) {

        // 1. Redis 配置
        Config config = new Config();
        config.useClusterServers().addNodeAddress("redis://127.0.0.1:7181");
        // 2. 构造 RedissionClient
        RedissonClient redisson = Redisson.create(config);
        // 3. 获取锁实例
        RLock rLock = redisson.getLock("reentrant lock test");
        // 4. 获取分布式锁
        rLock.lock();

        try {
            System.out.println("reentrant lock test");
        } finally {
            // 5. 释放锁
            rLock.unlock();
        }
    }
}
```

### getLock() 方法

```java
    // RedissionLock.java
    @Override
    public RLock getLock(String name) {
        return new RedissonLock(commandExecutor, name);
    }
```
`RLock` 是一个继承自 `java.util.concurrent.locks.Lock` 的接口，返回了一个 `RedissonLock` 实例。

RedissonLock 的构造参数是：
- commandExecutor：是与 Redis 节点进行通信的具体实现。主要通过 `eval` 命令执行 `lua` 脚本来实现，此处不展开讲解。
- name：锁的全局名称，即是锁在 Redis 节点中的 key。

### lock() 方法

```java
    @Override
    public void lock() {
        try {
            lock(-1, null, false);
        } catch (InterruptedException e) {
            throw new IllegalStateException();
        }
    }

    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        // 1. 尝试获取锁
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
        // 2. 订阅锁
        RFuture<RedissonLockEntry> future = subscribe(threadId);
        // 3. 中断处理
        if (interruptibly) {
            commandExecutor.syncSubscriptionInterrupted(future);
        } else {
            commandExecutor.syncSubscription(future);
        }

        try {
            while (true) {
                // 4. 重新尝试获取锁
                ttl = tryAcquire(-1, leaseTime, unit, threadId);
                // 5. lock acquired 锁获取成功
                if (ttl == null) {
                    break;
                }

                // 6. waiting for message 等待锁释放
                if (ttl >= 0) {
                    try {
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                } else {
                    if (interruptibly) {
                        future.getNow().getLatch().acquire();
                    } else {
                        future.getNow().getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            // 7. 取消订阅
            unsubscribe(future, threadId);
        }
    }
```

### tryAquire() 方法

重点分析 `tryAquire()` 方法。

```java
    private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        // 1. 异步执行
        return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));
    }

    private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        RFuture<Long> ttlRemainingFuture;
        // 2. 根据有效时间尝试异步加锁
        if (leaseTime != -1) {
            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        } else {
            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                    TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        }
        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            if (e != null) {
                return;
            }

            // 3. lock acquired 锁获得成功
            if (ttlRemaining == null) {
                if (leaseTime != -1) {
                    internalLockLeaseTime = unit.toMillis(leaseTime);
                } else {
                    // 4. 锁过期时间刷新任务调度
                    scheduleExpirationRenewal(threadId);
                }
            }
        });
        return ttlRemainingFuture;  
    }

    <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        // 5. 使用 eval 命令执行 lua 脚本获取锁
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "return redis.call('pttl', KEYS[1]);",
                Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
    }
```

### unlock() 方法

```java
    @Override
    public void unlock() {
        try {
            get(unlockAsync(Thread.currentThread().getId()));
        } catch (RedisException e) {
            if (e.getCause() instanceof IllegalMonitorStateException) {
                throw (IllegalMonitorStateException) e.getCause();
            } else {
                throw e;
            }
        }
    }

    protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                        "return nil;" +
                        "end; " +
                        "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                        "if (counter > 0) then " +
                        "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                        "return 0; " +
                        "else " +
                        "redis.call('del', KEYS[1]); " +
                        "redis.call('publish', KEYS[2], ARGV[1]); " +
                        "return 1; " +
                        "end; " +
                        "return nil;",
                Arrays.asList(getRawName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
    }
```

## 4.2 Redlock 实现
 
基于 Redis 的 Redisson 红锁 RedissonRedLock 对象实现了 Redlock 介绍的加锁算法。该对象也可以用来将多个 RLock 对象关联为一个红锁，每个 RLock 对象实例可以来自于不同的Redisson实例。

实际上 RedissonRedLock 类已经被弃用了。
> This object is deprecated. RLock operations now propagated to all Redis slaves.

被弃用的原因：

> The main idea behind RedLock algorithm is to overcome losing of lock acquisition during failover since Redis replication is asynchronous. Moreover its effectiveness still subject of dispute https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
> To overcome this every Redis command execution made by RLock object is synchronized through WAIT command introduced in Redis 3.0.

TODO 源码阅读

# 5. 与其他分布式锁算法的比较

Martin 认为如果需要使用一个更加安全的分布式锁，应该使用 Zookeeper，而不是 Redlock，本节来分析一下 Zookeeper 的方案。

TODO

# 6. 结论

- 按照锁的两种用途，如果仅是为了效率(efficiency)，那么你可以自己选择你喜欢的一种分布式锁的实现。当然，你需要清楚地知道它在安全性上有哪些不足，以及它会带来什么后果。而如果你是为了正确性(correctness)，那么需要慎重考虑。

- Martin Kleppman 的新书《Designing Data-Intensive Applications》中会解答他在博客中的一些比较模糊的细节，需要在之后的学习中在慢慢研究。

- RedissionRedlock 的弃用原因还需要好好研究。
