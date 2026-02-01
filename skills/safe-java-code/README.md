# Safe Java Code - Java 代码安全检查 Skill

## 简介

这是一个用于代码审查的 Java 安全检查 Skill，帮助识别 Java 代码中的常见安全漏洞、并发问题、分布式系统风险和基础代码问题。

**适用场景**：通用 Java 应用 + 金融/高负载分布式系统

## 功能特性

### 关键检查维度

1. **分布式系统** (`details/distributed-systems.md`)
   - 分布式锁与事务顺序（锁在事务提交前释放）
   - 分布式锁超时与续期（看门狗机制）
   - SAGA 模式与补偿事务
   - 缓存一致性（Cache-Aside 模式）
   - 缓存穿透防护

2. **并发安全** (`details/concurrency.md`)
   - 同步锁使用与锁粒度
   - 锁顺序（死锁风险）
   - volatile 变量误用
   - ThreadLocal 内存泄漏
   - ConcurrentHashMap 误用

3. **竞态条件** (`details/race-conditions.md`)
   - Check-Then-Act 模式
   - 双重检查锁定
   - 不可变对象误用
   - 不安全的对象发布
   - 非线程安全对象共享

4. **数据库安全** (`details/database-safety.md`)
   - 更新结果未检查
   - 批量操作未验证
   - 乐观锁处理
   - 事务回滚处理
   - N+1 查询问题

5. **异步执行** (`details/async-execution.md`)
   - 线程池配置与关闭
   - 异常处理
   - 并发限制
   - 超时控制
   - 上下文传递

6. **常见漏洞** (`details/common-vulnerabilities.md`)
   - 资源泄漏
   - SQL 注入
   - 路径遍历
   - 输入验证缺失
   - IDOR（越权访问）
   - 命令注入
   - 敏感信息泄露

7. **基础代码问题** (`details/basic-code-issues.md`) ⭐ 新增
   - 空指针风险（NPE）
   - 数组/集合越界
   - 异常未记录/被吞掉
   - 数值溢出与精度问题
   - 字符串处理问题

## 使用方式

### 自动触发配置

Skill 配置了以下自动触发条件：

- **文件模式**: `**/service/**/*Service*.java`, `**/repository/**/*.java`, `**/controller/**/*.java`
- **代码触发**: `synchronized`, `ReentrantLock`, `@Transactional`, `update`, `ExecutorService`, `redisLock`, `FeignClient`, `.get(`, `catch`, `printStackTrace` 等

### 进阶加载

支持渐进式加载，根据代码内容动态加载相关检查模块：

```yaml
details/distributed-systems.md    # 检测到分布式锁、RPC调用时加载
details/concurrency.md            # 检测到并发关键字时加载
details/race-conditions.md        # 检测到竞态条件模式时加载
details/database-safety.md        # 检测到数据库操作时加载
details/async-execution.md        # 检测到异步执行时加载
details/basic-code-issues.md      # 检测到 .get(, catch, null 等时加载
```

## 输出格式

审查结果按严重程度分类输出：

- 🚨 **严重风险** - 直接威胁系统安全或数据完整性
- ⚠️ **潜在隐患** - 可能导致问题的代码模式
- ✅ **最佳实践** - 正确实现的模式

## 目录结构

```
safe-java-code/
├── SKILL.md                          # 核心检查逻辑
├── skill.yaml                        # Skill 配置文件
├── README.md                         # 本文件
├── details/                          # 详细检查模块
│   ├── distributed-systems.md        # 分布式系统安全
│   ├── concurrency.md                # 并发安全
│   ├── race-conditions.md            # 竞态条件
│   ├── database-safety.md            # 数据库安全
│   ├── async-execution.md            # 异步执行
│   ├── common-vulnerabilities.md     # 常见漏洞
│   ├── basic-code-issues.md          # 基础代码问题（新增）
│   └── comprehensive-checklist.md    # 综合检查清单
└── references/                       # 参考资源（可选）
```

## 与其他 Skill 的关系

- **spring-aop-prompt**: 专注于 Spring AOP 代理问题
- **safe-java-code**: 通用 Java 代码安全检查（包含分布式系统 + 基础问题）

本 Skill 可以与其他 Skill 配合使用，提供更全面的代码审查。

## 示例

### 分布式锁与事务顺序（严重问题）

```java
// ❌ 错误：锁在事务提交前释放
@Transactional
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

// ✅ 正确：编程式事务
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

### 空指针风险（常见问题）

```java
// ❌ 错误：未检查直接使用
public void processUser(User user) {
    String name = user.getName();  // NPE 风险
}

// ❌ 链式调用未检查
public String getCity(Order order) {
    return order.getUser().getAddress().getCity();  // 多重 NPE 风险
}

// ❌ 自动拆箱未检查
public int getAge(Long userId) {
    User user = userRepository.findById(userId);
    return user.getAge();  // getAge() 返回 Integer，可能为 null
}

// ✅ 正确：使用 Optional
public void processUser(User user) {
    Optional.ofNullable(user)
        .map(User::getName)
        .ifPresent(System.out::println);
}

// ✅ 正确：链式调用使用 Optional
public String getCity(Order order) {
    return Optional.ofNullable(order)
        .map(Order::getUser)
        .map(User::getAddress)
        .map(Address::getCity)
        .orElse("Unknown");
}

// ✅ 正确：自动拆箱前检查
public int getAge(Long userId) {
    User user = userRepository.findById(userId);
    if (user == null || user.getAge() == null) {
        return 0;
    }
    return user.getAge();
}
```

### 数组越界风险

```java
// ❌ 错误：未检查边界
public String getItem(String[] array, int index) {
    return array[index];  // IndexOutOfBoundsException
}

// ❌ 字符串截取未检查
public String getExtension(String filename) {
    int dotIndex = filename.lastIndexOf('.');
    return filename.substring(dotIndex);  // dotIndex 可能为 -1
}

// ❌ 空集合未检查
public String getFirst(List<String> list) {
    return list.get(0);  // 如果 list 为空，抛出异常
}

// ✅ 正确：检查边界
public Optional<String> getItem(String[] array, int index) {
    if (array != null && index >= 0 && index < array.length) {
        return Optional.of(array[index]);
    }
    return Optional.empty();
}

// ✅ 正确：字符串截取检查
public String getExtension(String filename) {
    if (filename == null || filename.isEmpty()) {
        return "";
    }
    int dotIndex = filename.lastIndexOf('.');
    if (dotIndex <= 0) {
        return "";
    }
    return filename.substring(dotIndex);
}

// ✅ 正确：检查集合是否为空
public Optional<String> getFirst(List<String> list) {
    if (list == null || list.isEmpty()) {
        return Optional.empty();
    }
    return Optional.of(list.get(0));
}
```

### 异常未记录

```java
// ❌ 错误：吞掉异常
public void processData(String data) {
    try {
        JSONObject json = JSON.parseObject(data);
    } catch (Exception e) {
        // 什么都不做，问题难以排查
    }
}

// ❌ 错误：只打印，不记录日志
public void loadConfig(String path) {
    try {
        config = loadFromFile(path);
    } catch (IOException e) {
        e.printStackTrace();  // 不应该使用 printStackTrace
    }
}

// ❌ 错误：异常信息丢失
public User getUser(Long id) {
    try {
        return userRepository.findById(id);
    } catch (Exception e) {
        throw new BusinessException("查询用户失败");  // 未记录原始异常
    }
}

// ✅ 正确：记录异常日志
public void processData(String data) {
    try {
        JSONObject json = JSON.parseObject(data);
    } catch (JSONException e) {
        log.error("JSON 解析失败, data: {}", data, e);
        throw new InvalidDataException("JSON 格式错误", e);
    }
}

// ✅ 正确：使用 SLF4J 记录
public void loadConfig(String path) {
    try {
        config = loadFromFile(path);
    } catch (IOException e) {
        log.error("加载配置文件失败, path: {}", path, e);
        throw new ConfigException("配置加载失败", e);
    }
}

// ✅ 正确：保留原始异常
public User getUser(Long id) {
    try {
        return userRepository.findById(id);
    } catch (Exception e) {
        log.error("查询用户失败, userId: {}", id, e);
        throw new BusinessException("查询用户失败: " + id, e);
    }
}
```

## 版本历史

- **1.0.0** - 初始版本，支持并发、竞态条件、数据库、异步执行等检查
- **1.1.0** - 合并 java-transaction-checker，新增分布式系统安全检查（锁与事务顺序、SAGA、缓存一致性）、输入验证、IDOR 检测
- **1.2.0** - 新增基础代码问题检查：空指针风险、数组越界、异常未记录、数值溢出、字符串处理问题
