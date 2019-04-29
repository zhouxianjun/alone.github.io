---
title: redisson 分布式锁
date: 2019-04-29 21:13:00
tags:
- redisson
- redis
- lock
- distributed
- 分布式锁

categories:
- 后端
- Java
---

在很多场景中，我们为了保证数据的最终一致性，需要很多的技术方案来支持，比如分布式事务、分布式锁等。有的时候，我们需要保证一个方法在同一时间内只能被同一个线程执行。在单机环境中，Java中其实提供了很多并发处理相关的API，例如：`ReentrantLock`与`synchronized`等，
但是这些API在分布式场景中就无能为力了。也就是说单纯的Java API并不能提供分布式锁的能力。所以针对分布式锁的实现目前有多种方案。

下面我们主要说使用Redisson(Redis开源框架)[分布式锁和同步器](https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器)做分布式锁，当然你也可以直接使用Redis操作，例如lua脚本，保证一致性。

### 单锁
```java
/**
     * 单个锁
     * @param times 等待时长(毫秒)
     * @param keys key 多个key自动以冒号连接
     * @return
     * @throws TimeoutException
     */
public RLock lock(long times, String ...keys) throws TimeoutException {
    // 因为redis默认以冒号作为目录分隔（redis没有目录这一说）
    String key = String.join(":", keys);
    RLock rLock = redissonClient.getFairLock(key);
    try {
        // 设置锁住时间为等待时间的2倍
        boolean res = rLock.tryLock(times, times * 2, TimeUnit.MILLISECONDS);
        if (!res) {
            throw new TimeoutException(key);
        }
    } catch (InterruptedException | TimeoutException e) {
        log.warn("分布式锁:{} 超时", key, e);
        throw new TimeoutException(key);
    }
    return rLock;
}
```
### 联锁
```java
public RedissonMultiLock lock(long times, Set<String> keys) throws TimeoutException {
    RLock[] locks = keys.stream().map(key -> redissonClient.getFairLock(key)).toArray(RLock[]::new);
    RedissonMultiLock lock = new RedissonMultiLock(locks);
    try {
        boolean res = lock.tryLock(times, times * 2, TimeUnit.MILLISECONDS);
        if (!res) {
            throw new TimeoutException();
        }
    } catch (InterruptedException | TimeoutException e) {
        log.warn("分布式联锁:{} 超时", keys, e);
        throw new TimeoutException();
    }
    return lock;
}
```

### 完整代码
```java
package com.alone.common.lock;

import lombok.extern.slf4j.Slf4j;
import org.redisson.RedissonMultiLock;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.stream.Collectors;

/**
 * @author zhouxianjun(Alone)
 * @ClassName:
 * @Description:
 * @date 2018/10/14 10:36
 */
@Slf4j
@Component
public class DistributedLock {
    public static final long TIMES = 1000 * 30;
    @Autowired
    private RedissonClient redissonClient;

    /**
     * 锁
     * @param times 等待时长(毫秒)
     * @param keys key 多个key自动以冒号连接
     * @return
     * @throws TimeoutException
     */
    public RLock lock(long times, String ...keys) throws TimeoutException {
        String key = String.join(":", keys);
        RLock rLock = redissonClient.getFairLock(key);
        try {
            boolean res = rLock.tryLock(times, times * 2, TimeUnit.MILLISECONDS);
            if (!res) {
                throw new TimeoutException(key);
            }
        } catch (InterruptedException | TimeoutException e) {
            log.warn("分布式锁:{} 超时", key, e);
            throw new TimeoutException(key);
        }
        return rLock;
    }

    public RLock lock(String ...keys) throws TimeoutException {
        return lock(TIMES, keys);
    }

    public RedissonMultiLock lockAutoJoin(long times, Set<String[]> keys) throws TimeoutException {
        Set<String> kSet = keys.stream().map(strings -> String.join(":", strings)).collect(Collectors.toSet());
        return lock(times, kSet);
    }

    public RedissonMultiLock lock(long times, Set<String> keys) throws TimeoutException {
        RLock[] locks = keys.stream().map(key -> redissonClient.getFairLock(key)).toArray(RLock[]::new);
        RedissonMultiLock lock = new RedissonMultiLock(locks);
        try {
            boolean res = lock.tryLock(times, times * 2, TimeUnit.MILLISECONDS);
            if (!res) {
                throw new TimeoutException();
            }
        } catch (InterruptedException | TimeoutException e) {
            log.warn("分布式联锁:{} 超时", keys, e);
            throw new TimeoutException();
        }
        return lock;
    }

    public RedissonMultiLock lockAutoJoin(Set<String[]> keys) throws TimeoutException {
        return lockAutoJoin(TIMES, keys);
    }

    public RedissonMultiLock lock(Set<String> keys) throws TimeoutException {
        return lock(TIMES, keys);
    }
}
```

> 不管锁分布式锁还是Java API的各种锁，只要用了锁则会损耗一定的性能作为代价，所以要在需要的地方使用。
