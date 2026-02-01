# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

Spring AOP Prompt skill with progressive loading - minimizes token usage by loading only relevant sections.

## Progressive Loading Structure

```
SKILL.md - Core detection (ALWAYS load, ~90 lines)
├── details/transaction-rollback.md - Load for exception handling review
├── details/transaction-boundary.md - Load for transaction scope review
├── details/event-rpc-message.md - Load for event/RPC/messaging review
├── details/cache-concurrency.md - Load for cache/concurrency review
└── details/comprehensive-checklist.md - Load for full systematic review
```

## How to Use This Skill

### Step 1: Always Load SKILL.md First
Contains core AOP rules and detection heuristics.

### Step 2: Load Detail Files Based on Context

| Trigger | Load This |
|---------|-----------|
| `@Transactional` with try-catch | `details/transaction-rollback.md` |
| Long `@Transactional` methods | `details/transaction-boundary.md` |
| Events, RPC, messaging | `details/event-rpc-message.md` |
| `@Cacheable`, ThreadLocal | `details/cache-concurrency.md` |
| Full review needed | `details/comprehensive-checklist.md` |

## Core AOP Rules

- **Only public methods proxied** - Private/protected/package-private NOT intercepted
- **Self-invocation bypasses proxy** - `this.method()` does NOT go through proxy
- **Final methods cannot be proxied**
- **Fix for self-invocation**: Inject self-reference: `@Autowired private SelfClass self;`

## Pre-Code Review (Always Do First)

Before analyzing code:
1. Read `application.yml`/`application.properties`
2. Read `bootstrap.yml`/`bootstrap.properties`
3. Identify: **annotation-based** OR **configuration-based** transactions
4. If config-based: find method patterns (e.g., `save*`, `update*`, `delete*`)
