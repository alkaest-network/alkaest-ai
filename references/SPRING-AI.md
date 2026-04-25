# Spring AI 2.x Reference

Spring AI 2.x with Ollama. Used in this project for local intent classification (Qwen3 4B).

---

## Dependency Setup

```xml
<!-- BOM — manages all spring-ai versions -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>2.0.0-M4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Ollama starter -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

Spring AI 2.x is milestone — requires the Spring milestones repository:

```xml
<repository>
    <id>spring-milestones</id>
    <name>Spring Milestones</name>
    <url>https://repo.spring.io/milestone</url>
    <snapshots><enabled>false</enabled></snapshots>
</repository>
```

---

## Auto-configuration

`spring-ai-starter-model-ollama` auto-configures:
- `OllamaChatModel` — the raw model client
- `ChatClient.Builder` — the fluent client builder (preferred API)
- `OllamaApi` — low-level HTTP client (rarely needed directly)

Inject `ChatClient.Builder`, not `ChatClient` directly, so you can add defaults before building.

---

## ChatClient API

### Build with defaults

```java
ChatClient client = builder
        .defaultSystem("You are a scheduling assistant. Reply in JSON only.")
        .defaultOptions(OllamaOptions.builder()
                .temperature(0.0)
                .numPredict(256)
                .build())
        .build();
```

### Prompt fluent API

```java
// Simple call
String text = client.prompt()
        .user("What day is tomorrow?")
        .call()
        .content();

// With system override
IntentResult result = client.prompt()
        .system(systemPrompt)
        .user(userMessage)
        .call()
        .entity(IntentResult.class);  // structured output

// Streaming (not used in this project — synchronous only)
client.prompt()
        .user(message)
        .stream()
        .content()
        .subscribe(chunk -> ...);
```

### Response types

| Method | Returns |
|--------|---------|
| `.call().content()` | `String` — raw text |
| `.call().entity(MyRecord.class)` | `MyRecord` — JSON deserialized |
| `.call().chatResponse()` | `ChatResponse` — full response with metadata |
| `.stream().content()` | `Flux<String>` — streaming chunks |

---

## Structured Output

### `BeanOutputConverter`

Spring AI converts the LLM's response to a Java record:

```java
// 1. Define the output record
public record IntentResult(
    Intent intent,
    String date,
    String time,
    String professional
) {
    public enum Intent { BOOK, CANCEL, LIST, UNKNOWN }
}

// 2. Use .entity() — format instructions injected automatically
IntentResult result = client.prompt()
        .system(SYSTEM_PROMPT)
        .user(userMessage)
        .call()
        .entity(IntentResult.class);
```

Spring AI appends a format instruction to the prompt telling the model to respond in JSON matching the record's schema. Works best with `temperature: 0.0`.

### Manual conversion

```java
var converter = new BeanOutputConverter<>(IntentResult.class);
// converter.getFormat() returns the JSON schema instruction string

String raw = client.prompt()
        .system(SYSTEM_PROMPT + "\n\n" + converter.getFormat())
        .user(userMessage)
        .call()
        .content();

IntentResult result = converter.convert(raw);
```

---

## Ollama Options

```java
OllamaOptions options = OllamaOptions.builder()
        .model("qwen3:4b")
        .temperature(0.0)        // 0.0 = deterministic; 1.0 = creative
        .numPredict(256)         // max output tokens
        .topK(1)                 // further increases determinism
        .build();
```

Override per-call:

```java
client.prompt()
        .options(OllamaOptions.builder().temperature(0.0).build())
        .user(message)
        .call()
        .entity(IntentResult.class);
```

---

## Configuration Properties

```yaml
spring:
  ai:
    ollama:
      base-url: ${OLLAMA_BASE_URL:http://localhost:11434}
      chat:
        model: ${OLLAMA_MODEL:qwen3:4b}
        options:
          temperature: 0.0
          num-predict: 256
```

| Property | Default | Notes |
|----------|---------|-------|
| `spring.ai.ollama.base-url` | `http://localhost:11434` | Set via `OLLAMA_BASE_URL` env |
| `spring.ai.ollama.chat.model` | `mistral` | Set via `OLLAMA_MODEL` env |
| `spring.ai.ollama.chat.options.temperature` | `0.7` | Override to `0.0` for classification |
| `spring.ai.ollama.chat.options.num-predict` | `-1` (unlimited) | Cap for structured output tasks |

---

## PromptTemplate

For parameterized prompts:

```java
import org.springframework.ai.chat.prompt.PromptTemplate;

String rendered = new PromptTemplate("""
        Available slots on {date} for {professional}:
        {slots}
        """)
        .render(Map.of(
            "date", date.toString(),
            "professional", name,
            "slots", formattedSlots));
```

`{variable}` syntax. Throws if a variable is missing from the map.

---

## Testing

### Mock ChatClient (unit tests)

```java
@Mock ChatClient.Builder builder;
@Mock ChatClient chatClient;
@Mock ChatClient.ChatClientRequestSpec spec;
@Mock ChatClient.CallResponseSpec call;

when(builder.build()).thenReturn(chatClient);
when(chatClient.prompt()).thenReturn(spec);
when(spec.system(any())).thenReturn(spec);
when(spec.user(any())).thenReturn(spec);
when(spec.call()).thenReturn(call);
when(call.entity(IntentResult.class)).thenReturn(expectedResult);
```

### Real Ollama (integration tests)

```java
// Only for manual/local runs — Ollama must be running with the model pulled
// Name: *ManualIT.java so Failsafe skips in CI
@SpringBootTest
@TestPropertySource(properties = "spring.ai.ollama.base-url=http://localhost:11434")
class OllamaIntentAdapterManualIT { ... }
```

---

## Known Issues / Quirks (Spring AI 2.0.0-M4)

- **Milestone API instability**: APIs may change between M4 and GA. Pin the BOM version.
- **`@ConditionalOnProperty` + auto-config**: `ChatClient.Builder` is always auto-configured even when Ollama is disabled. Guard the `OllamaIntentAdapter` bean instead of the `ChatClient` bean.
- **Qwen3 thinking tokens**: Qwen3 4B can emit `<think>...</think>` blocks before JSON. Use `temperature: 0.0` and `top_k: 1` to suppress. If they still appear, strip them before passing to `BeanOutputConverter`.
- **Model not pulled**: If Ollama returns 404, the model isn't pulled. Run `ollama pull qwen3:4b` on the host first.
