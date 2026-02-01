# 异步执行安全 (Async Execution Safety)

## 检查清单

### 1. 线程池未正确关闭

#### ❌ 错误模式

```java
public class TaskExecutor {
    // ❌ 没有优雅关闭机制
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public void submitTask(Runnable task) {
        executor.submit(task);
    }

    // 应用关闭时，线程池没有被关闭，导致 JVM 无法正常退出
}
```

#### ✅ 正确模式

```java
public class TaskExecutor {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public void submitTask(Runnable task) {
        executor.submit(task);
    }

    // 添加关闭钩子
    @PreDestroy
    public void shutdown() {
        executor.shutdown();  // 停止接受新任务
        try {
            // 等待现有任务完成
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();  // 强制终止
                // 等待任务响应中断
                if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                    log.error("线程池未能完全关闭");
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2. 异常被吞没

#### ❌ 错误模式

```java
// ❌ 使用 Future 但不检查异常
public void processData() {
    ExecutorService executor = Executors.newFixedThreadPool(10);
    Future<String> future = executor.submit(() -> {
        // 如果这里抛出异常，调用方可能不知道
        return heavyComputation();
    });

    // 没有检查 future.get() 的异常
}

// 或使用 CompletableFuture 时忽略异常
public void asyncProcess() {
    CompletableFuture.supplyAsync(() -> {
        return process();
    })
    .thenAccept(result -> {
        // 只处理成功情况
        log.info("处理成功: {}", result);
    });
    // ❌ 异常被静默忽略
}
```

#### ✅ 正确模式

```java
// 方式1：正确处理 Future
public void processData() throws Exception {
    ExecutorService executor = Executors.newFixedThreadPool(10);
    Future<String> future = executor.submit(() -> {
        return heavyComputation();
    });

    try {
        String result = future.get(10, TimeUnit.SECONDS);
        log.info("处理成功: {}", result);
    } catch (TimeoutException e) {
        future.cancel(true);
        log.error("处理超时", e);
        throw e;
    } catch (ExecutionException e) {
        log.error("处理失败", e.getCause());
        throw e;
    }
}

// 方式2：使用 exceptionally
public void asyncProcess() {
    CompletableFuture.supplyAsync(() -> process())
        .thenAccept(result -> {
            log.info("处理成功: {}", result);
        })
        .exceptionally(throwable -> {
            log.error("处理失败", throwable);
            // 返回默认值或重新抛出
            return null;
        });
}

// 方式3：使用 handle
public void asyncProcess() {
    CompletableFuture.supplyAsync(() -> process())
        .handle((result, throwable) -> {
            if (throwable != null) {
                log.error("处理失败", throwable);
                return null;
            }
            log.info("处理成功: {}", result);
            return result;
        });
}
```

### 3. @Async 异常处理

#### ❌ 错误模式

```java
@Service
public class AsyncService {

    @Async
    public void asyncMethod() {
        // ❌ 异常会被吞没，调用方无法感知
        throw new RuntimeException("异步执行失败");
    }

    // 即使调用方有 try-catch，也捕获不到这个异常
}
```

#### ✅ 正确模式

```java
// 方式1：返回 Future
@Async
public Future<Void> asyncMethod() {
    try {
        // 操作
        return AsyncResult.forValue(null);
    } catch (Exception e) {
        return AsyncResult.forExecutionException(e);
    }
}

// 方式2：配置全局异常处理器
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> {
            log.error("异步方法执行失败: method={}, params={}",
                method.getName(), Arrays.toString(params), throwable);
            // 发送告警等
        };
    }
}

// 方式3：使用 CompletableFuture
@Async
public CompletableFuture<Void> asyncMethod() {
    return CompletableFuture.runAsync(() -> {
        // 操作
    })
    .exceptionally(throwable -> {
        log.error("异步执行失败", throwable);
        return null;
    });
}
```

### 4. 线程池配置不当

#### ❌ 错误模式

```java
// ❌ 无界队列导致内存溢出
ExecutorService executor = new ThreadPoolExecutor(
    1, 1,
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>()  // 无界队列！
);

// ❌ 使用 Executors 创建的线程池
ExecutorService executor1 = Executors.newFixedThreadPool(10);
// 问题：使用无界队列，任务积压可能导致 OOM

ExecutorService executor2 = Executors.newCachedThreadPool();
// 问题：线程数无限制，可能导致线程数过多
```

#### ✅ 正确模式

```java
// 使用有界队列和拒绝策略
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,           // 核心线程数
    maximumPoolSize,        // 最大线程数
    keepAliveTime,          // 空闲线程存活时间
    TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000),  // 有界队列
    new ThreadFactoryBuilder()
        .setNameFormat("async-pool-%d")
        .setUncaughtExceptionHandler((t, e) -> {
            log.error("线程执行异常: {}", t.getName(), e);
        })
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
);
```

### 5. 上下文丢失

#### ❌ 错误模式

```java
public void processRequest() {
    // 主线程设置上下文
    UserContext.set(getCurrentUser());
    MDC.put("traceId", UUID.randomUUID().toString());

    // 异步执行
    CompletableFuture.runAsync(() -> {
        // ❌ 上下文丢失！
        // UserContext.get() 返回 null
        // MDC.get("traceId") 返回 null
        doSomething();
    });
}
```

#### ✅ 正确模式

```java
// 方式1：使用装饰器
public void processRequest() {
    UserContext.set(getCurrentUser());
    String traceId = UUID.randomUUID().toString();
    MDC.put("traceId", traceId);

    CompletableFuture.runAsync(() -> {
        // 手动传递上下文
        UserContext.set(getCurrentUser());
        MDC.put("traceId", traceId);
        try {
            doSomething();
        } finally {
            UserContext.clear();
            MDC.clear();
        }
    });
}

// 方式2：使用上下文装饰器
public class ContextAwareExecutor implements Executor {
    private final Executor delegate;

    @Override
    public void execute(Runnable command) {
        // 捕获当前上下文
        User user = UserContext.get();
        Map<String, String> mdcContext = MDC.getCopyOfContextMap();

        delegate.execute(() -> {
            try {
                // 恢复上下文
                if (user != null) {
                    UserContext.set(user);
                }
                if (mdcContext != null) {
                    MDC.setContextMap(mdcContext);
                }
                command.run();
            } finally {
                UserContext.clear();
                MDC.clear();
            }
        });
    }
}
```

### 6. 并发限制

#### ❌ 错误模式

```java
// ❌ 无限制地创建异步任务
public void processAllItems(List<Item> items) {
    for (Item item : items) {
        CompletableFuture.runAsync(() -> {
            processItem(item);
        });
    }
    // 如果有 10000 个项目，会创建 10000 个线程！
}
```

#### ✅ 正确模式

```java
// 方式1：使用限流的执行器
private final ExecutorService executor =
    new ThreadPoolExecutor(10, 50, 60, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy());

public void processAllItems(List<Item> items) {
    for (Item item : items) {
        executor.submit(() -> processItem(item));
    }
}

// 方式2：使用并行流控制并发
public void processAllItems(List<Item> items) {
    items.parallelStream()
        .forEach(item -> processItem(item));
}

// 方式3：使用信号量限制并发
private final Semaphore semaphore = new Semaphore(10);

public void processAllItems(List<Item> items) {
    List<CompletableFuture<Void>> futures = items.stream()
        .map(item -> CompletableFuture.runAsync(() -> {
            try {
                semaphore.acquire();
                try {
                    processItem(item);
                } finally {
                    semaphore.release();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }))
        .collect(Collectors.toList());

    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
}
```

### 7. 超时控制

#### ❌ 错误模式

```java
// ❌ 没有超时控制，可能永远阻塞
public void asyncOperation() {
    CompletableFuture.supplyAsync(() -> {
        return callExternalService();  // 可能永远不返回
    })
    .thenAccept(result -> {
        log.info("操作完成: {}", result);
    });
}
```

#### ✅ 正确模式

```java
public void asyncOperation() {
    CompletableFuture.supplyAsync(() -> callExternalService())
        .orTimeout(5, TimeUnit.SECONDS)  // 添加超时
        .thenAccept(result -> {
            log.info("操作完成: {}", result);
        })
        .exceptionally(throwable -> {
            if (throwable instanceof TimeoutException) {
                log.error("操作超时");
            } else {
                log.error("操作失败", throwable);
            }
            return null;
        });
}

// 或使用 completeOnTimeout
public void asyncOperation() {
    CompletableFuture.supplyAsync(() -> callExternalService())
        .completeOnTimeout(null, 5, TimeUnit.SECONDS)
        .thenAccept(result -> {
            if (result == null) {
                log.warn("操作超时，使用默认值");
            } else {
                log.info("操作完成: {}", result);
            }
        });
}
```

## 检查要点总结

| 检查项 | 风险 | 优先级 |
|--------|------|--------|
| 线程池未关闭 | 资源泄漏、无法退出 | 🚨 严重 |
| 异常被吞没 | 问题隐藏、难以调试 | 🚨 严重 |
| @Async 异常未处理 | 问题隐藏 | 🚨 严重 |
| 无界队列 | 内存溢出 | 🚨 严重 |
| 上下文丢失 | 跟踪困难、权限问题 | ⚠️ 中等 |
| 无限并发创建 | 资源耗尽 | 🚨 严重 |
| 无超时控制 | 长时间阻塞 | ⚠️ 中等 |
