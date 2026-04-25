---
name: observability-patterns
description: Micrometer counters/timers, low-cardinality tags, OTLP traces, MDC correlation IDs. Use when adding metrics, traces, or structured log context to the application.
---

# Observability Patterns

Stack: Micrometer → Grafana Alloy (OTLP receiver) → Prometheus (metrics) + Tempo (traces) + Grafana (dashboards).

---

## Dependencies (already in pom.xml)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

---

## Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,info
  metrics:
    tags:
      application: chatbot-core-api   # global tag on all metrics
  tracing:
    sampling:
      probability: 1.0                # 100% in dev; reduce in prod if noisy

spring:
  application:
    name: chatbot-core-api

# OTLP exporter → Alloy
management:
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318}/v1/traces
```

---

## Counters

Use counters for events you want to count over time.

```java
@Component
public class AppointmentService {

    private final Counter bookingCounter;
    private final Counter cancellationCounter;
    private final Counter bookingFailureCounter;

    public AppointmentService(MeterRegistry registry, ...) {
        this.bookingCounter = Counter.builder("chatbot.appointments.booked")
                .description("Total appointments successfully booked")
                .register(registry);
        this.cancellationCounter = Counter.builder("chatbot.appointments.cancelled")
                .description("Total appointments cancelled")
                .register(registry);
        this.bookingFailureCounter = Counter.builder("chatbot.appointments.booking_failed")
                .description("Booking attempts that failed (no slot, calendar error, etc.)")
                .register(registry);
    }

    @Transactional
    public BookingResult book(BookingRequest request) {
        var result = doBook(request);
        if (result.isSuccess()) {
            bookingCounter.increment();
        } else {
            bookingFailureCounter.increment();
        }
        return result;
    }
}
```

---

## Timers

Use timers for durations — especially external calls (Ollama, Google Calendar).

```java
@Component
public class OllamaIntentAdapter implements IntentPort {

    private final Timer intentTimer;
    private final ChatClient chatClient;

    public OllamaIntentAdapter(MeterRegistry registry, ChatClient.Builder builder) {
        this.intentTimer = Timer.builder("chatbot.intent.classification.duration")
                .description("Time to classify intent via Ollama")
                .register(registry);
        this.chatClient = builder.build();
    }

    public IntentResult detect(String message) {
        return intentTimer.record(() ->
            chatClient.prompt()
                    .system(SYSTEM_PROMPT)
                    .user(message)
                    .call()
                    .entity(IntentResult.class));
    }
}
```

### Timer with tags (low cardinality only)

```java
Timer.builder("chatbot.calendar.api.duration")
        .tag("operation", "create_event")   // ✓ low cardinality: fixed set of values
        .tag("status", "success")           // ✓ low cardinality: success | failure
        // ✗ never: .tag("phone", phone)    — high cardinality, kills Prometheus
        .register(registry)
        .record(duration, TimeUnit.MILLISECONDS);
```

**Low cardinality rule**: tags must have a small, bounded set of values. Phone numbers, user IDs, request IDs — never as tags. Put those in log MDC or trace attributes instead.

---

## Tracing

Micrometer + OTel bridge adds traces automatically to Spring MVC requests and JdbcClient calls. You only need manual spans for external calls that aren't auto-instrumented.

```java
@Component
public class GoogleCalendarBookingAdapter implements BookingCalendarPort {

    private final Tracer tracer;
    private final Calendar calendarClient;

    public GoogleCalendarBookingAdapter(Tracer tracer, Calendar calendarClient) {
        this.tracer = tracer;
        this.calendarClient = calendarClient;
    }

    public void createEvent(Appointment appt) {
        Span span = tracer.nextSpan().name("google-calendar.create-event").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("professional.id", String.valueOf(appt.professionalId()));
            calendarClient.events().insert(calendarId, buildEvent(appt)).execute();
        } catch (Exception e) {
            span.error(e);
            throw new CalendarException("Failed to create calendar event", e);
        } finally {
            span.end();
        }
    }
}
```

### Trace + log correlation

With `micrometer-tracing-bridge-otel`, `traceId` and `spanId` are automatically added to MDC. Add them to your log format:

```xml
<!-- logback-spring.xml -->
<pattern>%d{HH:mm:ss} %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n</pattern>
```

---

## Structured Logging (MDC)

Add business context to logs so Loki/Grafana can filter by it:

```java
@Component
public class ConversationService {

    public ConversationResponse handle(String phone, String message) {
        MDC.put("customer.phone.hash", hash(phone));   // hash PII before logging
        MDC.put("conversation.state", currentState.name());
        try {
            log.info("Handling message");
            // ... processing
        } finally {
            MDC.clear();
        }
    }

    private String hash(String phone) {
        // SHA-256 first 8 chars — enough to correlate, not enough to identify
        return DigestUtils.sha256Hex(phone).substring(0, 8);
    }
}
```

**Never log raw PII** (phone numbers, names) — hash or mask before logging.

---

## Health Endpoint

`/actuator/health` is the only health check available in the distroless container (no shell/curl inside). Kubernetes uses HTTP GET against this endpoint.

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized
  health:
    db:
      enabled: true      # checks postgres connection
    ollama:
      enabled: false     # Ollama not critical for health — bookings can still show existing slots
```

---

## Prometheus Naming Conventions

Follow Prometheus naming conventions:

| Convention | Example |
|------------|---------|
| snake_case | `chatbot_appointments_booked_total` |
| `_total` suffix for counters | `chatbot_intent_classifications_total` |
| `_duration_seconds` for timers | `chatbot.intent.classification.duration` → auto-suffixed |
| Namespace prefix | `chatbot_` |

Micrometer handles unit suffixes automatically (`.duration` becomes `_duration_seconds`).

---

## Anti-patterns

```java
// ✗ High-cardinality tag — kills Prometheus memory
Timer.builder("chatbot.requests")
        .tag("phone", phone)       // millions of unique values
        .register(registry);

// ✗ Logging PII in plain text
log.info("Processing message from {}", phone);  // use hash

// ✗ Catching and swallowing exceptions without recording on span
try {
    calendarClient.createEvent(event);
} catch (Exception e) {
    log.error("error", e);  // span doesn't know this failed
}

// ✓ Record on span
span.error(e);  // marks span as error in Tempo
```
