# Transaction Boundary Issues

Load this when reviewing transaction scope, performance, or long-running `@Transactional` methods.

## Problem: Oversized Transaction Boundaries

Transaction boundaries that are too large cause:
- **Lock contention**: DB locks held during non-DB operations
- **Connection pool exhaustion**: DB connections held too long
- **Deadlock risk**: Higher chance with large transactions
- **Poor performance**: Serial execution kills throughput
- **Transaction timeout**: Risk of timeout

## Detection Heuristics

Transaction method contains:
- HTTP/REST API calls
- File I/O operations
- Message queue operations
- Complex calculations/business rules
- Multiple independent database operations
- Method execution time > 1-2 seconds

## Anti-Pattern vs Best Practice

```java
// ❌ BAD: Everything in one transaction
@Transactional
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    PaymentResult payment = paymentService.charge(order.getAmount()); // Remote call!
    String report = reportGenerator.generate(order); // File I/O!
    businessRuleEngine.validate(order); // Complex logic!
    order.setStatus(COMPLETED);
    orderRepository.save(order);
    notificationService.sendNotification(order);
}

// ✅ GOOD: Separate transactional from non-transactional
public void processOrder(Long orderId) {
    // Step 1: No transaction - fetch data
    Order order = orderRepository.findById(orderId);

    // Step 2: No transaction - validate
    businessRuleEngine.validate(order);

    // Step 3: No transaction - external call
    PaymentResult payment = paymentService.charge(order.getAmount());

    // Step 4: Dedicated transaction - only DB operations
    updateOrderWithPayment(orderId, payment);

    // Step 5: No transaction - fire-and-forget
    notificationService.sendNotification(order);
}

@Transactional
public void updateOrderWithPayment(Long orderId, PaymentResult payment) {
    // Only essential DB operations here
    orderRepository.updateStatus(orderId, COMPLETED);
    paymentRepository.save(payment);
}
```

## Transaction Boundary Guidelines

- Keep transactions **short and focused** (< 100ms ideal)
- Include **only database operations** within transaction
- Move remote calls, file I/O, complex logic **outside** transaction
- Use `REQUIRES_NEW` for independent write operations
- Consider eventual consistency for non-critical operations

## Detection Checklist

- [ ] Transaction contains non-DB operations (HTTP, file I/O, messaging)
- [ ] Remote API calls inside transaction boundaries
- [ ] Complex business logic should be outside transaction
- [ ] Multiple independent operations in one transaction
- [ ] Could split into smaller, focused transactions
- [ ] `REQUIRES_NEW` appropriate for independent operations
- [ ] Transaction timeout configured for operation scope
