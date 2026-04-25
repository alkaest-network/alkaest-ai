---
name: jdbc-client-patterns
description: Spring JdbcClient patterns for this project. Named params, row mappers, batch ops, transactions. No JPA/ORM. Use when writing or reviewing repository adapters.
---

# JdbcClient Patterns

`JdbcClient` is Spring's fluent JDBC client (available since Spring 6.1 / Boot 3.2). This project uses it exclusively — no JPA, no Spring Data repositories.

---

## Basic Setup

```java
@Repository
public class JdbcAppointmentRepository implements AppointmentRepository {

    private final JdbcClient jdbc;

    public JdbcAppointmentRepository(JdbcClient jdbc) {
        this.jdbc = jdbc;
    }
}
```

`JdbcClient` is auto-configured from `spring-boot-starter-jdbc`. No extra beans needed.

---

## Query Patterns

### Single result

```java
// Optional — preferred for lookups that may not exist
public Optional<Appointment> findById(long id) {
    return jdbc.sql("SELECT * FROM appointments WHERE id = :id")
            .param("id", id)
            .query(this::mapRow)
            .optional();
}

// Single — throws if 0 or >1 results; use only when you're certain it exists
public Appointment getById(long id) {
    return jdbc.sql("SELECT * FROM appointments WHERE id = :id")
            .param("id", id)
            .query(this::mapRow)
            .single();
}
```

### List

```java
public List<Appointment> findByPhone(String phone) {
    return jdbc.sql("""
            SELECT * FROM appointments
            WHERE customer_phone = :phone
            ORDER BY appointment_time
            """)
            .param("phone", phone)
            .query(this::mapRow)
            .list();
}
```

### Scalar

```java
public int countActive(long professionalId) {
    return jdbc.sql("""
            SELECT COUNT(*) FROM appointments
            WHERE professional_id = :id AND status = 'BOOKED'
            """)
            .param("id", professionalId)
            .query(Integer.class)
            .single();
}
```

---

## Mutation Patterns

### INSERT

```java
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
```

### INSERT returning generated key

```java
public long insert(Appointment appt) {
    return jdbc.sql("""
            INSERT INTO appointments (customer_phone, professional_id, appointment_time, status)
            VALUES (:phone, :professionalId, :time, :status)
            """)
            .param("phone", appt.customerPhone())
            .param("professionalId", appt.professionalId())
            .param("time", appt.appointmentTime())
            .param("status", appt.status().name())
            .update(keyHolder, new String[]{"id"})
            // prefer a sequence + RETURNING instead:
            // use RETURNING id in SQL and .query(Long.class).single()
            ;
}

// Cleaner: use RETURNING
public long insertReturning(Appointment appt) {
    return jdbc.sql("""
            INSERT INTO appointments (customer_phone, professional_id, appointment_time, status)
            VALUES (:phone, :professionalId, :time, :status)
            RETURNING id
            """)
            .param("phone", appt.customerPhone())
            .param("professionalId", appt.professionalId())
            .param("time", appt.appointmentTime())
            .param("status", appt.status().name())
            .query(Long.class)
            .single();
}
```

### UPDATE

```java
public void updateStatus(long id, AppointmentStatus newStatus) {
    int updated = jdbc.sql("""
            UPDATE appointments SET status = :status, updated_at = now()
            WHERE id = :id
            """)
            .param("status", newStatus.name())
            .param("id", id)
            .update();

    if (updated == 0) {
        throw new AppointmentNotFoundException(id);
    }
}
```

---

## Row Mapper

```java
private Appointment mapRow(ResultSet rs, int rowNum) throws SQLException {
    return new Appointment(
            rs.getLong("id"),
            rs.getString("customer_phone"),
            rs.getLong("professional_id"),
            rs.getObject("appointment_time", LocalDateTime.class),
            AppointmentStatus.valueOf(rs.getString("status")));
}
```

Rules:
- One `mapRow` method per entity — private, named consistently
- Use `rs.getObject(col, SomeType.class)` for Java time types — never `rs.getTimestamp()`
- Store enum names as `VARCHAR` in DB; convert with `Enum.valueOf(rs.getString(col))`
- Never use `BeanPropertyRowMapper` with records — it relies on setters which records don't have

---

## Transactions

`@Transactional` belongs on **service methods**, not repositories:

```java
@Service
public class AppointmentService {

    @Transactional
    public BookingResult book(BookingRequest request) {
        // both writes happen in same transaction
        appointmentRepository.save(appointment);
        auditRepository.logBooking(appointment);
        return BookingResult.booked(appointment);
    }

    // read-only: no write intent, allows read replicas
    @Transactional(readOnly = true)
    public List<Appointment> listForProfessional(long professionalId) {
        return appointmentRepository.findByProfessional(professionalId);
    }
}
```

Repository methods participate in the caller's transaction automatically — don't annotate them.

---

## Batch Operations

```java
public void markExpired(List<Long> ids) {
    // JdbcClient doesn't have first-class batch — use JdbcTemplate for batch
    // or run individual updates inside a @Transactional service method
    ids.forEach(id -> jdbc.sql("UPDATE appointments SET status = 'EXPIRED' WHERE id = :id")
            .param("id", id)
            .update());
}
```

For true batch performance (thousands of rows), inject `JdbcTemplate` alongside `JdbcClient`:

```java
public void batchInsert(List<Appointment> appts) {
    jdbcTemplate.batchUpdate(
            "INSERT INTO appointments (customer_phone, status) VALUES (?, ?)",
            appts,
            50,
            (ps, a) -> {
                ps.setString(1, a.customerPhone());
                ps.setString(2, a.status().name());
            });
}
```

---

## Named vs Positional Parameters

Always use named parameters (`:name`). Never use `?`:

```java
// ✓ named — readable, order-independent
.param("phone", phone).param("status", status.name())

// ✗ positional — order-sensitive, error-prone
// jdbc.sql("SELECT * FROM appointments WHERE phone = ? AND status = ?")
```

---

## Anti-patterns

```java
// ✗ Never use BeanPropertyRowMapper with records
.query(BeanPropertyRowMapper.newInstance(Appointment.class))

// ✗ Never concatenate values into SQL — SQL injection
.sql("SELECT * FROM appointments WHERE phone = '" + phone + "'")

// ✗ Never put @Transactional on repository methods
@Transactional  // wrong place
public void save(Appointment appt) { ... }

// ✗ Never suppress the row count check on UPDATE
jdbc.sql("UPDATE ...").update();  // OK if you don't care
// but always check when update must affect exactly one row
if (jdbc.sql("UPDATE ...").update() == 0) { throw new NotFoundException(...); }
```
