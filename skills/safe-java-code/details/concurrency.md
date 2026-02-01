# 并发安全 (Concurrency Safety)

## 检查清单

### 0. 分布式锁与事务顺序 ⭐ 优先检查

#### ❌ 错误模式 - 锁在事务提交前释放

```java
// ❌ @Transactional 在方法上，锁在 try-finally 中
@Transactional(rollbackFor = Exception.class)
public void decreaseAsset(Long id, Long delta) {
    String lockKey = "LOCK:ASSET:" + id;
    redisLock.lock(lockKey);
    try {
        Account account = accountDao.selectById(id);
        account.setBalance(account.getBalance() - delta);
        accountDao.updateById(account);
    } finally {
        redisLock.unlock(lockKey);  // 事务还未提交就释放锁！
    }
}
// 问题：锁释放后，其他线程可以获取锁并读取到未提交的数据
```

#### ✅ 正确模式 - 编程式事务

```java
// 使用编程式事务，确保事务在锁释放前提交
public void decreaseAsset(Long id, Long delta) {
    String lockKey = "LOCK:ASSET:" + id;
    redisLock.lock(lockKey);
    try {
        transactionTemplate.execute(status -> {
            Account account = accountDao.selectById(id);
            account.setBalance(account.getBalance() - delta);
            accountDao.updateById(account);
            return null;
        });  // 事务在这里提交
    } finally {
        redisLock.unlock(lockKey);  // 然后释放锁
    }
}
```

### 1. 同步锁 (Synchronized)

### 1. 同步锁 (Synchronized)

#### ❌ 错误模式

```java
// 锁粒度过大 - 整个方法被锁定
public synchronized void processOrder(Order order) {
    // 验证 - 需要锁
    validateOrder(order);

    // 调用外部服务 - 不需要锁，但持有锁时间过长
    externalService.notify(order);

    // 保存 - 需要锁
    repository.save(order);
}
```

#### ✅ 正确模式

```java
// 减小锁粒度，只锁定必要的代码块
public void processOrder(Order order) {
    validateOrder(order);
    externalService.notify(order);  // 不需要锁

    // 只锁定临界区
    synchronized(lock) {
        repository.save(order);
    }
}
```

### 2. 锁顺序 (Lock Ordering)

#### ❌ 错误模式 - 死锁风险

```java
// Thread 1
synchronized(lockA) {
    synchronized(lockB) {
        // ...
    }
}

// Thread 2 - 不同的锁顺序！
synchronized(lockB) {
    synchronized(lockA) {
        // ...
    }
}
```

#### ✅ 正确模式

```java
// 所有线程按相同顺序获取锁
private final Object[] locks = new Object[]{lockA, lockB};

// 总是按固定顺序获取
synchronized(locks[0]) {
    synchronized(locks[1]) {
        // ...
    }
}
```

### 3. ReentrantLock 使用

#### ❌ 错误模式

```java
private final ReentrantLock lock = new ReentrantLock();

public void riskyMethod() {
    // 没有超时 - 可能永久阻塞
    lock.lock();
    try {
        // 操作
    } finally {
        lock.unlock();
    }
}
```

#### ✅ 正确模式

```java
public void saferMethod() throws InterruptedException {
    // 带超时的锁尝试
    if (lock.tryLock(5, TimeUnit.SECONDS)) {
        try {
            // 操作
        } finally {
            lock.unlock();
        }
    } else {
        throw new TimeoutException("获取锁超时");
    }
}
```

### 4. volatile 变量

#### ⚠️ 常见误解

```java
// ❌ 错误：volatile 不保证原子性
private volatile int counter = 0;

public void increment() {
    counter++;  // 非原子操作！读取-修改-写入
}

// ✅ 正确：使用原子类
private final AtomicInteger counter = new AtomicInteger(0);

public void increment() {
    counter.incrementAndGet();  // 原子操作
}
```

### 5. ThreadLocal 使用

#### ❌ 错误模式 - 内存泄漏

```java
// 在线程池中使用 ThreadLocal
private static final ThreadLocal<UserContext> context = new ThreadLocal<>();

public void processRequest() {
    context.set(new UserContext());
    // ...
    // 忘记清理！线程被放回池中后，UserContext 仍然存在
}
```

#### ✅ 正确模式

```java
public void processRequest() {
    try {
        context.set(new UserContext());
        // ...
    } finally {
        context.remove();  // 必须清理
    }
}
```

### 6. ConcurrentHashMap 误用

#### ❌ 错误模式 - 复合操作非原子

```java
// check-then-act 竞态条件
if (!concurrentMap.containsKey(key)) {
    concurrentMap.put(key, computeValue());  // 可能重复计算
}
```

#### ✅ 正确模式

```java
// 使用原子操作
concurrentMap.computeIfAbsent(key, k -> computeValue());

// 或者
concurrentMap.putIfAbsent(key, value);
```

### 7. 集合的线程安全

#### ❌ 错误模式

```java
// Collections.synchronizedList 只保证单个操作线程安全
List<String> list = Collections.synchronizedList(new ArrayList<>());

// 迭代时仍需同步
for (String item : list) {  // ConcurrentModificationException 风险
    // ...
}
```

#### ✅ 正确模式

```java
// 迭代时手动同步
synchronized(list) {
    for (String item : list) {
        // ...
    }
}

// 或使用 CopyOnWriteArrayList
List<String> list = new CopyOnWriteArrayList<>();
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| 无锁共享可变状态 | 数据竞态、数据损坏 | 🚨 严重 |
| 锁顺序不一致 | 死锁 | 🚨 严重 |
| synchronized 范围过大 | 性能问题 | ⚠️ 中等 |
| ReentrantLock 无超时 | 死锁风险 | 🚨 严重 |
| volatile 用于复合操作 | 数据竞态 | 🚨 严重 |
| ThreadLocal 未清理 | 内存泄漏 | ⚠️ 中等 |
| HashMap 多线程使用 | 无限循环、数据丢失 | 🚨 严重 |
| ConcurrentHashMap 复合操作 | 竞态条件 | ⚠️ 中等 |
