# 分布式系统安全 (Distributed Systems Safety)

## 检查清单

### 1. 分布式锁与事务顺序

#### ❌ 错误模式 - 锁在事务提交前释放

```java
// ❌ 错误：@Transactional 在方法上，锁在 try-finally 中
@Transactional(rollbackFor = Exception.class)
public void decreaseAsset(Long id, Long delta) {
    String lockKey = "LOCK:ASSET:" + id;
    redisLock.lock(lockKey);
    try {
        Account account = accountDao.selectById(id);
        account.setBalance(account.getBalance() - delta);
        accountDao.updateById(account);
        // 事务在这里还未提交！
    } finally {
        redisLock.unlock(lockKey);  // 锁先释放，事务后才提交
        // 其他线程可以获取锁，但事务还未提交，读到旧数据
    }
}

// 时序问题：
// Thread 1: 获取锁 -> 查询(余额100) -> 更新(余额90) -> 释放锁
// Thread 2:                                  获取锁 -> 查询(读到旧数据100) -> 更新(余额90)
// Thread 1:                                                       事务提交(余额90)
// Thread 2:                                                       事务提交(余额90)
// 结果：应该扣除两次，实际只扣除了一次
```

#### ✅ 正确模式 - 编程式事务在锁内执行

```java
// 方式1：编程式事务
public void decreaseAsset(Long id, Long delta) {
    String lockKey = "LOCK:ASSET:" + id;
    redisLock.lock(lockKey);
    try {
        transactionTemplate.execute(status -> {
            Account account = accountDao.selectById(id);
            account.setBalance(account.getBalance() - delta);
            accountDao.updateById(account);
            return null;
        });  // 事务在锁释放前提交
    } finally {
        redisLock.unlock(lockKey);
    }
}

// 方式2：使用 TransactionTemplate
@Autowired
private TransactionTemplate transactionTemplate;

public void decreaseAsset(Long id, Long delta) {
    String lockKey = "LOCK:ASSET:" + id;
    redisLock.lock(lockKey);
    try {
        transactionTemplate.executeWithoutResult(status -> {
            Account account = accountDao.selectById(id);
            account.setBalance(account.getBalance() - delta);
            accountDao.updateById(account);
        });
    } finally {
        redisLock.unlock(lockKey);
    }
}
```

### 2. 分布式锁超时与续期

#### ❌ 错误模式 - 无超时的分布式锁

```java
// ❌ 无超时设置，可能导致永久阻塞
public void processOrder(Long orderId) {
    String lockKey = "LOCK:ORDER:" + orderId;
    redisLock.lock(lockKey);  // 如果获取锁的节点崩溃，永远无法获取
    try {
        // 处理订单
    } finally {
        redisLock.unlock(lockKey);
    }
}
```

#### ✅ 正确模式 - 带超时的锁获取

```java
public void processOrder(Long orderId) {
    String lockKey = "LOCK:ORDER:" + orderId;

    // 尝试获取锁，最多等待 5 秒
    if (!redisLock.tryLock(lockKey, 5, TimeUnit.SECONDS)) {
        throw new BusinessException("系统繁忙，请稍后重试");
    }

    try {
        // 处理订单
    } finally {
        redisLock.unlock(lockKey);
    }
}
```

#### ❌ 错误模式 - 无锁续期机制

```java
// ❌ 手动 Redis SETNX 锁，没有续期机制
public void longRunningTask(Long id) {
    String lockKey = "LOCK:" + id;
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "locked", 30, TimeUnit.SECONDS);

    if (!locked) {
        throw new BusinessException("获取锁失败");
    }

    try {
        // 任务执行可能超过 30 秒
        heavyProcessing();  // 锁过期了，其他线程获取锁，导致并发问题
    } finally {
        redisTemplate.delete(lockKey);
    }
}
```

#### ✅ 正确模式 - 使用 Redisson 看门狗续期

```java
// Redisson 会自动续期（看门狗机制）
@Autowired
private RedissonClient redissonClient;

public void longRunningTask(Long id) {
    RLock lock = redissonClient.getLock("LOCK:" + id);

    try {
        // lock() 会自动续期，直到业务代码执行完毕
        // leaseTime = -1 表示启用看门狗，默认 30 秒续期一次
        if (lock.tryLock(5, -1, TimeUnit.SECONDS)) {
            try {
                heavyProcessing();  // 即使超过 30 秒，锁也会自动续期
            } finally {
                lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new BusinessException("任务被中断", e);
    }
}

// 或手动指定看门狗时间
public void longRunningTask(Long id) {
    RLock lock = redissonClient.getLock("LOCK:" + id);
    lock.lock();  // 默认看门狗时间 30 秒
    try {
        heavyProcessing();
    } finally {
        lock.unlock();
    }
}
```

### 3. 事务与远程调用混用

#### ❌ 错误模式 - @Transactional 无法覆盖远程调用

```java
// ❌ @Transactional 只对本地数据库有效，无法回滚远程调用
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    // 本地数据库操作 - 可以回滚
    orderMapper.insert(dto);

    // 远程 RPC 调用 - 无法回滚！
    // 如果这里失败，本地数据库已经插入，但远程库存未扣减
    stockFeignClient.deduct(dto.getSkuId());

    // 发送消息 - 无法回滚！
    messageProducer.send(new OrderCreatedEvent(dto.getId()));
}

// 问题场景：
// 1. orderMapper.insert() 成功
// 2. stockFeignClient.deduct() 失败（网络超时）
// 3. 事务回滚，订单表数据删除
// 4. 但库存扣减可能已经成功（重试后成功）
// 结果：订单不存在，但库存被扣减
```

#### ✅ 正确模式 - SAGA 模式

```java
// 方式1：使用 SAGA 框架
@SagaStart
public void createOrder(OrderDTO dto) {
    Saga.with("createOrder", () -> {
        // 步骤1：创建订单
        orderService.create(dto);
    })
    .with("deductStock", () -> {
        // 步骤2：扣减库存
        stockClient.deduct(dto.getSkuId());
    })
    .with("sendNotification", () -> {
        // 步骤3：发送通知
        notificationClient.send(dto.getUserId());
    })
    // 补偿动作
    .compensate("createOrder", () -> {
        // 取消订单
        orderService.cancel(dto.getId());
    })
    .compensate("deductStock", () -> {
        // 恢复库存
        stockClient.restore(dto.getSkuId());
    })
    .execute();
}

// 方式2：手动实现补偿事务
public void createOrder(OrderDTO dto) {
    try {
        // 1. 创建订单
        orderService.create(dto);

        try {
            // 2. 扣减库存
            stockClient.deduct(dto.getSkuId());

            try {
                // 3. 发送通知
                notificationClient.send(dto.getUserId());
            } catch (Exception e) {
                // 补偿：取消通知（通常是幂等的）
                log.error("发送通知失败，但不影响主流程", e);
            }

        } catch (Exception e) {
            // 补偿：取消订单
            orderService.cancel(dto.getId());
            // 补偿：恢复库存
            stockClient.restore(dto.getSkuId());
            throw new BusinessException("创建订单失败", e);
        }

    } catch (Exception e) {
        throw new BusinessException("创建订单失败", e);
    }
}

// 方式3：使用消息队列的最终一致性
public void createOrder(OrderDTO dto) {
    // 只保存本地订单
    orderService.create(dto);

    // 发送消息到 MQ，由消费者处理后续逻辑
    messageProducer.send("order-created", dto);
    // 库存扣减、通知等由订阅者异步处理
}

// 消费者处理库存（带重试）
@RocketMQMessageListener(topic = "order-created")
public class StockConsumer {

    @RocketMQMessageListener(consumerGroup = "stock-group")
    public void onMessage(OrderDTO dto) {
        try {
            stockClient.deduct(dto.getSkuId());
        } catch (Exception e) {
            // 失败会自动重试
            throw e;
        }
    }
}
```

### 4. 缓存一致性

#### ❌ 错误模式 - 更新缓存导致脏数据

```java
// ❌ 先更新数据库，再更新缓存 - 存在时序问题
public void updateProduct(Product product) {
    // Thread A: 更新数据库
    productDao.updateById(product);

    // Thread B: 更新数据库（A 还未更新缓存）
    // Thread A: 更新缓存（旧数据）
    // Thread B: 更新缓存（新数据） <- 缓存是新数据
    // 但如果 Thread A 慢，可能：
    // Thread A: 更新数据库（新数据）
    // Thread B: 更新数据库（新数据2）
    // Thread B: 更新缓存（新数据2）
    // Thread A: 更新缓存（新数据） <- 缓存是旧数据！
    redisTemplate.opsForValue().set("PRODUCT:" + product.getId(), product, 30, TimeUnit.MINUTES);
}
```

#### ✅ 正确模式 - Cache-Aside 模式

```java
// 方式1：先删除缓存，再更新数据库（推荐）
public void updateProduct(Product product) {
    Long id = product.getId();

    // 先删除缓存
    redisTemplate.delete("PRODUCT:" + id);

    // 再更新数据库
    productDao.updateById(product);

    // 问题：删除缓存后、更新数据库前，有读请求会怎样？
    // 读请求：缓存未命中 -> 读数据库（旧数据）-> 写入缓存（旧数据）
    // 解决：延迟双删
}

// 方式2：延迟双删
public void updateProduct(Product product) {
    Long id = product.getId();

    // 第一次删除缓存
    redisTemplate.delete("PRODUCT:" + id);

    // 更新数据库
    productDao.updateById(product);

    // 延迟删除缓存（确保其他线程读到的旧缓存也被删除）
    ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
    executor.schedule(() -> {
        redisTemplate.delete("PRODUCT:" + id);
    }, 500, TimeUnit.MILLISECONDS);
}

// 方式3：读写分离 + 订阅 Binlog（推荐用于高一致性要求）
// 使用 Canal 订阅 MySQL Binlog，数据变更时自动刷新缓存

// 方式4：先更新数据库，再删除缓存（大多数场景）
public void updateProduct(Product product) {
    // 更新数据库
    productDao.updateById(product);

    // 删除缓存（而不是更新）
    redisTemplate.delete("PRODUCT:" + product.getId());

    // 下次读取时会重新加载缓存
    // 问题：读请求可能在删除前读到旧缓存
    // 但这个问题影响较小，因为缓存过期后会自动刷新
}
```

#### ⚠️ 缓存穿透问题

```java
// ❌ 恶意请求不存在的 key，导致每次都查数据库
public Product getProduct(Long id) {
    String key = "PRODUCT:" + id;
    Product product = redisTemplate.opsForValue().get(key);
    if (product == null) {
        product = productDao.selectById(id);  // 不存在的数据
        if (product == null) {
            return null;  // 下次请求仍然会查数据库
        }
        redisTemplate.opsForValue().set(key, product, 30, TimeUnit.MINUTES);
    }
    return product;
}

// ✅ 缓存空值
public Product getProduct(Long id) {
    String key = "PRODUCT:" + id;
    Product product = redisTemplate.opsForValue().get(key);
    if (product == null) {
        product = productDao.selectById(id);
        if (product == null) {
            // 缓存空值，防止缓存穿透
            redisTemplate.opsForValue().set(key, NULL_VALUE, 5, TimeUnit.MINUTES);
            return null;
        }
        redisTemplate.opsForValue().set(key, product, 30, TimeUnit.MINUTES);
    }
    return product == NULL_VALUE ? null : product;
}

// 或使用布隆过滤器
@Autowired
private BloomFilter<Long> bloomFilter;

public Product getProduct(Long id) {
    // 先用布隆过滤器判断
    if (!bloomFilter.mightContain(id)) {
        return null;  // 一定不存在
    }

    String key = "PRODUCT:" + id;
    Product product = redisTemplate.opsForValue().get(key);
    if (product == null) {
        product = productDao.selectById(id);
        if (product != null) {
            redisTemplate.opsForValue().set(key, product, 30, TimeUnit.MINUTES);
        }
    }
    return product;
}
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| 锁在事务提交前释放 | 并发数据错误 | 🚨 严重 |
| 分布式锁无超时 | 死锁风险 | 🚨 严重 |
| 无锁续期机制 | 锁过期导致并发 | 🚨 严重 |
| 事务与远程调用混用 | 数据不一致 | 🚨 严重 |
| 更新缓存而非删除 | 脏数据 | ⚠️ 中等 |
| 缓存穿透 | 数据库压力 | ⚠️ 中等 |
