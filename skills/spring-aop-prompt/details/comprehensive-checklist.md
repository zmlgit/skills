# Comprehensive Spring Code Review Checklist

Load this for full systematic review of Spring applications.

## 2.1 Configuration and Startup Phase

- [ ] **Circular dependency**: Use constructor injection, check `@Autowired` cycles
- [ ] **Profile override**: Check activation order of multiple config files
- [ ] **@ConfigurationProperties**: Check collection type initialization
- [ ] **Configuration security**: Sensitive information hardcoded?

## 2.2 Bean Lifecycle Issues

- [ ] **@PostConstruct ordering**: Dependencies fully initialized?
- [ ] **Prototype in singleton**: Use `ObjectFactory`/`Provider` correctly?
- [ ] **@Lazy initialization**: Hiding initialization errors?
- [ ] **Bean scope misuse**: Thread safety when injecting prototype into singleton?

## 2.3 Transaction Management

### @Transactional Failure Scenarios
- [ ] Self-invocation (`this.method()`)
- [ ] Non-public methods
- [ ] Exception type mismatch
- [ ] Same-class method calls

### Transaction Propagation
- [ ] `REQUIRES_NEW` mixed with `REQUIRED`
- [ ] `NESTED` usage scenarios
- [ ] Transaction timeout settings

### Transaction Isolation
- [ ] Using default level
- [ ] Isolation level under high concurrency
- [ ] Phantom reads, non-repeatable reads

## 2.4 Spring AOP Specific

### Proxy Mode
- [ ] JDK vs CGLIB proxy selection
- [ ] Final methods being proxied?
- [ ] Private methods being proxied?

### Self-Invocation
- [ ] Same-class method calls bypassing proxy
- [ ] Self-proxy injection used as fix?

### AOP Order
- [ ] Multiple aspects execution order
- [ ] `@Transactional` with `@Async` order

## 2.5 ApplicationEvent

- [ ] Event publication timing (sync vs async)
- [ ] `@TransactionalEventListener` phase selection
- [ ] Listener exception rollback behavior
- [ ] `AFTER_COMMIT` failure compensation
- [ ] Async event transaction context propagation

## 2.6 Transaction Interception Configuration

- [ ] AOP expression over-matching (wildcards)
- [ ] `@Transactional` on non-business methods
- [ ] Read-only query optimization
- [ ] Interceptor order (cache, security, logging, transaction)

## 2.7 RPC Calls

- [ ] RPC calls within database transaction
- [ ] Long RPC holding DB locks
- [ ] Distributed transaction consistency strategy
- [ ] RPC failure after local commit
- [ ] Idempotency design

## 2.8 Message Middleware

- [ ] Send timing: before commit (phantom) vs after commit (send failure)
- [ ] Outbox pattern or transaction synchronization
- [ ] Message consumption order
- [ ] Idempotent consumption

## 2.9 Cache

- [ ] Cache update before transaction commit
- [ ] Cache penetration, avalanche, breakdown protection
- [ ] @Cacheable self-invocation
- [ ] Cache key design (null, complex objects)

## 2.10 Concurrency Safety

- [ ] Singleton bean state fields
- [ ] Non-thread-safe classes (SimpleDateFormat)
- [ ] ThreadLocal cleanup
- [ ] ThreadLocal in thread pools
- [ ] Connection pool leaks
- [ ] Redis connection pool config

## 2.11 Performance

### Startup
- [ ] Too many `@Configuration` classes
- [ ] Premature bean initialization

### Runtime
- [ ] Frequent short-lived object creation
- [ ] Improper `@Conditional` usage

### Database
- [ ] N+1 query problems
- [ ] Batch operation implementation

## 2.12 Testing

- [ ] `@Transactional` test auto-rollback
- [ ] Tests requiring commit
- [ ] `@MockBean` affecting other tests
- [ ] Integration vs unit test separation
- [ ] Test data pollution
- [ ] Test data cleanup strategy
