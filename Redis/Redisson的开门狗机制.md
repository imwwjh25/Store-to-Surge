Redisson 的 **看门狗机制（Watch Dog）** 是解决分布式锁因持有锁线程异常（如崩溃、网络中断）或任务超时导致锁无法释放问题的核心机制，其核心目标是：**在持有锁的线程未完成任务前，自动延长锁的有效期，防止锁被提前释放**。

### 一、看门狗机制的核心背景

分布式锁（如基于 Redis 的 `SET key value NX PX timeout` 实现）通常需要设置过期时间（防止死锁），但存在一个矛盾：

- 若过期时间设置过短，可能导致持有锁的线程尚未完成任务，锁就已自动释放，引发并发安全问题；
- 若过期时间设置过长，一旦持有锁的线程异常崩溃，锁会长期占用，导致其他线程阻塞。

Redisson 的看门狗机制正是为解决此矛盾而生：**让锁的过期时间 “动态适配任务执行时间”**，在任务未完成时自动续期，任务完成后正常释放锁。

### 二、看门狗机制的工作原理

#### 1. 核心逻辑

当线程成功获取分布式锁时，若未指定锁的过期时间（或设置为 `-1`），Redisson 会自动启用看门狗机制，流程如下：

- **初始锁过期时间**：默认设置为 30 秒（可通过 `config.setLockWatchdogTimeout(ms)` 全局修改）。

- 启动后台续约线程 ：获取锁的线程会启动一个

   

  定时任务（看门狗）

  ，每隔 10 秒（默认是锁过期时间的 1/3）检查当前线程是否仍持有锁：

  - 若持有（任务未完成），则自动延长锁的过期时间至 30 秒；
  - 若未持有（任务已完成或线程异常），则停止续约。

- **锁释放**：当线程执行完任务并调用 `unlock()` 时，会删除锁并停止看门狗定时任务。

#### 2. 关键细节

- **续约条件**：只有当前线程仍持有锁（即 Redis 中锁的 value 仍是当前线程的 ID）时，才会续约，避免为其他线程持有的锁续期。
- **异常处理**：若持有锁的线程崩溃或网络中断，看门狗定时任务会随线程终止，不再续约，锁会在 30 秒后自动过期释放，避免死锁。
- **自定义过期时间**：若用户手动指定了锁的过期时间（如 `lock(10, TimeUnit.SECONDS)`），则看门狗机制**不生效**，锁会在指定时间后自动释放（需确保任务能在过期前完成）。

### 三、源码层面的实现逻辑

Redisson 的分布式锁核心类为 `RedissonLock`，看门狗机制的关键代码如下：

#### 1. 获取锁时启动看门狗

当调用 `lock()` 方法且未指定过期时间时，会触发 `tryAcquireAsync(-1, ...)`，其中：





```java
private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
    if (leaseTime != -1) { // 手动指定过期时间，不启用看门狗
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    }
    // 未指定过期时间，启用看门狗，默认 30 秒
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(
        commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(),
        TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG
    );
    ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
        if (e != null) {
            return;
        }
        // 锁获取成功，启动看门狗续约任务
        if (ttlRemaining == null) {
            scheduleExpirationRenewal(threadId);
        }
    });
    return ttlRemainingFuture;
}
```

#### 2. 定时续约任务（看门狗核心）

`scheduleExpirationRenewal(threadId)` 方法会启动一个定时任务，每隔 10 秒（锁过期时间的 1/3）执行一次续约：








```java
private void scheduleExpirationRenewal(long threadId) {
    ExpirationEntry entry = new ExpirationEntry();
    ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
    if (oldEntry != null) {
        oldEntry.addThreadId(threadId);
        return;
    }
    // 启动定时任务
    Timeout task = commandExecutor.getConnectionManager().newTimeout(timerTask -> {
        // 执行续约 Lua 脚本，延长锁过期时间至 30 秒
        RFuture<Boolean> future = renewExpirationAsync(threadId);
        future.onComplete((res, e) -> {
            if (e != null) {
                log.error("Can't update lock " + getName() + " expiration", e);
                return;
            }
            if (res) {
                // 续约成功，继续下一次定时任务
                scheduleExpirationRenewal(threadId);
            }
        });
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS); // 间隔为 30s/3=10s
    entry.setTimeout(task);
}
```

续约的 Lua 脚本 `renewExpirationAsync` 逻辑：仅当锁存在且属于当前线程时，才延长过期时间：







```lua
-- KEYS[1] 为锁的 key，ARGV[1] 为过期时间（30s），ARGV[2] 为当前线程 ID
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return 1;
end;
return 0;
```

#### 3. 释放锁时停止看门狗

调用 `unlock()` 时，会删除锁并移除定时任务：









```java
public RFuture<Void> unlockAsync(long threadId) {
    // 执行释放锁的 Lua 脚本
    RFuture<Boolean> future = unlockInnerAsync(threadId);
    future.onComplete((opStatus, e) -> {
        // 移除定时任务，停止续约
        cancelExpirationRenewal(threadId);
        // ... 其他逻辑
    });
    return future;
}
```

### 四、如何防止锁被提前释放？

看门狗机制通过以下三点确保锁不会被提前释放：

1. **动态续期**：只要持有锁的线程正常运行且未释放锁，看门狗就会每隔 10 秒自动将锁的过期时间延长至 30 秒，保证锁的有效期始终覆盖任务执行时间。
2. **线程绑定**：续约操作通过 Lua 脚本校验锁的持有者是否为当前线程（对比 Redis 中存储的线程 ID），避免为其他线程的锁续期，确保安全性。
3. **异常自动释放**：若持有锁的线程崩溃，看门狗定时任务会终止，不再续期，锁会在当前过期时间（如 30 秒）后自动释放，避免死锁。

### 总结

Redisson 的看门狗机制是分布式锁的 “智能续期器”，通过 **定时续约 + 线程绑定校验** 解决了 “锁过期时间难以适配任务时长” 的痛点，既防止了锁提前释放导致的并发问题，又避免了线程异常时的死锁风险，是 Redisson 分布式锁相比原生 Redis 锁的核心优势之一。
