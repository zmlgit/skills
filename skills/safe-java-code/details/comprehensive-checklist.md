# 综合检查清单 (Comprehensive Checklist)

## 完整代码审查检查表

在进行完整的代码审查时，请按以下顺序检查各项：

## 第一阶段：并发安全

- [ ] **同步块检查**
  - [ ] 是否有无同步的共享可变状态？
  - [ ] 锁粒度是否过大？
  - [ ] 是否存在锁顺序问题（可能导致死锁）？
  - [ ] ReentrantLock 是否设置了超时？

- [ ] **原子操作**
  - [ ] volatile 是否用于复合操作？（需要 AtomicInteger）
  - [ ] long/double 是否使用 volatile？（64位写入）
  - [ ] 是否错误使用 HashMap？（应使用 ConcurrentHashMap）

- [ ] **ThreadLocal**
  - [ ] ThreadLocal 是否在 finally 块中清理？
  - [ ] 在线程池环境下是否正确清理？

- [ ] **集合操作**
  - [ ] ConcurrentHashMap 是否正确使用？（避免复合操作）
  - [ ] 迭代时是否修改集合？

## 第二阶段：竞态条件

- [ ] **Check-Then-Act 模式**
  - [ ] 是否存在 if-contains-then-put 模式？
  - [ ] 是否使用 map.computeIfAbsent() 替代？
  - [ ] 单例模式是否正确实现？（需要 volatile）

- [ ] **可变状态**
  - [ ] 是否返回了内部可变对象的引用？
  - [ ] 构造函数中 this 是否逃逸？
  - [ ] 日期格式化是否被共享使用？（应使用 ThreadLocal 或 DateTimeFormatter）

- [ ] **不安全发布**
  - [ ] 对象是否安全发布？（final、volatile、安全初始化）

## 第三阶段：数据库安全

- [ ] **更新结果检查**
  - [ ] save/update 操作是否检查了返回值？
  - [ ] 批量操作是否检查了影响的行数？
  - [ ] JDBC executeUpdate 是否检查返回值？

- [ ] **乐观锁**
  - [ ] 实体是否有 @Version 字段？
  - [ ] 是否处理了 ObjectOptimisticLockingFailureException？

- [ ] **事务边界**
  - [ ] 事务回滚后是否正确处理了状态？
  - [ ] 是否存在大事务？
  - [ ] @Transactional 事件处理是否正确？

- [ ] **N+1 查询**
  - [ ] 是否存在循环中的数据库查询？
  - [ ] 是否使用 JOIN FETCH 或 @EntityGraph？

## 第四阶段：异步执行

- [ ] **线程池**
  - [ ] 线程池是否正确关闭（@PreDestroy）？
  - [ ] 是否使用无界队列？（应使用有界队列）
  - [ ] 是否配置了拒绝策略？

- [ ] **异常处理**
  - [ ] Future.get() 的异常是否被处理？
  - [ ] @Async 方法的异常是否被捕获？
  - [ ] CompletableFuture 是否配置了异常处理？

- [ ] **并发控制**
  - [ ] 是否有无限并发创建的问题？
  - [ ] 是否使用信号量限制并发？

- [ ] **超时控制**
  - [ ] 异步操作是否设置了超时？
  - [ ] 是否使用 orTimeout() 或 completeOnTimeout()？

- [ ] **上下文传递**
  - [ ] 异步线程中是否丢失了 UserContext？
  - [ ] MDC 跟踪 ID 是否正确传递？

## 第五阶段：资源安全

- [ ] **资源关闭**
  - [ ] 是否使用 try-with-resources？
  - [ ] finally 块中的 close() 是否可能抛出异常？

- [ ] **资源泄漏**
  - [ ] 连接是否正确关闭？
  - [ ] 文件句柄是否正确释放？
  - [ ] InputStream/OutputStream 是否在 finally 中关闭？

## 第六阶段：安全漏洞

- [ ] **SQL 注入**
  - [ ] 是否使用字符串拼接 SQL？
  - [ ] LIKE 查询是否正确转义？

- [ ] **路径遍历**
  - [ ] 文件路径是否规范化？
  - [ ] 是否验证路径在基础路径内？

- [ ] **命令注入**
  - [ ] Runtime.exec() 是否使用用户输入？
  - [ ] 是否验证参数白名单？

- [ ] **敏感信息**
  - [ ] 日志中是否包含密码/密钥？
  - [ ] 异常信息是否暴露敏感数据？
  - [ ] toString() 是否包含敏感字段？

## 第七阶段：代码质量

- [ ] **空指针安全**
  - [ ] Optional.get() 前是否检查 isPresent()？
  - [ ] 是否使用 Optional.orElse() 处理空值？

- [ ] **异常处理**
  - [ ] 是否捕获了过于宽泛的 Exception？
  - [ ] 异常是否被静默吞没？
  - [ ] 是否正确处理了受检异常？

- [ ] **类型安全**
  - [ ] 是否使用 @SuppressWarnings("unchecked")？
  - [ ] 泛型是否正确使用？

## 输出模板

完成审查后，按以下格式输出：

```markdown
## 代码审查报告

### 🚨 严重风险 (必须修复)

1. **并发安全 - 无锁共享状态**
   - **位置**: `UserService.java:45`
   - **问题**: `userCache` 使用 HashMap 在多线程环境下不安全
   - **风险**: 可能导致无限循环或数据丢失
   - **修复**:
   ```java
   private final Map<Long, User> userCache = new ConcurrentHashMap<>();
   ```

2. **数据库安全 - 更新未检查**
   - **位置**: `OrderService.java:78`
   - **问题**: `userRepository.save()` 未检查返回值
   - **风险**: 更新失败但代码认为成功
   - **修复**:
   ```java
   User saved = userRepository.save(user);
   if (saved == null || saved.getId() == null) {
       throw new IllegalStateException("保存失败");
   }
   ```

### ⚠️ 潜在隐患 (建议修复)

1. **资源安全 - 流未正确关闭**
   - **位置**: `FileService.java:23`
   - **建议**: 使用 try-with-resources 确保流被关闭

2. **性能 - N+1 查询**
   - **位置**: `OrderRepository.java:15`
   - **建议**: 使用 @EntityGraph 或 JOIN FETCH

### ✅ 最佳实践

1. 正确使用 @Version 实现乐观锁
2. 异常处理得当，错误信息清晰
```
