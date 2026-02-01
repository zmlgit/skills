---
name: spring-aop
description: Spring AOP detection and guidance for code review - identifies AOP proxy behavior, self-invocation issues, and transaction-related problems in Spring projects with progressive loading
version: 1.0.0
author: zml
license: MIT
tags: [spring, aop, java, code-review, transaction, backend]
---

# Spring AOP

Spring AOP detection and guidance skill for code review agents.

## Purpose

Help code review agents identify and handle Spring AOP-related issues when reviewing Java/Spring projects.

## Progressive Loading Structure

```
SKILL.md (this file) - Core detection heuristics (ALWAYS load)
├── details/transaction-rollback.md - Load when reviewing @Transactional exception handling
├── details/transaction-boundary.md - Load when reviewing transaction scope/performance
├── details/event-rpc-message.md - Load when reviewing ApplicationEvent, RPC, or messaging
├── details/cache-concurrency.md - Load when reviewing cache or concurrency issues
└── details/comprehensive-checklist.md - Load for full systematic review
```

## Quick Detection

Detect Spring AOP usage by checking for:
- `@Aspect`, `@Before`, `@After`, `@Around`, `@Transactional`, `@Async`, `@Cacheable`
- `@Service`, `@Component`, `@Repository`, `@Controller`

## Core AOP Rules (Essential)

| Rule | Description |
|------|-------------|
| **Only public methods proxied** | Private/protected/package-private methods are NOT intercepted |
| **Self-invocation bypasses proxy** | `this.method()` calls do NOT go through proxy |
| **Final methods cannot be proxied** | `final` methods cannot have AOP applied |
| **Self-invocation fix** | Inject self-reference: `@Autowired private SelfClass self;` |

## Common Anti-Patterns

```java
// ❌ BAD: Self-invocation - @Transactional ignored
@Service
public class MyService {
    @Transactional
    public void outer() {
        inner(); // Does NOT trigger transaction!
    }
    @Transactional
    public void inner() { ... }
}

// ✅ GOOD: Inject self-proxy
@Service
public class MyService {
    @Autowired
    private MyService self;
    public void outer() {
        self.inner(); // Goes through proxy - transaction works!
    }
}
```

## When to Load Detailed Sections

| Trigger | Load This Section |
|---------|------------------|
| Found `@Transactional` with exception handling | `details/transaction-rollback.md` |
| Found long methods with `@Transactional` | `details/transaction-boundary.md` |
| Found `@TransactionalEventListener`, RPC calls, or message queues | `details/event-rpc-message.md` |
| Found `@Cacheable` or ThreadLocal usage | `details/cache-concurrency.md` |
| Full systematic review needed | `details/comprehensive-checklist.md` |

## Pre-Code Review Checklist (ALWAYS Do First)

Before analyzing code:
- [ ] Read `application.yml`/`application.properties`
- [ ] Read `bootstrap.yml`/`bootstrap.properties`
- [ ] Identify: **annotation-based** OR **configuration-based** transactions
- [ ] If config-based: find method patterns (e.g., `save*`, `update*`, `delete*`)

## Quick Reference: @Transactional Failure Modes

| Cause | Fix |
|-------|-----|
| Self-invocation `this.method()` | Inject self-proxy |
| Non-public method | Make method public |
| Exception type mismatch | Add `rollbackFor = CheckedException.class` |
| Try-catch swallows exception | Re-throw or use `setRollbackOnly()` |

## Related Skills

- `springboot-patterns`: Spring Boot architecture patterns
- `springboot-security`: Spring Security best practices
