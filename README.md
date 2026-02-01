# Claude Code Skills Marketplace

A curated collection of skills for [Claude Code](https://claude.ai/code) - the AI-powered development assistant.

> **Claude Code Skills** are specialized prompts and patterns that enhance code review, development assistance, and productivity.

## Available Skills

### [Spring AOP Prompt](skills/spring-aop-prompt/)
**Version:** 1.0.0 | **Author:** zml

Spring AOP detection and guidance for code review. Helps identify:
- AOP proxy behavior issues
- Self-invocation problems
- Transaction boundary issues
- Event/RPC/messaging integration patterns

**Auto-triggers:** `@Transactional`, `@Aspect`, `@Async`, `@Cacheable`

---

### [Safe Java Code](skills/safe-java-code/)
**Version:** 1.2.0 | **Author:** zml

Comprehensive Java code safety checker for both general applications and high-load financial systems. Detects:
- **Distributed systems:** Lock/transaction ordering, SAGA pattern, cache consistency
- **Concurrency:** synchronized blocks, ReentrantLock, ThreadLocal, ConcurrentHashMap
- **Race conditions:** Check-Then-Act, double-checked locking, unsafe publication
- **Database safety:** Unchecked updates, optimistic locking, N+1 queries
- **Async execution:** Thread pool shutdown, exception handling, context passing
- **Security vulnerabilities:** SQL injection, IDOR, input validation, resource leaks
- **Basic code issues:** NPE, array bounds, exception handling

**Auto-triggers:** `synchronized`, `@Transactional`, `ExecutorService`, `redisLock`, `.get(`

---

## Skill Structure

Each skill follows this standard structure:

```
skill-name/
├── SKILL.md                     # Core skill logic (main entry point)
├── skill.yaml                   # Metadata and auto-trigger configuration
├── README.md                    # User-facing documentation
├── CLAUDE.md                    # Skill-specific guidance (optional)
└── details/                     # Progressive loading modules
    ├── module-name.md
    └── comprehensive-checklist.md
```

## Using Skills

Skills are auto-loaded by Claude Code when:

1. **File patterns match** - e.g., `**/service/**/*.java`
2. **Code triggers are found** - e.g., `@Transactional` annotation
3. **Annotation triggers are present** - e.g., `@Async`
4. **Manual invocation** - Use the `/skill` command

## Progressive Loading

Skills use progressive loading to minimize token usage:

```
SKILL.md (always loaded, ~90 lines)
├── details/module-1.md (loaded when triggers match)
├── details/module-2.md (loaded when triggers match)
└── details/comprehensive-checklist.md (loaded for full review)
```

## Contributing

To add a new skill:

1. **Create the directory:** `mkdir -p skills/new-skill/{details,references}`
2. **Create SKILL.md:** Main entry point with core detection logic
3. **Create skill.yaml:** Metadata including auto-trigger and progressive loading config
4. **Create detail modules:** Organized by category under `details/`
5. **Create README.md:** User-facing documentation

See [CLAUDE.md](CLAUDE.md) for detailed guidance.

## License

See individual skill directories for license information.
