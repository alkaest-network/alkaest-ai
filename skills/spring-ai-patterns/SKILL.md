---
name: spring-ai-patterns
description: Spring AI 2.x patterns for this project. ChatClient, Ollama structured output, prompt templates, BeanOutputConverter, retry. Use when writing or reviewing AI/intent adapter code.
---

# Spring AI Patterns

This project uses Spring AI 2.x (`spring-ai-starter-model-ollama`) with a local Ollama instance running Qwen3 4B. AI is used only for intent classification — all business logic remains deterministic Java.

---

## Dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

Managed by `spring-ai-bom`. Milestone repo required:

```xml
<repository>
    <id>spring-milestones</id>
    <url>https://repo.spring.io/milestone</url>
</repository>
```

---

## Configuration

```yaml
spring:
  ai:
    ollama:
      base-url: ${OLLAMA_BASE_URL:http://localhost:11434}
      chat:
        model: ${OLLAMA_MODEL:qwen3:4b}
        options:
          temperature: 0.0      # deterministic output for intent parsing
          num-predict: 256      # short structured responses only
```

Use `temperature: 0.0` for classification tasks — you want determinism, not creativity.

---

## ChatClient — Basic Usage

```java
@Component
public class OllamaIntentAdapter implements IntentPort {

    private final ChatClient chatClient;

    public OllamaIntentAdapter(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public IntentResult detect(String userMessage) {
        return chatClient.prompt()
                .system(SYSTEM_PROMPT)
                .user(userMessage)
                .call()
                .entity(IntentResult.class);  // structured output
    }
}
```

`ChatClient.Builder` is auto-configured. Inject the builder, call `.build()` to get a client — this allows decorating with defaults.

---

## Structured Output

Spring AI can map LLM output to a Java record automatically via `BeanOutputConverter`:

```java
// The record the model should fill in
public record IntentResult(
    Intent intent,
    String date,      // "2025-04-25" or null
    String time,      // "14:30" or null
    String professional  // name or null
) {
    public enum Intent {
        BOOK, CANCEL, LIST, UNKNOWN
    }
}
```

```java
// Automatic: .entity(IntentResult.class) handles format instructions + deserialization
IntentResult result = chatClient.prompt()
        .system(SYSTEM_PROMPT)
        .user(userMessage)
        .call()
        .entity(IntentResult.class);
```

Spring AI appends JSON format instructions to the prompt automatically. The model must return valid JSON — Qwen3 handles this well with `temperature: 0.0`.

### Manual structured output (more control)

```java
var converter = new BeanOutputConverter<>(IntentResult.class);

String response = chatClient.prompt()
        .system(SYSTEM_PROMPT + "\n\n" + converter.getFormat())
        .user(userMessage)
        .call()
        .content();

IntentResult result = converter.convert(response);
```

Use manual mode when you need to inspect the raw response or add custom format instructions alongside the generated ones.

---

## Prompt Engineering

### System prompt guidelines

```java
private static final String SYSTEM_PROMPT = """
        You are an intent classifier for a WhatsApp scheduling assistant.
        Respond ONLY with valid JSON matching the required schema.
        Do not add explanations, markdown, or extra text.

        Classify the user's intent as one of: BOOK, CANCEL, LIST, UNKNOWN.
        Extract date (ISO format YYYY-MM-DD), time (HH:mm), and professional name when present.
        If a field cannot be determined from the message, set it to null.
        """;
```

Rules for classification prompts:
- Explicitly say "ONLY JSON" — models add explanations by default
- List the exact enum values — don't say "one of the intents"
- Describe null behavior — models assume they should guess rather than null
- Keep it short — long system prompts degrade instruction-following on small models

### Template prompts (for parameterized tasks)

```java
private static final String BOOKING_CONTEXT_PROMPT = """
        Available slots for {professional} on {date}:
        {slots}

        The customer said: {message}

        Confirm which slot they want, or ask for clarification if ambiguous.
        """;

String rendered = new PromptTemplate(BOOKING_CONTEXT_PROMPT)
        .render(Map.of(
            "professional", professional.name(),
            "date", date.toString(),
            "slots", formatSlots(slots),
            "message", userMessage));
```

---

## Retry and Resilience

Ollama is local but can be slow or temporarily unavailable:

```java
@Component
public class OllamaIntentAdapter implements IntentPort {

    private final ChatClient chatClient;
    private final RetryTemplate retry;

    public OllamaIntentAdapter(ChatClient.Builder builder) {
        this.chatClient = builder.build();
        this.retry = RetryTemplate.builder()
                .maxAttempts(3)
                .fixedBackoff(500)
                .retryOn(Exception.class)
                .build();
    }

    public IntentResult detect(String userMessage) {
        return retry.execute(ctx -> chatClient.prompt()
                .system(SYSTEM_PROMPT)
                .user(userMessage)
                .call()
                .entity(IntentResult.class));
    }
}
```

Or use Spring's `@Retryable` if you have `spring-retry` on the classpath.

---

## Fallback Pattern

When classification fails or returns UNKNOWN, fall through to a deterministic fallback:

```java
public ConversationResponse handle(String phone, String userMessage) {
    IntentResult intent;
    try {
        intent = intentPort.detect(userMessage);
    } catch (Exception e) {
        log.warn("Intent detection failed for phone={}, falling back to UNKNOWN", phone);
        intent = new IntentResult(IntentResult.Intent.UNKNOWN, null, null, null);
    }

    return switch (intent.intent()) {
        case BOOK -> bookingFlow.handle(phone, intent);
        case CANCEL -> cancellationFlow.handle(phone, intent);
        case LIST -> listingFlow.handle(phone, intent);
        case UNKNOWN -> responseGenerator.unknownIntent();
    };
}
```

AI failure must never crash the conversation — always have a graceful UNKNOWN path.

---

## Testing

### Unit test (mock ChatClient)

```java
@ExtendWith(MockitoExtension.class)
class OllamaIntentAdapterTest {

    @Mock ChatClient.Builder builder;
    @Mock ChatClient chatClient;
    @Mock ChatClient.ChatClientRequestSpec spec;
    @Mock ChatClient.CallResponseSpec callSpec;

    OllamaIntentAdapter adapter;

    @BeforeEach
    void setUp() {
        when(builder.build()).thenReturn(chatClient);
        adapter = new OllamaIntentAdapter(builder);
    }

    @Test
    void detectsBookingIntent() {
        var expected = new IntentResult(BOOK, "2025-04-25", "14:00", "Maria");

        when(chatClient.prompt()).thenReturn(spec);
        when(spec.system(any())).thenReturn(spec);
        when(spec.user(any())).thenReturn(spec);
        when(spec.call()).thenReturn(callSpec);
        when(callSpec.entity(IntentResult.class)).thenReturn(expected);

        var result = adapter.detect("quero agendar com a Maria amanha as 2");

        assertThat(result.intent()).isEqualTo(BOOK);
    }
}
```

### Integration test

Integration tests against a real Ollama instance are slow and require the model to be pulled. Only run them manually or in a separate CI stage. Name them `*ManualIT.java` and exclude from the normal Failsafe run.

---

## Anti-patterns

```java
// ✗ Never put business logic in the AI adapter — classification only
public BookingResult detect(String message) {
    var intent = chatClient.prompt()...entity(IntentResult.class);
    return appointmentRepository.book(...);  // wrong: adapter should return intent, not act on it
}

// ✗ Never set temperature > 0 for structured classification
options:
  temperature: 0.7  // creative responses break JSON output

// ✗ Never skip the fallback
IntentResult intent = intentPort.detect(message);  // if this throws, conversation dies
```
