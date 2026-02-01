# 竞态条件 (Race Conditions)

## 检查清单

### 1. Check-Then-Act 模式（先检查后执行）

#### ❌ 错误模式 - 经典竞态条件

```java
// 单例模式的错误实现
private static Instance instance;

public static Instance getInstance() {
    if (instance == null) {  // 检查
        instance = new Instance();  // 执行
    }
    return instance;
}

// 多线程环境下可能创建多个实例！
```

#### ✅ 正确模式 - 双重检查锁定（需要 volatile）

```java
private static volatile Instance instance;

public static Instance getInstance() {
    Instance result = instance;
    if (result == null) {
        synchronized(Instance.class) {
            result = instance;
            if (result == null) {
                instance = result = new Instance();
            }
        }
    }
    return result;
}
```

### 2. 集合操作的竞态条件

#### ❌ 错误模式

```java
// 检查后操作 - 竞态条件
if (map.containsKey(key)) {
    return map.get(key);  // 可能已被删除
} else {
    return defaultValue;
}

// 或
if (!map.containsKey(key)) {
    map.put(key, value);  // 可能被其他线程先插入
}
```

#### ✅ 正确模式

```java
// 使用原子方法
return map.getOrDefault(key, defaultValue);

// 或
map.putIfAbsent(key, value);

// 或 computeIfAbsent
map.computeIfAbsent(key, k -> computeValue(k));
```

### 3. 不可变对象的误用

#### ❌ 错误模式

```java
// Date 是可变的！
public class Event {
    private final Date date;

    public Event(Date date) {
        this.date = date;  // 引用可变对象
    }

    public Date getDate() {
        return date;  // 返回内部可变对象的引用
    }
}

// 调用方可以修改
Date eventDate = event.getDate();
eventDate.setTime(0);  // 破坏了封装
```

#### ✅ 正确模式

```java
public class Event {
    private final Date date;

    public Event(Date date) {
        // 防御性拷贝
        this.date = new Date(date.getTime());
    }

    public Date getDate() {
        // 防御性拷贝
        return new Date(date.getTime());
    }
}

// 更好的方式：使用 LocalDateTime（不可变）
```

### 4. 发布不完整的对象

#### ❌ 错误模式 - 不安全的发布

```java
public class Holder {
    private int n;

    public Holder(int n) {
        this.n = n;
    }

    public void assertSanity() {
        if (n != n) {
            throw new AssertionError("不安全发布");
        }
    }
}

// 线程 A 可能看到未完全构造的对象
```

#### ✅ 正确模式

```java
// 方式1：使用 final 字段
public class Holder {
    private final int n;  // final 保证安全发布

    public Holder(int n) {
        this.n = n;
    }
}

// 方式2：使用安全发布
public static Holder holder = new Holder(42);  // static final

// 方式3：使用 volatile
public static volatile Holder holder;

// 方式4：使用 AtomicReference
private static final AtomicReference<Holder> holderRef =
    new AtomicReference<>();
```

### 5. 延迟初始化的竞态条件

#### ❌ 错误模式

```java
public class LazyInit {
    private ExpensiveObject instance;

    public ExpensiveObject getInstance() {
        if (instance == null) {  // 竞态条件
            instance = new ExpensiveObject();
        }
        return instance;
    }
}
```

#### ✅ 正确模式

```java
// 方式1： synchronized（简单但性能较低）
public synchronized ExpensiveObject getInstance() {
    if (instance == null) {
        instance = new ExpensiveObject();
    }
    return instance;
}

// 方式2：双重检查锁定
private volatile ExpensiveObject instance;

public ExpensiveObject getInstance() {
    ExpensiveObject result = instance;
    if (result == null) {
        synchronized(this) {
            result = instance;
            if (result == null) {
                instance = result = new ExpensiveObject();
            }
        }
    }
    return result;
}

// 方式3：内部静态类 Holder 模式（推荐）
private static class Holder {
    static final ExpensiveObject INSTANCE = new ExpensiveObject();
}

public ExpensiveObject getInstance() {
    return Holder.INSTANCE;
}
```

### 6. 非线程安全的对象共享

#### ❌ 错误模式

```java
// SimpleDateFormat 非线程安全
public class DateUtil {
    private static final SimpleDateFormat FORMAT =
        new SimpleDateFormat("yyyy-MM-dd");

    public static String format(Date date) {
        return FORMAT.format(date);  // 多线程下会出错！
    }
}
```

#### ✅ 正确模式

```java
// 方式1：每次创建新实例
public static String format(Date date) {
    return new SimpleDateFormat("yyyy-MM-dd").format(date);
}

// 方式2：使用 ThreadLocal
private static final ThreadLocal<SimpleDateFormat> FORMAT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public static String format(Date date) {
    return FORMAT.get().format(date);
}

// 方式3：使用 DateTimeFormatter（线程安全）
private static final DateTimeFormatter FORMATTER =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

### 7. 迭代器修改集合

#### ❌ 错误模式

```java
List<String> list = new ArrayList<>();

for (String item : list) {
    if (shouldRemove(item)) {
        list.remove(item);  // ConcurrentModificationException
    }
}
```

#### ✅ 正确模式

```java
// 使用 Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String item = it.next();
    if (shouldRemove(item)) {
        it.remove();
    }
}

// 或使用 removeIf（Java 8+）
list.removeIf(this::shouldRemove);

// 或使用流过滤
List<String> result = list.stream()
    .filter(item -> !shouldRemove(item))
    .collect(Collectors.toList());
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| if-contains-then-put/get | 竞态条件 | 🚨 严重 |
| 单例双重检查无 volatile | 不安全发布 | 🚨 严重 |
| 返回内部可变对象引用 | 封装破坏 | 🚨 严重 |
| 共享 SimpleDateFormat | 数据错误 | 🚨 严重 |
| 迭代时修改集合 | 异常/错误 | 🚨 严重 |
| 构造函数中 this 引用逃逸 | 不安全发布 | 🚨 严重 |
