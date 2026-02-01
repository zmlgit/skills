# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository hosts **Claude Code Skills** - specialized prompts and patterns for code review and development assistance.

## Repository Structure

```
skills/                    # Claude Code Skills Marketplace
├── skills/                # All skill packages
│   ├── spring-aop-prompt/    # Spring AOP detection and guidance
│   ├── safe-java-code/       # Java code safety checker (includes distributed systems)
│   └── [other skills]/
├── CLAUDE.md              # This file - project guidance
└── README.md              # Skills marketplace homepage
```

## Skill Directory Structure

Each skill follows this standard structure:

```
skill-name/
├── SKILL.md                     # Core skill logic (main entry point)
├── skill.yaml                   # Skill metadata and auto-trigger config
├── README.md                    # Documentation
├── CLAUDE.md                    # Skill-specific guidance (optional)
└── details/                     # Progressive loading modules
    ├── module-name.md
    └── comprehensive-checklist.md
```

## Creating a New Skill

When creating a new skill:

1. **Create the directory**: `mkdir -p new-skill/{details,references}`
2. **Create SKILL.md**: Main entry point with core detection logic
3. **Create skill.yaml**: Metadata including:
   - `name`, `description`, `version`, `author`, `tags`
   - `auto_trigger`: File patterns, code triggers, annotation triggers
   - `progressive_loading`: Core module and detail modules with triggers
4. **Create detail modules**: Organized by category/concern under `details/`
5. **Create README.md**: User-facing documentation

## Skill YAML Configuration

Key sections in `skill.yaml`:

```yaml
# Auto-triggers for /review and code-reviewer agents
auto_trigger:
  enabled: true
  languages: [java, typescript, ...]
  file_patterns: ["**/service/**/*.java"]
  code_triggers: ["synchronized", "@Transactional"]
  annotation_triggers: ["@Transactional", "@Async"]

# Progressive loading minimizes token usage
progressive_loading:
  enabled: true
  core_module: "SKILL.md"
  detail_modules:
    - file: "details/module.md"
      triggers: ["keyword1", "keyword2"]
```

## Common Development Commands

No build/test commands - this is a documentation-only repository containing skill definitions.

## Skill Integration

Skills are auto-loaded by Claude Code when:
1. File patterns match (`**/service/**/*.java`)
2. Code triggers are found (`@Transactional`)
3. Annotation triggers are present
4. User explicitly invokes via `/skill` command

## Existing Skills

### spring-aop-prompt
- Spring AOP proxy behavior detection
- Self-invocation issues
- Transaction boundary problems
- Progressive loading architecture

### safe-java-code
- **General Java code safety** + **Distributed systems** (merged from java-transaction-checker)
- Detects:
  - **Distributed systems**: Lock/transaction ordering, SAGA pattern, cache consistency, lock renewal
  - **Concurrency**: synchronized, ReentrantLock, ThreadLocal, ConcurrentHashMap
  - **Race conditions**: Check-Then-Act, double-checked locking, unsafe publication
  - **Database safety**: Unchecked updates, optimistic locking, N+1 queries
  - **Async execution**: Thread pool shutdown, exception handling, context passing
  - **Vulnerabilities**: SQL injection, IDOR, input validation, resource leaks
- Context: General Java apps + high-load financial systems
