---
name: aot-hints
description: GraalVM AOT hints for Spring Boot native images. @RegisterReflectionForBinding, RuntimeHintsRegistrar, @ImportRuntimeHints, testing hints.
---

Required when building native images (`-Pnative`) via GraalVM. Classes used via reflection at runtime must be registered at build time.

See `SKILL.md` for complete patterns.
