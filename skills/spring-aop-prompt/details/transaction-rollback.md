# Transaction Rollback Edge Cases

Load this when reviewing `@Transactional` methods with exception handling.

## Default Behavior

Spring ONLY rolls back on `RuntimeException` and `Error`, NOT on checked exceptions.

## 8 Common Edge Cases

### 1. Checked Exceptions Don't Trigger Rollback

```java
// ❌ BAD: IOException does NOT rollback
@Transactional
public void updateUser(Long id) throws IOException {
    userRepository.save(user);
    throw new IOException("File processing failed");
}

// ✅ GOOD: Explicit rollbackFor
@Transactional(rollbackFor = IOException.class)
public void updateUser(Long id) throws IOException {
    userRepository.save(user);
    throw new IOException("File processing failed");
}
```

### 2. Try-Catch Swallows Exception

```java
// ❌ BAD: Exception swallowed
@Transactional
public void transferMoney(...) {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("failed", e); // Transaction commits!
    }
}

// ✅ GOOD: Re-throw
@Transactional
public void transferMoney(...) {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("failed", e);
        throw e;
    }
}

// ✅ GOOD: setRollbackOnly
@Transactional
public void transferMoney(...) {
    try {
        riskyOperation();
    } catch (Exception e) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

### 3. Wrong Exception Type in rollbackFor

```java
// ❌ BAD: Wrong exception class
@Transactional(rollbackFor = SQLException.class) // Not a Spring wrapped exception
public void saveUser(User user) { ... }
```

### 4. Nested Transactions with Different Rollback Rules

```java
// With REQUIRED propagation (default), only OUTER transaction's rules apply
@Service
public class OuterService {
    @Transactional(rollbackFor = Exception.class)
    public void outerMethod() {
        innerService.innerMethod(); // Inner's rollbackFor ignored!
    }
}
```

### 5. UnexpectedRollbackException

```java
// ❌ PROBLEMATIC: Inner marks rollback-only, outer catches exception
@Transactional
public void outerMethod() {
    try {
        innerService.methodThatFails();
    } catch (Exception e) {
        // Caught, but commit throws UnexpectedRollbackException!
    }
}

// ✅ FIX: Use REQUIRES_NEW
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodThatFails() {
    throw new RuntimeException("Failed");
}
```

### 6. No Rollback on Specific Exceptions

```java
// ❌ BAD: Validation exception rolls back unnecessarily
@Transactional
public void processOrder(Order order) throws OrderValidationException {
    validate(order); // Throws exception, rolls back
    orderRepository.save(order);
}

// ✅ GOOD: Exclude from rollback
@Transactional(noRollbackFor = OrderValidationException.class)
public void processOrder(Order order) throws OrderValidationException {
    validate(order);
    orderRepository.save(order);
}
```

### 7. LazyInitializationException After Transaction

```java
// ❌ BAD: Lazy load outside transaction
@Transactional
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElse(null);
    // Transaction ends
}
public void processOrder(Long id) {
    Order order = getOrder(id);
    order.getItems(); // LazyInitializationException!
}

// ✅ GOOD: Transaction spans lazy access
@Transactional
public void processOrder(Long id) {
    Order order = orderRepository.findById(id).orElse(null);
    order.getItems(); // OK
}
```

### 8. Database Constraints vs Application Validation

```java
// ❌ BAD: Rely on constraints for rollback
@Transactional
public void createUser(User user) {
    userRepository.save(user); // Slow, error-prone
}

// ✅ GOOD: Validate first
@Transactional
public void createUser(User user) {
    if (userRepository.existsByEmail(user.getEmail())) {
        throw new DuplicateEmailException();
    }
    userRepository.save(user);
}
```

## Detection Checklist

- [ ] Checked exceptions have `rollbackFor` specification
- [ ] Try-catch blocks re-throw or use `setRollbackOnly()`
- [ ] `rollbackFor` / `noRollbackFor` correctly specified
- [ ] Nested transactions have compatible rollback rules
- [ ] Lazy loading within transaction boundaries
- [ ] Application validation before database operations
