# Cache and Concurrency Issues

Load this when reviewing `@Cacheable`, ThreadLocal, or singleton bean thread safety.

## 1. Cache Related Issues

### Cache and Transaction Consistency

```java
// ❌ BAD: Cache updated before transaction commit
@Transactional
public void updateUser(User user) {
    userRepository.save(user);
    cache.put("user:" + user.getId(), user); // Cached, but may rollback!
}

// ✅ GOOD: Update cache after commit via @TransactionalEventListener
@Transactional
public void updateUser(User user) {
    userRepository.save(user);
    applicationEventPublisher.publishEvent(new UserUpdatedEvent(user.getId()));
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onUserUpdated(UserUpdatedEvent event) {
    cache.evict("user:" + event.getUserId());
}
```

### @Cacheable Self-Invocation Problem

```java
// ❌ BAD: Self-invocation bypasses cache proxy
@Service
public class UserService {
    @Cacheable("users")
    public User findById(Long id) {
        return userRepository.findById(id).orElse(null);
    }

    public User getUserWithValidation(Long id) {
        validate(id);
        return findById(id); // Cache NOT applied!
    }
}

// ✅ GOOD: Inject self-proxy
@Service
public class UserService {
    @Autowired
    private UserService self;

    @Cacheable("users")
    public User findById(Long id) {
        return userRepository.findById(id).orElse(null);
    }

    public User getUserWithValidation(Long id) {
        validate(id);
        return self.findById(id); // Cache applied!
    }
}
```

### Cache Key Design Issues

```java
// ❌ BAD: Complex object as key - may not hash properly
@Cacheable(value = "users", key = "#user.filter")
public List<User> findUsers(Filter filter) { ... }

// ❌ BAD: Null value key issues
@Cacheable(value = "user", key = "#id")
public User findById(Long id) {
    return userRepository.findById(id).orElse(null); // Caches null!
}

// ✅ GOOD: Use `unless` to exclude null
@Cacheable(value = "user", key = "#id", unless = "#result == null")
public User findById(Long id) {
    return userRepository.findById(id).orElse(null);
}
```

### Cache Protection Patterns

| Issue | Protection |
|-------|------------|
| **Cache penetration** (non-existent keys) | Cache null values with TTL, or use bloom filter |
| **Cache avalanche** (many expires at once) | Randomize TTL, use multi-level cache |
| **Cache breakdown** (hot key expiry) | Mutex lock, never expire hot keys |

## 2. Concurrency Safety Issues

### Singleton Bean State Fields

```java
// ❌ BAD: Mutable state in singleton bean
@Service
public class UserService {
    private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd"); // Not thread-safe!
    private List<User> cache = new ArrayList<>(); // Not thread-safe!

    public String formatDate(Date date) {
        return sdf.format(date); // Race condition!
    }
}

// ✅ GOOD: Use thread-safe alternatives
@Service
public class UserService {
    private DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    private ConcurrentHashMap<Long, User> cache = new ConcurrentHashMap<>();

    public String formatDate(LocalDate date) {
        return formatter.format(date);
    }
}
```

### ThreadLocal Cleanup

```java
// ❌ BAD: ThreadLocal not cleaned up
@Component
public class UserContextFilter implements Filter {
    private static final ThreadLocal<User> CONTEXT = new ThreadLocal<>();

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        User user = authenticate(request);
        CONTEXT.set(user);
        chain.doFilter(request, response);
        // Missing cleanup! Memory leak in thread pool!
    }
}

// ✅ GOOD: Always cleanup in finally
@Component
public class UserContextFilter implements Filter {
    private static final ThreadLocal<User> CONTEXT = new ThreadLocal<>();

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        try {
            User user = authenticate(request);
            CONTEXT.set(user);
            chain.doFilter(request, response);
        } finally {
            CONTEXT.remove(); // Always cleanup!
        }
    }
}
```

### Connection Pool Issues

```java
// ❌ BAD: Connection leak
@Transactional
public void processData(Long id) {
    Connection conn = dataSource.getConnection();
    // Missing conn.close() - leak on exception!
    ResultSet rs = conn.createStatement().executeQuery("SELECT ...");
}

// ✅ GOOD: Use try-with-resources
@Transactional
public void processData(Long id) {
    try (Connection conn = dataSource.getConnection()) {
        ResultSet rs = conn.createStatement().executeQuery("SELECT ...");
    } // Automatically closed
}
```

## Detection Checklist

### Cache
- [ ] Cache update timing relative to transaction commit
- [ ] @Cacheable self-invocation issues
- [ ] Cache key design: null handling, complex objects
- [ ] Cache penetration, avalanche, breakdown protection

### Concurrency
- [ ] Singleton bean mutable state fields
- [ ] Non-thread-safe classes (SimpleDateFormat, ArrayList)
- [ ] ThreadLocal cleanup in filters/interceptors
- [ ] ThreadLocal in thread pool scenarios
- [ ] Connection leak: proper resource cleanup
- [ ] Connection pool configuration appropriate
