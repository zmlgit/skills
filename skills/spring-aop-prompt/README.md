# Spring AOP Prompt Skill

Spring AOP detection and guidance skill for code review agents with progressive loading.

## Overview

This skill helps code review agents identify and handle Spring AOP-related issues. Uses progressive loading to minimize token usage - only load relevant sections as needed.

## Progressive Loading Structure

```
SKILL.md (90 lines) - Core detection heuristics (ALWAYS loaded)
├── details/transaction-rollback.md - @Transactional exception handling
├── details/transaction-boundary.md - Transaction scope/performance
├── details/event-rpc-message.md - ApplicationEvent, RPC, messaging
├── details/cache-concurrency.md - Cache and concurrency issues
└── details/comprehensive-checklist.md - Full systematic review
```

## Usage

### Always Load First
```bash
SKILL.md
```
Contains core AOP rules and detection heuristics.

### Load On-Demand Based on Context

| Context | Load This File |
|---------|----------------|
| Reviewing `@Transactional` with exception handling | `details/transaction-rollback.md` |
| Reviewing long methods with `@Transactional` | `details/transaction-boundary.md` |
| Reviewing events, RPC calls, or messaging | `details/event-rpc-message.md` |
| Reviewing `@Cacheable` or ThreadLocal usage | `details/cache-concurrency.md` |
| Full systematic review | `details/comprehensive-checklist.md` |

## Key Features

- **Progressive Loading**: Core ~90 lines always loaded, detailed sections on-demand
- **Detection Heuristics**: Quick reference for common AOP issues
- **Modular Content**: Load only what you need for the review context

## Core AOP Rules

| Rule | Description |
|------|-------------|
| Only public methods proxied | Private/protected/package-private NOT intercepted |
| Self-invocation bypasses proxy | `this.method()` does NOT go through proxy |
| Final methods cannot be proxied | `final` methods cannot have AOP applied |
| Self-invocation fix | Inject self-reference: `@Autowired private SelfClass self;` |

## Common Anti-Patterns

```java
// ❌ BAD: Self-invocation - @Transactional ignored
@Service
public class MyService {
    @Transactional
    public void outer() {
        inner(); // Does NOT trigger transaction!
    }
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

## Installation

Copy this skill directory to your Claude skills repository or install according to your agent's skill loading mechanism.

## See Also

- `SKILL.md` - Core skill documentation
- `details/` - Detailed guides for specific topics
- `springboot-patterns` - Spring Boot architecture patterns
