# 数据库安全 (Database Safety)

## 检查清单

### 1. 更新结果未检查

#### ❌ 错误模式 - 未检查更新是否成功

```java
public void updateUserBalance(Long userId, BigDecimal amount) {
    User user = userRepository.findById(userId).orElseThrow();
    user.setBalance(user.getBalance().add(amount));

    // ❌ 没有检查更新是否实际发生！
    userRepository.save(user);

    // 如果更新失败（数据库约束、版本冲突等），代码不会知道
    log.info("用户余额更新成功");  // 实际上可能失败了
}
```

#### ✅ 正确模式 - 检查更新结果

```java
public void updateUserBalance(Long userId, BigDecimal amount) {
    User user = userRepository.findById(userId).orElseThrow();
    user.setBalance(user.getBalance().add(amount));

    User saved = userRepository.save(user);

    // 检查保存是否成功
    if (saved == null || saved.getId() == null) {
        throw new IllegalStateException("用户余额更新失败");
    }

    // 或使用 JPA 的 update 返回值
    int updated = userRepository.updateBalance(userId, amount);
    if (updated == 0) {
        throw new IllegalStateException("用户不存在或更新失败");
    }

    log.info("用户余额更新成功，更新行数: {}", updated);
}
```

### 2. 批量更新未检查影响行数

#### ❌ 错误模式

```java
public void batchUpdateStatus(List<Long> ids, Status newStatus) {
    // ❌ 没有检查实际更新了多少行
    jdbcTemplate.update(
        "UPDATE orders SET status = ? WHERE id IN (?)",
        newStatus, ids
    );

    log.info("批量更新完成");
}
```

#### ✅ 正确模式

```java
public void batchUpdateStatus(List<Long> ids, Status newStatus) {
    int[] results = jdbcTemplate.batchUpdate(
        "UPDATE orders SET status = ? WHERE id = ?",
        new BatchPreparedStatementSetter() {
            // ...
        }
    );

    int totalUpdated = Arrays.stream(results).sum();
    int expected = ids.size();

    if (totalUpdated != expected) {
        log.warn("预期更新 {} 条，实际更新 {} 条", expected, totalUpdated);
        // 根据业务需求决定是否抛出异常
    }

    log.info("批量更新完成，影响行数: {}", totalUpdated);
}
```

### 3. 乐观锁未处理版本冲突

#### ❌ 错误模式

```java
@Entity
public class Product {
    @Id
    private Long id;
    private Integer stock;

    // ❌ 没有 @Version
}

public void reduceStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId).orElseThrow();

    if (product.getStock() < quantity) {
        throw new InsufficientStockException();
    }

    product.setStock(product.getStock() - quantity);
    productRepository.save(product);
    // 如果两个请求同时读取到相同的 stock 值，会导致超卖
}
```

#### ✅ 正确模式

```java
@Entity
public class Product {
    @Id
    private Long id;
    private Integer stock;

    @Version  // ✅ 添加版本字段
    private Long version;
}

public void reduceStock(Long productId, int quantity) {
    try {
        int updated = productRepository.reduceStock(productId, quantity);
        if (updated == 0) {
            throw new OptimisticLockException("库存更新失败，请重试");
        }
    } catch (ObjectOptimisticLockingFailureException e) {
        throw new OptimisticLockException("版本冲突，请重试", e);
    }
}

// Repository 方法
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :quantity WHERE p.id = :productId AND p.stock >= :quantity")
int reduceStock(@Param("productId") Long productId, @Param("quantity") int quantity);
```

### 4. 事务回滚后未处理状态

#### ❌ 错误模式

```java
@Transactional
public void processOrder(Order order) {
    try {
        // 处理订单
        orderService.create(order);

        // 发送通知
        notificationService.send(order);

    } catch (Exception e) {
        // ❌ 事务回滚了，但没有处理后续逻辑
        log.error("处理订单失败", e);

        // 这里的问题：
        // 1. 事务已回滚，但通知可能已发送
        // 2. 外部系统状态与本地数据库不一致
    }
}
```

#### ✅ 正确模式

```java
// 方式1：使用 @TransactionalEventListener
@Transactional(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderCreated(OrderCreatedEvent event) {
    // 只有在事务成功提交后才发送通知
    notificationService.send(event.getOrder());
}

// 方式2：使用编程式事务
public void processOrder(Order order) {
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

    try {
        orderService.create(order);
        transactionManager.commit(status);

        // 只有事务成功后才发送通知
        notificationService.send(order);

    } catch (Exception e) {
        transactionManager.rollback(status);
        log.error("处理订单失败，事务已回滚", e);

        // 明确不发送通知，或发送失败通知
        notificationService.sendFailure(order, e.getMessage());
    }
}
```

### 5. 查询后更新（Lost Update 问题）

#### ❌ 错误模式

```java
public void incrementCounter(Long id) {
    Counter counter = counterRepository.findById(id).orElseThrow();

    // ❌ 读取-修改-写入模式，竞态条件
    counter.setValue(counter.getValue() + 1);

    counterRepository.save(counter);
}
```

#### ✅ 正确模式

```java
// 方式1：使用数据库原子操作
@Modifying
@Query("UPDATE Counter c SET c.value = c.value + 1 WHERE c.id = :id")
int increment(@Param("id") Long id);

// 方式2：使用锁
@Transactional
public void incrementCounter(Long id) {
    Counter counter = counterRepository.findByIdWithLock(id).orElseThrow();
    counter.setValue(counter.getValue() + 1);
    counterRepository.save(counter);
}

// 方式3：使用乐观锁重试
@Retryable(value = ObjectOptimisticLockingFailureException.class, maxAttempts = 3)
public void incrementCounter(Long id) {
    Counter counter = counterRepository.findById(id).orElseThrow();
    counter.setValue(counter.getValue() + 1);
    counterRepository.save(counter);
}
```

### 6. N+1 查询问题

#### ❌ 错误模式

```java
public List<OrderDTO> getUserOrders(Long userId) {
    List<Order> orders = orderRepository.findByUserId(userId);

    // ❌ N+1 问题：对每个订单都执行一次查询
    return orders.stream()
        .map(order -> {
            OrderDTO dto = new OrderDTO();
            dto.setOrder(order);

            // 额外查询！
            dto.setItems(orderItemRepository.findByOrderId(order.getId()));

            return dto;
        })
        .collect(Collectors.toList());
}
```

#### ✅ 正确模式

```java
// 使用 JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.user.id = :userId")
List<Order> findByUserIdWithItems(@Param("userId") Long userId);

// 或使用 @EntityGraph
@EntityGraph(attributePaths = {"items"})
List<Order> findByUserId(Long userId);

// 或使用批量加载
@Query("SELECT o FROM Order o WHERE o.user.id = :userId")
List<Order> findByUserId(@Param("userId") Long userId);

@Query("SELECT i FROM OrderItem i WHERE i.order.id IN :orderIds")
@Param("orderIds")
List<OrderItem> findByOrderIds(Set<Long> orderIds);
```

### 7. 大事务问题

#### ❌ 错误模式

```java
@Transactional  // ❌ 事务时间过长
public void processLargeBatch(List<Order> orders) {
    for (Order order : orders) {
        // 复杂处理
        processOrder(order);

        // 调用外部服务
        externalService.validate(order);

        // 发送邮件
        emailService.sendConfirmation(order);
    }
}
```

#### ✅ 正确模式

```java
// 方式1：缩小事务范围
public void processLargeBatch(List<Order> orders) {
    for (Order order : orders) {
        // 每个订单独立事务
        processOrderInTransaction(order);
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void processOrderInTransaction(Order order) {
    // 只在这个事务中做必要的数据操作
    orderRepository.save(order);
}

// 方式2：使用批处理
@Transactional
public void batchInsert(List<Order> orders) {
    jdbcTemplate.batchUpdate(
        "INSERT INTO orders (user_id, amount) VALUES (?, ?)",
        new BatchPreparedStatementSetter() {
            // ...
        }
    );
}
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| 更新/删除未检查返回值 | 数据丢失、状态不一致 | 🚨 严重 |
| 批量操作未检查影响行数 | 数据部分丢失 | 🚨 严重 |
| 乐观锁未处理版本冲突 | 数据覆盖、超卖 | 🚨 严重 |
| 事务回滚后状态未处理 | 数据不一致 | 🚨 严重 |
| 查询后更新（无锁） | 丢失更新 | 🚨 严重 |
| N+1 查询 | 性能问题 | ⚠️ 中等 |
| 大事务 | 性能问题、死锁 | ⚠️ 中等 |
