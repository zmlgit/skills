# 常见漏洞模式 (Common Vulnerabilities)

## 检查清单

### 1. 资源泄漏

#### ❌ 错误模式 - 未正确关闭资源

```java
// ❌ 流未关闭
public void readFile(String path) throws IOException {
    FileInputStream fis = new FileInputStream(path);
    // 如果这里抛出异常，流不会关闭
    byte[] data = new byte[fis.available()];
    fis.read(data);
    fis.read();  // 可能永远不会执行
}

// ❌ 即使有 finally，也可能有问题
public void readFile(String path) throws IOException {
    FileInputStream fis = new FileInputStream(path);
    try {
        // 操作
    } finally {
        fis.close();  // 如果 close() 抛出异常，原始异常丢失
    }
}
```

#### ✅ 正确模式

```java
// 方式1：使用 try-with-resources
public void readFile(String path) throws IOException {
    try (FileInputStream fis = new FileInputStream(path)) {
        byte[] data = fis.readAllBytes();
        // 自动关闭，即使抛出异常
    }
}

// 方式2：多个资源
public void copyFile(String src, String dest) throws IOException {
    try (FileInputStream fis = new FileInputStream(src);
         FileOutputStream fos = new FileOutputStream(dest)) {
        fos.write(fis.readAllBytes());
    }
}

// 方式3：如果需要手动关闭，使用多个 finally 块
public void closeResources(Closeable... resources) {
    for (Closeable resource : resources) {
        if (resource != null) {
            try {
                resource.close();
            } catch (IOException e) {
                log.error("关闭资源失败", e);
                // 继续关闭其他资源
            }
        }
    }
}
```

### 2. SQL 注入风险

#### ❌ 错误模式

```java
// ❌ 字符串拼接 SQL
public User findUser(String username) {
    String sql = "SELECT * FROM users WHERE username = '" + username + "'";
    return jdbcTemplate.query(sql, userRowMapper);
    // 如果 username = "admin' OR '1'='1"，会导致 SQL 注入
}

// ❌ 即使使用 PreparedStatement，如果不正确使用也有问题
public User findUser(String username) {
    String sql = "SELECT * FROM users WHERE username = ?";
    return jdbcTemplate.query(sql, new Object[]{username}, userRowMapper);
}
```

#### ✅ 正确模式

```java
// 使用命名参数
public User findUser(String username) {
    String sql = "SELECT * FROM users WHERE username = :username";
    MapSqlParameterSource params = new MapSqlParameterSource()
        .addValue("username", username);
    return namedParameterJdbcTemplate.queryForObject(
        sql, params, userRowMapper);
}

// 使用 JPA Repository（自动防止 SQL 注入）
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.username = :username")
    Optional<User> findByUsername(@Param("username") String username);
}

// 使用 LIKE 时注意转义
@Query("SELECT u FROM User u WHERE u.username LIKE CONCAT('%', :keyword, '%')")
List<User> searchByUsername(@Param("keyword") String keyword);
```

### 3. 路径遍历漏洞

#### ❌ 错误模式

```java
// ❌ 未验证路径
public File getFile(String filename) {
    File file = new File("/app/files/" + filename);
    return file;
    // 如果 filename = "../../etc/passwd"，可以访问任意文件
}
```

#### ✅ 正确模式

```java
// 方式1：规范化并验证路径
public File getFile(String filename) throws IOException {
    Path basePath = Paths.get("/app/files").toAbsolutePath().normalize();
    Path requestedPath = basePath.resolve(filename).normalize();

    if (!requestedPath.startsWith(basePath)) {
        throw new SecurityException("非法路径访问");
    }

    return requestedPath.toFile();
}

// 方式2：使用白名单
private static final Pattern FILENAME_PATTERN =
    Pattern.compile("^[a-zA-Z0-9._-]+$");

public File getFile(String filename) {
    if (!FILENAME_PATTERN.matcher(filename).matches()) {
        throw new SecurityException("非法文件名");
    }

    File file = new File("/app/files", filename);
    if (!file.getCanonicalPath().startsWith("/app/files/")) {
        throw new SecurityException("路径遍历攻击");
    }
    return file;
}
```

### 4. 日志注入

#### ❌ 错误模式

```java
// ❌ 直接记录用户输入
public void logUserAction(String username, String action) {
    log.info("用户 {} 执行了 {}", username, action);
    // 如果 action 包含换行符，可能伪造日志
    // action = "转账\n2024-01-01 10:00:00 系统管理员 登录成功"
}
```

#### ✅ 正确模式

```java
// 清理换行符和其他控制字符
public void logUserAction(String username, String action) {
    String sanitizedUsername = username.replaceAll("[\r\n]", "");
    String sanitizedAction = action.replaceAll("[\r\n]", "");
    log.info("用户 {} 执行了 {}", sanitizedUsername, sanitizedAction);
}

// 或使用 JSON 格式
public void logUserAction(String username, String action) {
    log.info("用户操作: {}",
        Json.createObjectBuilder()
            .add("username", username)
            .add("action", action)
            .build());
}
```

### 5. 命令注入

#### ❌ 错误模式

```java
// ❌ 直接执行命令
public void pingHost(String host) throws IOException {
    String cmd = "ping -c 4 " + host;
    Runtime.getRuntime().exec(cmd);
    // 如果 host = "8.8.8.8; rm -rf /"，会执行恶意命令
}

// ❌ 即使使用 String[] 也有风险
public void pingHost(String host) throws IOException {
    Runtime.getRuntime().exec(new String[]{"ping", "-c", "4", host});
    // 仍然可能注入参数
}
```

#### ✅ 正确模式

```java
// 使用 ProcessBuilder 并验证输入
private static final Pattern HOSTNAME_PATTERN =
    Pattern.compile("^[a-zA-Z0-9.-]+$");

public void pingHost(String host) throws IOException {
    if (!HOSTNAME_PATTERN.matcher(host).matches()) {
        throw new IllegalArgumentException("非法主机名");
    }

    ProcessBuilder pb = new ProcessBuilder("ping", "-c", "4", host);
    pb.redirectErrorStream(true);
    Process process = pb.start();

    // 设置超时
    if (!process.waitFor(30, TimeUnit.SECONDS)) {
        process.destroyForcibly();
        throw new TimeoutException("Ping 超时");
    }
}

// 更好的方式：使用 Java 库而不是系统命令
public boolean checkHostConnectivity(String host, int port, int timeout) {
    try (Socket socket = new Socket()) {
        socket.connect(new InetSocketAddress(host, port), timeout);
        return true;
    } catch (IOException e) {
        return false;
    }
}
```

### 6. 不安全的反序列化

#### ❌ 错误模式

```java
// ❌ 反序列化不可信数据
public Object deserialize(byte[] data) throws Exception {
    ByteArrayInputStream bis = new ByteArrayInputStream(data);
    ObjectInputStream ois = new ObjectInputStream(bis);
    return ois.readObject();
    // 可能导致远程代码执行
}
```

#### ✅ 正确模式

```java
// 方式1：使用安全的格式（JSON）
public User deserializeUser(String json) throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    return mapper.readValue(json, User.class);
}

// 方式2：如果必须使用 Java 序列化，进行白名单验证
public class ValidatingObjectInputStream extends ObjectInputStream {
    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "com.example.model.User",
        "com.example.model.Order",
        "java.lang.String",
        "java.util.*"
    );

    public ValidatingObjectInputStream(InputStream in) throws IOException {
        super(in);
    }

    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException {
        String className = desc.getName();

        // 检查通配符
        boolean allowed = ALLOWED_CLASSES.stream().anyMatch(pattern -> {
            if (pattern.endsWith(".*")) {
                return className.startsWith(pattern.substring(0, pattern.length() - 1));
            }
            return className.equals(pattern);
        });

        if (!allowed) {
            throw new InvalidClassException("未授权的类: " + className);
        }

        return super.resolveClass(desc);
    }
}
```

### 7. 敏感信息泄露

#### ❌ 错误模式

```java
// ❌ 日志中记录敏感信息
public void processPayment(Payment payment) {
    log.info("处理支付: 卡号={}, CVV={}",
        payment.getCardNumber(), payment.getCvv());
}

// ❌ 异常中暴露敏感信息
public void login(String username, String password) {
    try {
        User user = userService.authenticate(username, password);
    } catch (Exception e) {
        throw new RuntimeException("登录失败: " + username + ":" + password, e);
    }
}

// ❌ toString() 暴露敏感信息
public class User {
    private String password;
    private String apiKey;

    @Override
    public String toString() {
        return "User{password='" + password + "', apiKey='" + apiKey + "'}";
    }
}
```

#### ✅ 正确模式

```java
// 不记录敏感信息
public void processPayment(Payment payment) {
    String maskedCard = maskCardNumber(payment.getCardNumber());
    log.info("处理支付: 卡号={}", maskedCard);
}

private String maskCardNumber(String cardNumber) {
    if (cardNumber == null || cardNumber.length() < 4) {
        return "****";
    }
    return "****" + cardNumber.substring(cardNumber.length() - 4);
}

// 异常不包含敏感信息
public void login(String username, String password) {
    try {
        User user = userService.authenticate(username, password);
    } catch (Exception e) {
        // 不记录密码
        log.error("用户 {} 登录失败", username);
        throw new AuthenticationException("登录失败", e);
    }
}

// toString() 不包含敏感字段
public class User {
    private String password;
    private String apiKey;

    @Override
    public String toString() {
        return "User{username='" + username + "'}";
    }
}
```

### 8. 输入验证缺失

#### ❌ 错误模式 - 未验证业务参数

```java
// ❌ 金额允许负数
@PostMapping("/pay")
public Result pay(@RequestParam Long orderId, @RequestParam BigDecimal amount) {
    paymentService.pay(orderId, amount);  // amount 可能是负数
    return Result.success();
}

// ❌ 分页参数未验证
@GetMapping("/orders")
public List<Order> getOrders(@RequestParam Integer page,
                              @RequestParam Integer size) {
    // page 可能是负数，size 可能是 0 或负数
    return orderService.findOrders(page, size);
}

// ❌ 枚举值未验证
public void updateOrderStatus(Long orderId, String status) {
    // status 可以是任意字符串
    orderMapper.updateStatus(orderId, status);
}
```

#### ✅ 正确模式 - JSR-303 验证

```java
// 使用 JSR-303 Bean Validation
public static class PayRequest {
    @NotNull(message = "订单ID不能为空")
    private Long orderId;

    @NotNull(message = "支付金额不能为空")
    @DecimalMin(value = "0.01", message = "支付金额必须大于0")
    @Digits(integer = 10, fraction = 2, message = "金额格式不正确")
    private BigDecimal amount;
}

@PostMapping("/pay")
public Result pay(@Valid @RequestBody PayRequest request) {
    paymentService.pay(request.getOrderId(), request.getAmount());
    return Result.success();
}

// 分页参数验证
public static class PageRequest {
    @Min(value = 0, message = "页码必须大于等于0")
    private Integer page = 0;

    @Min(value = 1, message = "每页数量必须大于0")
    @Max(value = 100, message = "每页数量不能超过100")
    private Integer size = 20;
}

// 枚举验证
public void updateOrderStatus(Long orderId, String status) {
    if (!Arrays.asList("PENDING", "PAID", "CANCELLED").contains(status)) {
        throw new IllegalArgumentException("非法的订单状态: " + status);
    }
    orderMapper.updateStatus(orderId, status);
}

// 或使用自定义验证注解
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = OrderStatusValidator.class)
public @interface ValidOrderStatus {
    String message() default "非法的订单状态";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class OrderStatusValidator implements ConstraintValidator<ValidOrderStatus, String> {
    private static final Set<String> VALID_STATUSES = Set.of("PENDING", "PAID", "CANCELLED");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || VALID_STATUSES.contains(value);
    }
}
```

### 9. IDOR (不安全的直接对象引用)

#### ❌ 错误模式 - 未检查资源所有权

```java
// ❌ 任何人都可以取消任何订单
@DeleteMapping("/order/{orderId}")
public Result cancelOrder(@PathVariable Long orderId) {
    orderService.cancelOrder(orderId);  // 没有检查订单是否属于当前用户
    return Result.success();
}

// ❌ 任何人都可以查看任何用户的信息
@GetMapping("/user/{userId}")
public Result getUser(@PathVariable Long userId) {
    User user = userService.findById(userId);  // 没有检查权限
    return Result.success(user);
}

// ❌ 任何人都可以修改任何资源
@PutMapping("/account/{accountId}")
public Result updateAccount(@PathVariable Long accountId, @RequestBody Account account) {
    accountService.update(accountId, account);  // 没有所有权检查
    return Result.success();
}
```

#### ✅ 正确模式 - 所有权验证

```java
// 方式1：在 Controller 层检查
@DeleteMapping("/order/{orderId}")
public Result cancelOrder(@RequestHeader("Authorization") String token,
                         @PathVariable Long orderId) {
    // 从 token 获取当前用户
    Claims claims = jwtUtil.parseToken(token);
    Long currentUserId = claims.getUserId();

    // 检查订单所有权
    Order order = orderService.getById(orderId);
    if (order == null) {
        return Result.fail("订单不存在");
    }

    if (!order.getUserId().equals(currentUserId)) {
        log.warn("用户 {} 尝试取消他人订单 {}", currentUserId, orderId);
        return Result.fail("无权限操作该订单");
    }

    orderService.cancelOrder(orderId);
    return Result.success();
}

// 方式2：在 Service 层检查（推荐）
@Service
public class OrderService {

    public void cancelOrder(Long orderId, Long currentUserId) {
        Order order = orderMapper.selectById(orderId);
        if (order == null) {
            throw new NotFoundException("订单不存在");
        }

        if (!order.getUserId().equals(currentUserId)) {
            throw new ForbiddenException("无权限操作该订单");
        }

        orderMapper.updateStatus(orderId, "CANCELLED");
    }
}

// 方式3：使用 Spring Security
@PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")
@DeleteMapping("/order/{orderId}")
public Result cancelOrder(@PathVariable Long orderId) {
    orderService.cancelOrder(orderId);
    return Result.success();
}

@Component
public class OrderSecurity {
    public boolean isOwner(Long orderId, Authentication authentication) {
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        Order order = orderRepository.findById(orderId).orElse(null);
        return order != null && order.getUserId().equals(userDetails.getId());
    }
}

// 方式4：使用自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("#orderId != null")
public @interface CheckOwner {
    String resource() default "order";
}

@CheckOwner(resource = "order")
@DeleteMapping("/order/{orderId}")
public Result cancelOrder(@PathVariable Long orderId) {
    orderService.cancelOrder(orderId);
    return Result.success();
}

// AOP 切面实现
@Aspect
@Component
public class OwnerCheckAspect {

    @Around("@annotation(checkOwner)")
    public Object checkOwner(ProceedingJoinPoint joinPoint, CheckOwner checkOwner) throws Throwable {
        Long resourceId = (Long) joinPoint.getArgs()[0];
        Long currentUserId = getCurrentUserId();

        if (!isOwner(checkOwner.resource(), resourceId, currentUserId)) {
            throw new ForbiddenException("无权限操作该资源");
        }

        return joinPoint.proceed();
    }
}
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| 资源未关闭 | 资源泄漏、文件句柄耗尽 | 🚨 严重 |
| SQL 注入 | 数据泄露、数据篡改 | 🚨 严重 |
| 路径遍历 | 任意文件读取 | 🚨 严重 |
| 日志注入 | 日志伪造、问题隐藏 | ⚠️ 中等 |
| 命令注入 | 远程代码执行 | 🚨 严重 |
| 不安全反序列化 | 远程代码执行 | 🚨 严重 |
| 敏感信息泄露 | 数据泄露 | 🚨 严重 |
| 输入验证缺失 | 业务逻辑绕过、数据篡改 | 🚨 严重 |
| IDOR 漏洞 | 越权访问、数据泄露 | 🚨 严重 |
