# 基础代码问题 (Basic Code Issues)

## 检查清单

### 1. 空指针风险 (NullPointerException)

#### ❌ 错误模式

```java
// ❌ 未检查直接使用
public void processUser(User user) {
    String name = user.getName();  // NPE 风险
    System.out.println(name);
}

// ❌ 链式调用未检查
public void printOrderAddress(Order order) {
    String city = order.getUser().getAddress().getCity();  // 多重 NPE 风险
    System.out.println(city);
}

// ❌ 返回值未检查
public User findUser(String id) {
    return userRepository.findById(id);  // 可能返回 null
}

// 调用方
User user = findUser("123");
String name = user.getName();  // NPE

// ❌ 自动拆箱 NPE
public int getAge(Long userId) {
    User user = userRepository.findById(userId);
    return user.getAge();  // getAge() 返回 Integer，可能为 null
}

// ❌ Map.get() 返回 null
public void processValue(Map<String, String> map, String key) {
    String value = map.get(key);  // key 不存在时返回 null
    System.out.println(value.length());  // NPE
}

// ❌ 字符串比较
public boolean checkName(String name) {
    return name.equals("admin");  // name 为 null 时 NPE
}

// ❌ 集合元素未检查
public List<String> getNames(List<User> users) {
    return users.stream()
        .map(User::getName)  // getName() 可能返回 null
        .collect(Collectors.toList());
}
```

#### ✅ 正确模式

```java
// 方式1：参数校验
public void processUser(User user) {
    if (user == null) {
        throw new IllegalArgumentException("user cannot be null");
    }
    String name = user.getName();
    System.out.println(name);
}

// 方式2：使用 Optional
public void processUser(User user) {
    Optional.ofNullable(user)
        .map(User::getName)
        .ifPresent(System.out::println);
}

// 方式3：工具类
public void processUser(User user) {
    String name = Optional.ofNullable(user)
        .map(User::getName)
        .orElse("Unknown");
    System.out.println(name);
}

// 链式调用使用 Optional
public String printOrderAddress(Order order) {
    return Optional.ofNullable(order)
        .map(Order::getUser)
        .map(User::getAddress)
        .map(Address::getCity)
        .orElse("Unknown");
}

// 返回 Optional 而非 null
public Optional<User> findUser(String id) {
    return userRepository.findById(id);
}

// 或使用注解
public @Nullable User findUser(String id) {
    return userRepository.findById(id);
}

// 使用 @NonNull 注解
public void processUser(@NonNull User user) {
    String name = user.getName();
    System.out.println(name);
}

// 自动拆箱前检查
public int getAge(Long userId) {
    User user = userRepository.findById(userId);
    if (user == null || user.getAge() == null) {
        return 0;  // 或抛出异常
    }
    return user.getAge();
}

// Map.getOrDefault()
public void processValue(Map<String, String> map, String key) {
    String value = map.getOrDefault(key, "");
    System.out.println(value.length());
}

// 或使用 Optional
public void processValue(Map<String, String> map, String key) {
    Optional.ofNullable(map.get(key))
        .ifPresent(value -> System.out.println(value.length()));
}

// 字符串比较常量在前
public boolean checkName(String name) {
    return "admin".equals(name);  // 安全
}

// 或 Objects.equals()
public boolean checkName(String name) {
    return Objects.equals(name, "admin");
}

// 集合元素过滤
public List<String> getNames(List<User> users) {
    return users.stream()
        .map(User::getName)
        .filter(Objects::nonNull)  // 过滤 null
        .collect(Collectors.toList());
}
```

### 2. 数组/集合越界

#### ❌ 错误模式

```java
// ❌ 数组访问未检查边界
public String getItem(String[] array, int index) {
    return array[index];  // IndexOutOfBoundsException
}

// ❌ List.subtring() 未检查
public List<String> getSubList(List<String> list, int from, int to) {
    return list.subList(from, to);  // IndexOutOfBoundsException
}

// ❌ 字符串截取未检查
public String getExtension(String filename) {
    int dotIndex = filename.lastIndexOf('.');
    return filename.substring(dotIndex);  // dotIndex 可能为 -1
}

// ❌ 循环边界错误
public void processArray(int[] array) {
    for (int i = 0; i <= array.length; i++) {  // 应该是 <
        System.out.println(array[i]);  // 最后一次越界
    }
}

// ❌ 空集合未检查
public String getFirst(List<String> list) {
    return list.get(0);  // 如果 list 为空，抛出 IndexOutOfBoundsException
}

// ❌ 字符数组转字符串未检查
public char getLastChar(String str) {
    return str.charAt(str.length() - 1);  // 空字符串时越界
}
```

#### ✅ 正确模式

```java
// 检查数组边界
public String getItem(String[] array, int index) {
    if (array == null) {
        throw new IllegalArgumentException("array cannot be null");
    }
    if (index < 0 || index >= array.length) {
        throw new IndexOutOfBoundsException("index: " + index + ", length: " + array.length);
    }
    return array[index];
}

// 使用 Optional
public Optional<String> getItem(String[] array, int index) {
    if (array != null && index >= 0 && index < array.length) {
        return Optional.of(array[index]);
    }
    return Optional.empty();
}

// 检查 subList 边界
public List<String> getSubList(List<String> list, int from, int to) {
    if (list == null || list.isEmpty()) {
        return Collections.emptyList();
    }
    if (from < 0 || to > list.size() || from > to) {
        throw new IndexOutOfBoundsException("from: " + from + ", to: " + to + ", size: " + list.size());
    }
    return list.subList(from, to);
}

// 字符串截取检查
public String getExtension(String filename) {
    if (filename == null || filename.isEmpty()) {
        return "";
    }
    int dotIndex = filename.lastIndexOf('.');
    if (dotIndex <= 0) {  // 点在开头或不存在
        return "";
    }
    return filename.substring(dotIndex);
}

// 正确的循环边界
public void processArray(int[] array) {
    for (int i = 0; i < array.length; i++) {
        System.out.println(array[i]);
    }
}

// 或使用 for-each
public void processArray(int[] array) {
    for (int item : array) {
        System.out.println(item);
    }
}

// 检查集合是否为空
public Optional<String> getFirst(List<String> list) {
    if (list == null || list.isEmpty()) {
        return Optional.empty();
    }
    return Optional.of(list.get(0));
}

// 字符串长度检查
public Optional<Character> getLastChar(String str) {
    if (str == null || str.isEmpty()) {
        return Optional.empty();
    }
    return Optional.of(str.charAt(str.length() - 1));
}
```

### 3. 异常未被记录/处理

#### ❌ 错误模式

```java
// ❌ 吞掉异常，什么都不做
public void processData(String data) {
    try {
        JSONObject json = JSON.parseObject(data);
        // 处理
    } catch (Exception e) {
        // 什么都不做，问题难以排查
    }
}

// ❌ 只打印，不记录日志
public void loadConfig(String path) {
    try {
        config = loadFromFile(path);
    } catch (IOException e) {
        e.printStackTrace();  // 不应该使用 printStackTrace
    }
}

// ❌ 捕获过于宽泛
public void process() {
    try {
        // 业务逻辑
    } catch (Exception e) {  // 捕获所有异常
        // 可能掩盖严重错误
    }
}

// ❌ 异常信息丢失
public void process() {
    try {
        doSomething();
    } catch (SQLException e) {
        throw new RuntimeException("处理失败");  // 丢失了原始异常信息
    }
}

// ❌ 捕获后立即抛出新异常，未记录
public User getUser(Long id) {
    try {
        return userRepository.findById(id);
    } catch (Exception e) {
        throw new BusinessException("查询用户失败");  // 未记录原始异常
    }
}

// ❌ finally 中抛出异常，掩盖 try 中的异常
public void process() {
    try {
        doSomething();
    } finally {
        closeResource();  // 如果这里抛异常，try 中的异常就丢失了
    }
}

// ❌ 多个 catch 只打印一条日志
public void process() {
    try {
        doSomething();
    } catch (IOException e) {
        log.error("IO 异常");
    } catch (SQLException e) {
        log.error("SQL 异常");
    }
}
```

#### ✅ 正确模式

```java
// 记录异常日志
public void processData(String data) {
    try {
        JSONObject json = JSON.parseObject(data);
        // 处理
    } catch (JSONException e) {
        log.error("JSON 解析失败, data: {}", data, e);  // 记录原始异常和数据
        throw new InvalidDataException("JSON 格式错误", e);
    }
}

// 使用 SLF4J 记录
public void loadConfig(String path) {
    try {
        config = loadFromFile(path);
    } catch (IOException e) {
        log.error("加载配置文件失败, path: {}", path, e);
        throw new ConfigException("配置加载失败", e);
    }
}

// 捕获具体异常
public void process() {
    try {
        // 业务逻辑
    } catch (IOException e) {
        log.error("IO 错误", e);
        // 处理 IO 异常
    } catch (NumberFormatException e) {
        log.error("数字格式错误", e);
        // 处理格式错误
    }
}

// 保留原始异常（异常链）
public void process() {
    try {
        doSomething();
    } catch (SQLException e) {
        log.error("数据库操作失败", e);
        throw new RuntimeException("处理失败", e);  // 保留 cause
    }
}

// 或使用
throw new RuntimeException("处理失败", e);

// 使用 @SneakyThrows 或在方法签名声明
public void process() throws IOException {
    doSomething();
}

// finally 中正确处理异常
public void process() {
    Exception mainException = null;
    try {
        doSomething();
    } catch (Exception e) {
        mainException = e;
        throw e;
    } finally {
        try {
            closeResource();
        } catch (Exception e) {
            if (mainException != null) {
                log.error("关闭资源时发生异常", e);
                // 主异常已经抛出，只记录这个异常
            } else {
                throw e;  // 没有主异常，抛出这个
            }
        }
    }
}

// 或使用 try-with-resources（推荐）
public void process() {
    try (Resource resource = acquireResource()) {
        doSomething();
    }  // 自动关闭，异常处理正确
}

// 在 catch 块中添加上下文信息
public User getUser(Long id) {
    try {
        return userRepository.findById(id);
    } catch (Exception e) {
        log.error("查询用户失败, userId: {}", id, e);
        throw new BusinessException("查询用户失败: " + id, e);
    }
}

// 记录详细的错误上下文
public void transfer(Long fromUserId, Long toUserId, BigDecimal amount) {
    try {
        accountService.transfer(fromUserId, toUserId, amount);
    } catch (InsufficientBalanceException e) {
        log.warn("账户余额不足, fromUserId: {}, toUserId: {}, amount: {}",
            fromUserId, toUserId, amount, e);
        throw e;
    } catch (Exception e) {
        log.error("转账失败, fromUserId: {}, toUserId: {}, amount: {}",
            fromUserId, toUserId, amount, e);
        throw new TransferException("转账失败", e);
    }
}
```

### 4. 数值溢出与精度问题

#### ❌ 错误模式

```java
// ❌ 整数溢出
public int calculateTotal(int count, int price) {
    return count * price;  // 可能溢出
}

// ❌ 无符号右移
public int divideBy2(int value) {
    return value >> 1;  // 负数会出错
}

// ❌ 浮点数比较
public boolean isEqual(double a, double b) {
    return a == b;  // 浮点数不应直接比较
}

// ❌ BigDecimal 使用 double 构造
public BigDecimal createAmount(double value) {
    return new BigDecimal(value);  // 精度丢失
}

// ❌ BigDecimal 除法未指定精度
public BigDecimal divide(BigDecimal a, BigDecimal b) {
    return a.divide(b);  // 可能抛出 ArithmeticException
}
```

#### ✅ 正确模式

```java
// 检查整数溢出
public long calculateTotal(int count, int price) {
    long total = (long) count * price;
    if (total > Integer.MAX_VALUE || total < Integer.MIN_VALUE) {
        throw new ArithmeticException("计算结果溢出");
    }
    return total;
}

// 或使用 Math.multiplyExact()
public int calculateTotal(int count, int price) {
    try {
        return Math.multiplyExact(count, price);
    } catch (ArithmeticException e) {
        throw new BusinessException("金额超出范围", e);
    }
}

// 使用算术右移
public int divideBy2(int value) {
    return value / 2;
}

// 浮点数比较使用误差范围
private static final double EPSILON = 1e-10;

public boolean isEqual(double a, double b) {
    return Math.abs(a - b) < EPSILON;
}

// 或使用 Double.compare()
public int compareDouble(double a, double b) {
    return Double.compare(a, b);
}

// BigDecimal 使用字符串构造
public BigDecimal createAmount(double value) {
    return new BigDecimal(Double.toString(value));
}

// 或直接使用字符串
public BigDecimal createAmount(String value) {
    return new BigDecimal(value);
}

// 除法指定精度和舍入模式
public BigDecimal divide(BigDecimal a, BigDecimal b) {
    return a.divide(b, 2, RoundingMode.HALF_UP);
}
```

### 5. 字符串处理问题

#### ❌ 错误模式

```java
// ❌ 字符串比较使用 ==
public boolean isAdmin(String role) {
    return role == "admin";  // 应该使用 equals
}

// ❌ 字符串拼接在循环中
public String buildList(List<String> items) {
    String result = "";
    for (String item : items) {
        result += item;  // 性能差
    }
    return result;
}

// ❌ 大小写转换用于比较
public boolean checkName(String name) {
    return name.toUpperCase().equals("ADMIN");  // 可能产生空指针
}
```

#### ✅ 正确模式

```java
// 使用 equals
public boolean isAdmin(String role) {
    return "admin".equals(role);  // 或 Objects.equals(role, "admin")
}

// 使用 StringBuilder
public String buildList(List<String> items) {
    StringBuilder sb = new StringBuilder();
    for (String item : items) {
        sb.append(item);
    }
    return sb.toString();
}

// 或使用 String.join()
public String buildList(List<String> items) {
    return String.join(",", items);
}

// 或使用流
public String buildList(List<String> items) {
    return items.stream().collect(Collectors.joining(","));
}

// 大小写不敏感比较
public boolean checkName(String name) {
    return "admin".equalsIgnoreCase(name);
}
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| 空指针未检查 | NullPointerException | 🚨 严重 |
| 链式调用未检查 | NullPointerException | 🚨 严重 |
| 自动拆箱未检查 | NullPointerException | 🚨 严重 |
| 数组越界 | IndexOutOfBoundsException | 🚨 严重 |
| 集合越界 | IndexOutOfBoundsException | 🚨 严重 |
| 字符串截取未检查 | StringIndexOutOfBoundsException | 🚨 严重 |
| 异常被吞掉 | 问题难以排查 | 🚨 严重 |
| 异常未记录 | 无法定位问题 | 🚨 严重 |
| 异常信息丢失 | 根因难以追踪 | ⚠️ 中等 |
| 整数溢出 | 数据错误 | 🚨 严重 |
| 浮点数比较 | 逻辑错误 | ⚠️ 中等 |
| BigDecimal 精度问题 | 金额计算错误 | 🚨 严重 |
| 字符串 == 比较 | 逻辑错误 | 🚨 严重 |
| 循环中字符串拼接 | 性能问题 | ⚠️ 中等 |
