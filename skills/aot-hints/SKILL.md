---
name: aot-hints
description: GraalVM AOT hints for Spring Boot native images. Use when a class used via reflection fails at native runtime, when adding new serialization targets, or when reviewing native build errors.
---

# AOT Hints for GraalVM Native Images

When running as a GraalVM native image, the JVM's dynamic features (reflection, proxies, serialization) must be declared at build time. Spring Boot's AOT processing handles most of this automatically — you only need to intervene for cases it can't detect statically.

---

## When you need manual hints

- Classes instantiated via reflection that Spring doesn't scan (e.g., Jackson subtypes, custom deserializers)
- Classes used with `Class.forName()` or `getDeclaredMethods()` at runtime
- Resources loaded with `ClassLoader.getResourceAsStream()` that aren't in `src/main/resources`
- JDK proxies for interfaces not discovered by Spring's component scan
- Third-party libraries that haven't published GraalVM reachability metadata

---

## `@RegisterReflectionForBinding`

Registers a class and all its fields/methods for reflection — the most common case:

```java
// On the class itself
@RegisterReflectionForBinding
public record IntentResult(Intent intent, String date, String time, String professional) {
    public enum Intent { BOOK, CANCEL, LIST, UNKNOWN }
}

// Or on a configuration class if you don't own the target class
@Configuration
@RegisterReflectionForBinding({IntentResult.class, IntentResult.Intent.class})
public class AiConfiguration { ... }
```

Use this for any record or class that Spring AI's `BeanOutputConverter` deserializes from JSON — Jackson reflection calls happen at runtime.

---

## `RuntimeHintsRegistrar`

For fine-grained control — register specific reflection, resources, or proxies:

```java
public class ChatbotRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Reflection: register all members of a class
        hints.reflection()
                .registerType(IntentResult.class, MemberCategory.values())
                .registerType(IntentResult.Intent.class, MemberCategory.values());

        // Resources: register files loaded from classpath at runtime
        hints.resources()
                .registerPattern("prompt-templates/*.txt");

        // Serialization: for classes used with Java serialization (rare)
        hints.serialization()
                .registerType(SomeSerializableClass.class);
    }
}
```

### `MemberCategory` options

| Category | What it allows |
|----------|---------------|
| `DECLARED_FIELDS` | `getDeclaredFields()` |
| `PUBLIC_FIELDS` | `getFields()` |
| `DECLARED_METHODS` | `getDeclaredMethods()` |
| `PUBLIC_METHODS` | `getMethods()` |
| `DECLARED_CONSTRUCTORS` | `getDeclaredConstructors()` |
| `INVOKE_DECLARED_METHODS` | invoke via reflection |
| `INVOKE_PUBLIC_METHODS` | invoke public via reflection |

Use `MemberCategory.values()` (all) for Jackson-serialized classes. Use specific categories for tighter control.

---

## `@ImportRuntimeHints`

Wire your registrar into Spring's AOT processing:

```java
// On the application class
@SpringBootApplication
@ImportRuntimeHints(ChatbotRuntimeHints.class)
public class ChatbotApplication { ... }

// Or on a @Configuration class
@Configuration
@ImportRuntimeHints(ChatbotRuntimeHints.class)
public class NativeHintsConfiguration { ... }
```

---

## Testing AOT Hints

Test that your hints are correctly registered without running a full native build:

```java
@Test
void registersIntentResultForReflection() {
    var hints = new RuntimeHints();
    new ChatbotRuntimeHints().registerHints(hints, getClass().getClassLoader());

    assertThat(RuntimeHintsPredicates.reflection()
            .onType(IntentResult.class)
            .withMemberCategory(MemberCategory.INVOKE_DECLARED_METHODS))
            .accepts(hints);
}
```

Dependencies for hint tests:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

`RuntimeHintsPredicates` is in `spring-core` — no extra dependency needed.

---

## Build-time Initialization

For classes that must be initialized at build time (not runtime), declare them in the native plugin:

```xml
<!-- pom.xml — native profile -->
<buildArg>--initialize-at-build-time=org.slf4j,ch.qos.logback</buildArg>
```

This project already has SLF4J and Logback initialized at build time to work around HikariCP 7.x + reachability-metadata incompatibility. Don't remove this flag.

When adding new `--initialize-at-build-time` classes, verify they don't hold runtime state (DB connections, thread locals, system properties). Static loggers are safe; connection pools are not.

---

## Diagnosing Missing Hints

When a native image fails at runtime with `MissingReflectionDataException` or `ClassNotFoundException`:

1. Run with `-Ob` (O-zero-b) for quick feedback build: `./mvnw -Pnative -DskipTests package -Dnative.buildArgs=-Ob`
2. The error message names the missing class and the reflection call
3. Add a hint for that class
4. For third-party libraries: check [GraalVM Reachability Metadata repo](https://github.com/oracle/graalvm-reachability-metadata) for published configs

This project disables the reachability-metadata repository (`<metadataRepository><enabled>false</enabled>`) because Boot 4.0.5 ships HikariCP 7.x but the bundled repo only has HikariCP 6.x configs — stale configs cause UnsupportedFeatureException at build time. If a library needs its metadata, add it manually as a `RuntimeHintsRegistrar` instead.

---

## Anti-patterns

```java
// ✗ Don't add @RegisterReflectionForBinding to every class defensively
// Only add it to classes that are actually used via reflection at runtime

// ✗ Don't use Class.forName() in application code
// It will fail at native runtime unless you register the class first

// ✗ Don't load resources dynamically by constructing paths
String path = "templates/" + name + ".txt";
getClass().getResourceAsStream(path);  // native can't discover this pattern
// ✓ Register the pattern explicitly:
hints.resources().registerPattern("templates/*.txt");
```
