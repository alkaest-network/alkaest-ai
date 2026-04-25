# alkaest-ai

Claude Code agent definitions, skill modules, and reference guides for building software on the Alkaest stack.

These tools are general-purpose and reusable — not Alkaest-specific. Anyone building Java/Spring Boot or Node.js/TypeScript services with Claude Code can use them.

## What's inside

### agents/

Claude Code personas optimized for specific tasks:

| Agent | Purpose |
|---|---|
| `architect` | System design, trade-off analysis, architectural decisions |
| `code-architect` | Design-forward feature creation |
| `code-explorer` | Trace execution paths, map dependencies, understand existing code |
| `code-reviewer` | General code review |
| `java-reviewer` | Java-specific review — style, patterns, best practices |
| `typescript-reviewer` | TypeScript/Node.js-specific review |
| `tdd-guide` | Test-driven development — enforces write-tests-first discipline |

### references/

Technical reference guides for the stack:

| Reference | Purpose |
|---|---|
| `SPRING-BOOT-4.md` | Spring Boot 4.0 migration guide — real gotchas, not the official docs |
| `TEST.md` | Spring Boot testing best practices — TestContainers 2.0, Maven Failsafe traps |
| `LOGGING.md` | Structured logging guidance |
| `SECURITY.md` | Java security checklist (OWASP Top 10) |
| `DOCKER.md` | Docker and Compose reference |
| `GRAALVM.md` | GraalVM native image reference |

### skills/

Focused expertise modules for specific audit and review tasks:

| Skill | Purpose |
|---|---|
| `api-contract-review` | REST API design audit — HTTP semantics, versioning, backward compatibility |
| `architecture-review` | Package organization, dependency direction, module boundaries |
| `clean-code` | Code quality and readability |
| `concurrency-review` | Concurrency and threading best practices |
| `design-patterns` | Common design patterns for Java |
| `git-commit` | Commit message best practices |
| `logging-patterns` | Structured logging best practices |
| `maven-dependency-audit` | Dependency security and health |
| `performance-smell-detection` | Performance bottleneck detection |
| `security-audit` | OWASP Top 10 security audit for Java |
| `solid-principles` | SOLID design principles |
| `spring-boot-patterns` | Spring Boot best practices |
| `test-quality` | JUnit 5 + AssertJ test quality — 576 lines of practical guidance |
| `changelog-generator` | Automated changelog generation |

## How to use

Place this repository alongside your project or reference it from your Claude Code configuration.

Agents live in `~/.claude/agents/` or a project-local `.claude/agents/` directory.
Skills are invoked via `/skill-name` in Claude Code.
References are loaded as context when working on specific technical areas.
