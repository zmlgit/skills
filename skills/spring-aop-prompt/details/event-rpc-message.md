# ApplicationEvent, RPC, and Message Integration

Load this when reviewing `@TransactionalEventListener`, RPC calls, or message queues.

## 1. ApplicationEvent Transaction Issues

### Event Publication Timing

```java
// ❌ BAD: Default synchronous execution affects main transaction
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    applicationEventPublisher.publishEvent(new OrderEvent(order));
    // If listener throws exception, main transaction rolls back!
}

// ✅ GOOD: Use @TransactionalEventListener
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderEvent(OrderEvent event) {
    // Executes AFTER transaction commit
}
```

### Event Phases

| Phase | When | Use Case |
|-------|------|----------|
| `BEFORE_COMMIT` | Before transaction commit | Modify entity before commit |
| `AFTER_COMMIT` | After transaction commit | External notifications, non-critical updates |
| `AFTER_ROLLBACK` | After transaction rollback | Cleanup, logging |
| `AFTER_COMPLETION` | After commit or rollback | Resource cleanup |

### Event Exception Handling

```java
// ❌ PROBLEMATIC: AFTER_COMMIT failure has no compensation
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderEvent(OrderEvent event) {
    emailService.send(event.getOrder()); // If fails, transaction already committed!
}

// ✅ SOLUTION: Implement retry/deadletter pattern
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderEvent(OrderEvent event) {
    try {
        emailService.send(event.getOrder());
    } catch (Exception e) {
        deadLetterQueue.publish(event);
    }
}
```

## 2. RPC Calls in Transactions

### Problem: RPC in Transaction

```java
// ❌ BAD: RPC call holds DB lock
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    paymentService.charge(order.getAmount()); // Long RPC holding DB lock!
}
```

### Problems
- DB locks held during network call
- Connection pool exhaustion
- Poor throughput

### Solution

```java
// ✅ GOOD: RPC outside transaction
public void processOrder(Order order) {
    // Step 1: Save in transaction
    saveOrder(order);

    // Step 2: RPC call outside transaction
    PaymentResult result = paymentService.charge(order.getAmount());

    // Step 3: Update with result in new transaction
    updateOrderWithPayment(order.getId(), result);
}

@Transactional
public void saveOrder(Order order) {
    orderRepository.save(order);
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void updateOrderWithPayment(Long orderId, PaymentResult result) {
    orderRepository.updateStatus(orderId, PAID);
}
```

### Distributed Transaction Consistency

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| **2PC/XA** | Two-phase commit | Strong consistency required, acceptable latency |
| **Compensating** | Execute compensating actions on failure | Eventual consistency acceptable |
| **Best-effort** | Local commit first, retry remote | Non-critical operations |

## 3. Message Middleware Integration

### Message Sending Timing

```java
// ❌ BAD: Send before commit - phantom message risk
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    messageQueue.send("order.created", order);
    // If rollback happens, message already sent!
}

// ❌ BAD: Send after commit - send failure risk
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
}
// Transaction committed, but send can fail

// ✅ GOOD: Outbox pattern
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    outboxRepository.save(new OutboxMessage("order.created", order));
}
// Separate job processes outbox and sends messages
```

### Transaction Message Patterns

| Pattern | Implementation | Pros | Cons |
|---------|---------------|------|------|
| **Outbox table** | Local message table + poller | Reliable, simple | Polling delay |
| **Transaction synchronization** | `TransactionSynchronizationManager` | Immediate | Complex, order issues |
| **Best-effort** | Send after commit | Simple | Can lose messages |

### Message Idempotency

```java
// ✅ GOOD: Idempotent consumer
@KafkaListener(topics = "orders")
public void handleOrder(OrderMessage message) {
    if (orderRepository.existsByMessageId(message.getId())) {
        return; // Already processed
    }
    orderRepository.save(message.toOrder());
}
```

## Detection Checklist

### Events
- [ ] Event publication timing: default sync vs `@TransactionalEventListener`
- [ ] Event exception handling: rollback behavior, compensation
- [ ] Async events with `@Async`: transaction context propagation

### RPC
- [ ] RPC calls inside transaction boundaries
- [ ] Long RPC calls holding DB locks
- [ ] Distributed transaction consistency strategy
- [ ] RPC exception handling and rollback
- [ ] Idempotency design for RPC calls

### Messaging
- [ ] Message sending timing relative to transaction commit
- [ ] Outbox pattern or transaction synchronization used
- [ ] Message consumption order guarantee
- [ ] Idempotent consumption implementation
