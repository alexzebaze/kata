# Code Review — Carrefour Java Kata

## Summary

The project implements a subscription management service with Spring Boot, JPA, and Kafka. The overall structure is coherent and the core business logic (create + change subscription with proration) is present. However, several issues affect correctness, completeness, and maintainability.

---

## 1. Code Quality

### Readability

**Positive points**
- Package structure is clear (`controller`, `service`, `dao`, `dto`, `model`, `event`).
- Use of records for DTOs (`CreateSubscriptionRequest`, `ChangeSubscriptionRequest`) is idiomatic Java.
- SQL init script is clean and self-contained.

**Issues**

#### Lombok declared but not used on entities
`@Data` is imported via Lombok on `Subscription`, `SubscriptionPlan`, and `Customer`, yet all getters/setters are written manually. This is contradictory and doubles the boilerplate. Either use `@Data` (or `@Getter`/`@Setter`) and remove the manual methods, or drop Lombok entirely.

```java
// Current: @AllArgsConstructor + @NoArgsConstructor + 10 manual getters/setters
// Fix: add @Getter @Setter (or @Data) and delete the manual methods
@Entity
@Table(name = "subscription")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
public class Subscription { ... }
```

#### `SubscriptionStatus2.java` is a duplicate
`SubscriptionStatus2` is byte-for-byte identical to `SubscriptionPlan` and maps to the same `@Table(name = "subscription_plan")`. It is dead code and will cause a Hibernate mapping conflict at startup. It must be deleted.

#### `PlanType` has a spurious `@AllArgsConstructor`
Enums cannot have a Lombok-generated all-args constructor (there are no fields). The annotation is noise and should be removed.

#### Commented-out code left in production files
`SubscriptionService` has two commented-out blocks (`eventPublisher` injection and its call). `SubscriptionEventPublisher` itself is commented out with `//@Component`. Dead code should not be committed; either implement it or remove it.

#### `SubscriptionResponse` is never used
The DTO exists but the controller returns raw `Subscription` entities. This exposes JPA internals (lazy proxies, bidirectional cycles) to the HTTP layer and makes the API contract implicit. The controller should return `SubscriptionResponse`.

#### Unused import in DTOs
`ChangeSubscriptionRequest` and `CreateSubscriptionRequest` both import `fr.carrefour.kata.model.PlanType` without using it.

---

### Maintainability

#### `renewExpiredSubscriptions()` is not implemented
The method is declared in the interface, annotated with `@Scheduled`, and returns `null`. This is the core of the "automatic renewal" user story — the primary requirement of the kata. A minimal implementation would be:

```java
@Scheduled(cron = "0 0 0 * * *")
@Transactional
public void renewExpiredSubscriptions() {
    LocalDate today = LocalDate.now();
    List<Subscription> expired = subscriptionDao
        .findByStatusAndAutoRenewTrueAndEndDateBefore(SubscriptionStatus.ACTIVE, today);
    for (Subscription sub : expired) {
        sub.setStartDate(today);
        sub.setEndDate(today.plusDays(sub.getPlan().getDuration()));
        sub.setNextCycleWithProrated(sub.getPlan().getPrice());
    }
    subscriptionDao.saveAll(expired);
}
```

This also requires adding the derived query to `SubscriptionDao`:
```java
List<Subscription> findByStatusAndAutoRenewTrueAndEndDateBefore(SubscriptionStatus status, LocalDate date);
```

Because the method returns `null` and the interface declares `Subscription` as return type, the signature is also semantically wrong — renewal affects multiple subscriptions, so the return type should be `void` (or `List<Subscription>`).

#### Kafka integration is disabled
`SubscriptionEventPublisher` is commented out (`//@Component`) and its usage in the service is also commented out. Kafka is a hard requirement of the kata. The publisher should be enabled and called after a successful `createSubscription` (and optionally after `changeSubscription`).

The event record `SubscriptionCreateEvent` is defined but never used — the publisher builds a raw JSON string manually instead. Use the record:

```java
// In SubscriptionEventPublisher
public void publishSubscriptionCreated(Subscription sub) {
    var event = new SubscriptionCreateEvent(sub.getId(), sub.getCustomer().getId(), sub.getPlan());
    kafkaTemplate.send("subscription-events", String.valueOf(sub.getId()), event);
}
// Change KafkaTemplate<String, String> to KafkaTemplate<String, SubscriptionCreateEvent>
```

#### `@Autowired` on fields
Constructor injection is the Spring-recommended approach. Field injection hides dependencies and makes unit testing harder.

```java
// Instead of:
@Autowired
private SubscriptionDao subscriptionDao;

// Prefer:
private final SubscriptionDao subscriptionDao;

public SubscriptionService(SubscriptionDao subscriptionDao, ...) {
    this.subscriptionDao = subscriptionDao;
}
```

#### No error handling / HTTP status codes
The controller returns `200 OK` for everything, including creation (should be `201 Created`). Exceptions like `EntityNotFoundException` or `IllegalStateException` are not mapped to HTTP responses. A `@ControllerAdvice` or `@ExceptionHandler` is missing.

```java
@PostMapping("/create")
@ResponseStatus(HttpStatus.CREATED)
public SubscriptionResponse create(@RequestBody CreateSubscriptionRequest request) { ... }
```

#### Proration logic has a rounding issue
```java
prorated = oldPrice.multiply(BigDecimal.valueOf(remainingDays))
        .divide(BigDecimal.valueOf(totalDays), 2); // missing RoundingMode
```
`BigDecimal.divide` without a `RoundingMode` will throw `ArithmeticException` on non-terminating decimals. Fix:

```java
.divide(BigDecimal.valueOf(totalDays), 2, RoundingMode.HALF_UP);
```

#### `@EnableScheduling` is missing
`@Scheduled` on `renewExpiredSubscriptions` will silently do nothing without `@EnableScheduling` on the application or a configuration class.

#### No input validation
`CreateSubscriptionRequest` has no `@NotNull` / `@Valid` constraints. A request with a null `customerId` will produce an unhelpful 500 instead of a 400.

#### Test class is empty
`KataApplicationTests` contains no assertions. At minimum, a context load test is useful, but the kata explicitly requires demonstrating that the solution meets requirements — there are no tests at all.

---

## 2. Summary Table

| Area | Issue | Severity |
|---|---|---|
| Correctness | `renewExpiredSubscriptions` returns `null`, not implemented | Critical |
| Correctness | Kafka disabled (commented out) | Critical |
| Correctness | `BigDecimal.divide` without `RoundingMode` | High |
| Correctness | `@EnableScheduling` missing | High |
| Design | Controller returns entity instead of DTO | High |
| Design | `SubscriptionStatus2` duplicate / conflicting entity | High |
| Design | `SubscriptionCreateEvent` record unused | Medium |
| Design | Field injection instead of constructor injection | Medium |
| Design | No `@ExceptionHandler` / HTTP status mapping | Medium |
| Readability | Lombok declared but getters/setters written manually | Medium |
| Readability | Commented-out code committed | Low |
| Readability | Unused imports in DTOs | Low |
| Testing | No tests | High |
