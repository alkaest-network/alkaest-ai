---
name: spring-boot-patterns
description: Spring Boot 4.x patterns for this project. Use when creating controllers, services, repositories, or reviewing Spring Boot code. Covers JdbcClient (not JPA), package-by-feature structure, @ConditionalOnProperty, records, no Lombok.
---

# Spring Boot Patterns Skill

Patterns for this project: Spring Boot 4.x, Java 25, `JdbcClient`, package-by-feature, no Lombok, no JPA/Hibernate.

## Project Structure (package-by-feature)

```
src/main/java/org/alkaest/chatbot/
├── ChatbotApplication.java
├── scheduling/                    # feature: booking flow
│   ├── domain/
│   │   ├── Appointment.java       # record or final class
│   │   └── AppointmentStatus.java
│   ├── ports/
│   │   ├── AppointmentRepository.java   # interface
│   │   └── BookingCalendarPort.java     # interface
│   ├── adapters/
│   │   ├── JdbcAppointmentRepository.java
│   │   └── GoogleCalendarBookingAdapter.java
│   └── AppointmentService.java
├── conversation/
│   ├── domain/
│   ├── ports/
│   └── adapters/
└── notification/
    └── domain/
        └── ResponseGenerator.java  # pure domain, no Spring deps
```

No `controller/`, `service/`, `repository/` top-level packages — features own their entire slice.

---

## Controller Patterns

```java
@RestController
@RequestMapping("/webhook/whatsapp")
public class WhatsAppWebhookController {

    private final ConversationService conversationService;

    public WhatsAppWebhookController(ConversationService conversationService) {
        this.conversationService = conversationService;
    }

    @PostMapping
    public ResponseEntity<Void> receive(@Valid @RequestBody InboundMessageRequest request) {
        conversationService.handle(request);
        return ResponseEntity.ok().build();
    }
}
```

- Constructor injection, no `@Autowired`, no Lombok `@RequiredArgsConstructor`
- `@Valid` on request body, always
- Return `ResponseEntity<Void>` for fire-and-forget endpoints
- Never return entity/domain objects directly — use records as response types

### Conditional controllers

```java
@RestController
@ConditionalOnProperty(name = "app.calendar.enabled", havingValue = "true")
public class GoogleCalendarWebhookController { ... }
```

Tests for conditional controllers must activate the property:

```java
@WebMvcTest(GoogleCalendarWebhookController.class)
@TestPropertySource(properties = "app.calendar.enabled=true")
class GoogleCalendarWebhookControllerTest { ... }
```

---

## Repository Patterns (JdbcClient only — no JPA)

```java
@Repository
public class JdbcAppointmentRepository implements AppointmentRepository {

    private final JdbcClient jdbc;

    public JdbcAppointmentRepository(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<Appointment> findById(long id) {
        return jdbc.sql("SELECT * FROM appointments WHERE id = :id")
                .param("id", id)
                .query(this::mapRow)
                .optional();
    }

    public List<Appointment> findByPhoneAndStatus(String phone, AppointmentStatus status) {
        return jdbc.sql("""
                SELECT * FROM appointments
                WHERE customer_phone = :phone AND status = :status
                ORDER BY appointment_time
                """)
                .param("phone", phone)
                .param("status", status.name())
                .query(this::mapRow)
                .list();
    }

    public void save(Appointment appt) {
        jdbc.sql("""
                INSERT INTO appointments (customer_phone, professional_id, appointment_time, status)
                VALUES (:phone, :professionalId, :time, :status)
                """)
                .param("phone", appt.customerPhone())
                .param("professionalId", appt.professionalId())
                .param("time", appt.appointmentTime())
                .param("status", appt.status().name())
                .update();
    }

    private Appointment mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new Appointment(
                rs.getLong("id"),
                rs.getString("customer_phone"),
                rs.getLong("professional_id"),
                rs.getObject("appointment_time", LocalDateTime.class),
                AppointmentStatus.valueOf(rs.getString("status")));
    }
}
```

### JdbcClient rules
- Always use named params (`:name`), never positional `?`
- `.query(rowMapper).list()` for collections, `.optional()` for single
- `.update()` for INSERT/UPDATE/DELETE (returns row count)
- Raw `rowMapper` method per entity — no `BeanPropertyRowMapper` (fragile with records)
- `@Transactional` on service methods, not repositories

---

## Service Patterns

```java
@Service
public class AppointmentService {

    private final AppointmentRepository appointmentRepository;
    private final BookingCalendarPort calendarPort;

    public AppointmentService(
            AppointmentRepository appointmentRepository,
            BookingCalendarPort calendarPort) {
        this.appointmentRepository = appointmentRepository;
        this.calendarPort = calendarPort;
    }

    @Transactional
    public BookingResult book(BookingRequest request) {
        // domain logic here, not in controller or repository
        var slot = calendarPort.findAvailableSlot(request.professionalId(), request.date());
        if (slot.isEmpty()) {
            return BookingResult.noSlot();
        }
        var appointment = new Appointment(...);
        appointmentRepository.save(appointment);
        calendarPort.createEvent(appointment);
        return BookingResult.booked(appointment);
    }
}
```

- Services own `@Transactional` — repositories are transaction-participants, not owners
- Return sealed domain result types instead of throwing for expected failures
- No `readOnly = true` default — add it explicitly per method when needed

---

## DTO Patterns (Java records)

```java
// Inbound HTTP request
public record InboundMessageRequest(
    @NotBlank String phone,
    @NotBlank String message,
    @NotNull Instant timestamp
) {}

// Outbound HTTP response
public record BookingConfirmation(
    long appointmentId,
    String professionalName,
    LocalDate date,
    LocalTime time
) {}
```

No MapStruct, no ModelMapper — map manually in service or a static factory method on the record:

```java
public record BookingConfirmation(...) {
    static BookingConfirmation from(Appointment a, Professional p) {
        return new BookingConfirmation(a.id(), p.name(), a.date(), a.time());
    }
}
```

---

## Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .toList();
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_ERROR", errors.toString()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled error", ex);
        return ResponseEntity.internalServerError()
                .body(new ErrorResponse("INTERNAL_ERROR", "Unexpected error"));
    }
}

public record ErrorResponse(String code, String message) {}
```

Use `LoggerFactory.getLogger(...)` — no `@Slf4j` (no Lombok).

---

## Configuration Patterns

```java
// Bind typed config from application.yml
@ConfigurationProperties(prefix = "app.calendar")
public record CalendarProperties(
    boolean enabled,
    String webhookUrl,
    String credentials
) {}

// Register it
@SpringBootApplication
@ConfigurationPropertiesScan
public class ChatbotApplication { ... }
```

```yaml
# application.yml
app:
  calendar:
    enabled: ${CALENDAR_ENABLED:false}
    webhook-url: ${CALENDAR_WEBHOOK_URL:}
    credentials: ${CALENDAR_CREDENTIALS:}
```

Feature flags via `@ConditionalOnProperty`:

```java
@Bean
@ConditionalOnProperty(name = "app.calendar.enabled", havingValue = "true")
public GoogleCalendarBookingAdapter calendarAdapter(...) { ... }
```

---

## Testing Quick Reference

| Test type | Annotation | When |
|-----------|-----------|------|
| Unit | `@ExtendWith(MockitoExtension.class)` | pure logic, no Spring context |
| Web slice | `@WebMvcTest(Controller.class)` | controller + serialization only |
| JDBC integration | `@JdbcTest` + `@Testcontainers` | repository against real DB |
| Full integration | `@SpringBootTest` + `@Testcontainers` | end-to-end flow |

Integration tests use `*IT.java` naming so Failsafe runs them on `mvn verify`, not `mvn test`.

```java
// JDBC integration test pattern
@JdbcTest
@AutoConfigureTestDatabase(replace = NONE)
@Testcontainers
class JdbcAppointmentRepositoryIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:18");

    @Autowired JdbcClient jdbc;
    JdbcAppointmentRepository repo;

    @BeforeEach
    void setUp() { repo = new JdbcAppointmentRepository(jdbc); }
}
```
