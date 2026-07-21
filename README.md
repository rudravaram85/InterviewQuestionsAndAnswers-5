   If you're preparing for a **Java Microservices code review interview for 5+ years of experience**, the expectations are significantly higher than basic syntax or Spring Boot usage. Interviewers typically expect you to identify architectural issues, performance bottlenecks, security risks, and maintainability concerns.

Here are the areas you should be comfortable reviewing:

| Area          | What to Look For                                                                  |
| ------------- | --------------------------------------------------------------------------------- |
| Java          | SOLID principles, Collections, Streams, Optional, Exception handling, Concurrency |
| Spring Boot   | Proper layering, Dependency Injection, Configuration, Bean lifecycle              |
| REST APIs     | Correct HTTP methods, status codes, validation, idempotency                       |
| Database      | Efficient JPA usage, N+1 query issues, transaction boundaries, indexing           |
| Security      | Spring Security, JWT/OAuth2, input validation, secret management                  |
| Microservices | Service communication, retries, circuit breakers, timeouts, API versioning        |
| Messaging     | Kafka/RabbitMQ, idempotent consumers, dead-letter queues                          |
| Performance   | Caching, pagination, connection pooling, async processing                         |
| Observability | Structured logging, metrics, tracing, health endpoints                            |
| Testing       | Unit, integration, mocking, Testcontainers                                        |

### Common code review findings

**Avoid field injection**

```java
@Autowired
private UserService userService;
```

Prefer constructor injection:

```java
private final UserService userService;

public UserController(UserService userService) {
    this.userService = userService;
}
```

---

**Avoid broad exception handling**

```java
try {
    ...
} catch (Exception e) {
}
```

Instead:

* Catch specific exceptions.
* Log appropriately.
* Return meaningful error responses.

---

**Validate request payloads**

```java
@PostMapping("/users")
public User create(@RequestBody UserRequest request)
```

Better:

```java
@PostMapping("/users")
public User create(@Valid @RequestBody UserRequest request)
```

with validation annotations such as:

```java
@NotBlank
private String name;

@Email
private String email;
```

---

**Do not expose entities directly**

```java
@GetMapping
public List<User> getUsers()
```

Prefer DTOs:

```java
public List<UserResponse> getUsers()
```

---

**Use global exception handling**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<?> handle(UserNotFoundException ex) {
        ...
    }
}
```

---

**Avoid hardcoded configuration**

```java
String url = "http://localhost:8081";
```

Use configuration:

```properties
service.user.url=http://user-service
```

Inject with:

```java
@Value("${service.user.url}")
private String userServiceUrl;
```

---

**External service calls should have resilience**
Instead of a plain REST call:

```java
restTemplate.getForObject(...)
```

Use:

* Timeouts
* Retries
* Circuit breakers (e.g., Resilience4j)
* Fallbacks where appropriate

---

### Interview expectations for 5+ years

You should be able to explain:

* Why one design is preferable over another.
* Transaction boundaries in distributed systems.
* Stateless service design.
* Optimistic vs. pessimistic locking.
* API versioning strategies.
* Synchronous vs. asynchronous communication.
* Event-driven architecture basics.
* Containerization and Kubernetes deployment considerations.
* Monitoring with metrics, logs, and distributed tracing.
* CI/CD practices and deployment strategies (blue-green, canary).

### Typical code review interview flow

An interviewer may provide a Spring Boot application with intentional issues and ask you to identify problems such as:

* Security vulnerabilities
* Memory leaks
* N+1 database queries
* Missing validation
* Incorrect transaction usage
* Poor exception handling
* Tight coupling
* Performance bottlenecks
* Thread-safety concerns
* Logging sensitive information

They'll usually expect you not only to point out the issue but also to explain its impact and propose a practical improvement.


In **Spring Boot**, a **second-level cache (L2 cache)** is a cache shared across all sessions of the persistence context. Unlike the first-level cache, which exists only within a single Hibernate session, the second-level cache stores entities across sessions, reducing database queries.

### First-Level vs Second-Level Cache

| Feature  | First-Level Cache        | Second-Level Cache                                         |
| -------- | ------------------------ | ---------------------------------------------------------- |
| Scope    | Single Hibernate Session | Shared across Sessions                                     |
| Enabled  | By default               | Must be configured                                         |
| Lifetime | Until session closes     | Until cache expires or is evicted                          |
| Storage  | Memory (Session)         | External cache (Ehcache, Caffeine, Hazelcast, Redis, etc.) |

---

## How Second-Level Cache Works

1. Application requests an entity.
2. Hibernate checks the first-level cache.
3. If not found, Hibernate checks the second-level cache.
4. If found in L2 cache:

   * Returns the entity.
   * No database query is executed.
5. If not found:

   * Fetches from the database.
   * Stores it in the second-level cache.
   * Returns the entity.

```
Application
      |
Hibernate Session (L1 Cache)
      |
      | Not Found
      ↓
Second Level Cache (L2)
      |
      | Not Found
      ↓
Database
```

---

## Popular Cache Providers

Hibernate doesn't provide the cache itself. Common providers include:

* Ehcache
* Caffeine
* Hazelcast
* Redis (via JCache or custom integrations)
* Infinispan

---

## Example using Ehcache

### Step 1: Add Dependencies (Maven)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>

<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

---

### Step 2: Configure `application.properties`

```properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.use_query_cache=true

spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory

spring.cache.jcache.config=classpath:ehcache.xml
```

---

### Step 3: Configure Entity

```java
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Employee {

    @Id
    private Long id;

    private String name;
}
```

---

### Step 4: Configure `ehcache.xml`

```xml
<config xmlns="http://www.ehcache.org/v3">

    <cache alias="com.example.entity.Employee">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>

        <resources>
            <heap unit="entries">1000</heap>
        </resources>
    </cache>

</config>
```

---

## Example

### First Request

```java
employeeRepository.findById(1L);
```

Hibernate executes:

```sql
SELECT * FROM employee WHERE id = 1;
```

The result is stored in the second-level cache.

---

### Second Request (Different Session)

```java
employeeRepository.findById(1L);
```

No SQL is executed.

The entity is returned from the second-level cache.

---

## Query Cache

The second-level cache stores **entities**, not query results.

If you execute:

```java
employeeRepository.findByDepartment("IT");
```

Hibernate still executes the SQL unless the query cache is enabled.

Enable:

```properties
hibernate.cache.use_query_cache=true
```

Use:

```java
Query query = session.createQuery(
    "from Employee where department=:dept");

query.setHint("org.hibernate.cacheable", true);
```

Now Hibernate caches the query result IDs and retrieves the entities from the second-level cache.

---

## Cache Concurrency Strategies

| Strategy               | Description                  | Use Case                                      |
| ---------------------- | ---------------------------- | --------------------------------------------- |
| `READ_ONLY`            | Never updated                | Reference/master data (countries, currencies) |
| `READ_WRITE`           | Safe for updates             | Most business entities                        |
| `NONSTRICT_READ_WRITE` | Allows occasional stale data | Data updated infrequently                     |
| `TRANSACTIONAL`        | Transaction-aware cache      | Distributed/JTA environments                  |

Example:

```java
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
```

---

## When to Use Second-Level Cache

Good candidates:

* Product catalog
* Countries
* States
* Roles
* Permissions
* Configuration tables
* Frequently read, rarely updated data

Avoid caching:

* Frequently updated tables
* Large binary data (BLOBs)
* Highly transactional data

---

## Spring Cache vs Hibernate Second-Level Cache

| Spring Cache               | Hibernate L2 Cache                           |
| -------------------------- | -------------------------------------------- |
| Caches method results      | Caches JPA entities                          |
| Uses `@Cacheable` (Spring) | Uses `@Cacheable` (JPA) + Hibernate `@Cache` |
| Works with any method      | Works only with Hibernate/JPA                |
| Managed by Spring          | Managed by Hibernate                         |

---

## Advantages

* Reduces database load.
* Improves application performance.
* Shared across multiple sessions.
* Transparent to repository/service code once configured.

## Limitations

* Uses additional memory.
* Cache invalidation can be complex for frequently updated data.
* Not beneficial for data that changes often.

### Summary

* **First-level cache**: Session-scoped, enabled by default.
* **Second-level cache**: Shared across sessions, requires configuration.
* **Query cache**: Caches query result identifiers and works best when paired with the second-level cache.
* Choose an appropriate cache provider (Ehcache, Caffeine, Hazelcast, Redis, etc.) and a concurrency strategy based on your application's read/write patterns.
Idempotency in a REST API means that making the same request multiple times has the same effect as making it once. This is especially important for operations like payments, order creation, and booking systems, where retries due to network failures could otherwise create duplicate records.

## 1. Understand HTTP idempotency

According to HTTP semantics:

* **GET** ✅ Idempotent
* **PUT** ✅ Idempotent
* **DELETE** ✅ Idempotent
* **POST** ❌ Not idempotent by default
* **PATCH** Depends on implementation

The biggest challenge is making **POST** requests idempotent.

---

## 2. Use an Idempotency Key (Recommended)

The client generates a unique key for each logical operation.

Example request:

```http
POST /orders
Idempotency-Key: 6db7b91e-d48f-4f62-a93f-2c5fbe2b8bfa
```

If the client retries with the same key:

* No duplicate order is created.
* Return the previously generated response.

### Spring Boot Entity

```java
@Entity
public class IdempotencyRecord {

    @Id
    private String idempotencyKey;

    @Lob
    private String responseBody;

    private int statusCode;

    private LocalDateTime createdAt;
}
```

Repository

```java
public interface IdempotencyRepository
        extends JpaRepository<IdempotencyRecord, String> {
}
```

---

## 3. Check the key before processing

```java
@Service
public class OrderService {

    @Autowired
    private IdempotencyRepository repository;

    public ResponseEntity<?> createOrder(String key, OrderRequest request) {

        Optional<IdempotencyRecord> existing =
                repository.findById(key);

        if (existing.isPresent()) {

            IdempotencyRecord record = existing.get();

            return ResponseEntity
                    .status(record.getStatusCode())
                    .body(record.getResponseBody());
        }

        Order order = createNewOrder(request);

        String response = convertToJson(order);

        repository.save(new IdempotencyRecord(
                key,
                response,
                201,
                LocalDateTime.now()));

        return ResponseEntity.status(201).body(order);
    }
}
```

---

## 4. Add a unique database constraint

Even if two requests arrive simultaneously, the database should prevent duplicates.

Example:

```java
@Entity
@Table(
    uniqueConstraints =
    @UniqueConstraint(columnNames = "transactionId")
)
public class Payment {

    @Id
    @GeneratedValue
    private Long id;

    private String transactionId;
}
```

If duplicate inserts occur:

```java
try {
    paymentRepository.save(payment);
} catch (DataIntegrityViolationException ex) {
    // Already processed
}
```

---

## 5. Use Transactions

Wrap the operation in a transaction to ensure atomicity.

```java
@Transactional
public Order createOrder(OrderRequest request) {

    // save order

    // update inventory

    // publish event

    return order;
}
```

---

## 6. Prevent race conditions

Two identical requests may arrive at nearly the same time.

Options include:

* Database unique constraints (most common)
* Pessimistic locking

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Order> findByTransactionId(String id);
```

* Optimistic locking (`@Version`)
* Distributed locks (e.g., Redis) for multi-instance deployments

---

## 7. Store response for retries

If a request is successfully processed:

| Idempotency Key | Status | Response   |
| --------------- | ------ | ---------- |
| abc123          | 201    | Order #101 |

On retry:

```
Client --> POST (abc123)

Server

lookup key

Found

return saved response
```

No business logic runs again.

---

## 8. Expire old keys

Idempotency records do not need to be kept forever.

Typical TTLs:

* Payments: 24–48 hours
* Orders: 24 hours
* File uploads: 1 hour

A scheduled job can remove expired records.

```java
@Scheduled(cron = "0 0 * * * *")
public void cleanup() {
    repository.deleteExpired();
}
```

---

## 9. Use Redis for high throughput

Instead of storing keys in a relational database:

```java
Boolean exists = redisTemplate.hasKey(key);

if (Boolean.TRUE.equals(exists)) {
    return previousResponse;
}

redisTemplate.opsForValue().set(
        key,
        response,
        Duration.ofHours(24));
```

This reduces database load and is suitable for high-volume APIs.

---

## 10. Example request flow

```
Client
   |
POST /orders
Idempotency-Key: xyz123
   |
   v
Spring Boot
   |
Check Redis/DB
   |
+------------------+
| Exists?          |
+------------------+
      |
  Yes | No
      |-------------------------+
      |                         |
Return saved response     Process request
                           Save order
                           Save response
                           Save key
                           Return 201
```

---

## Best practices

* Require an `Idempotency-Key` header for non-idempotent operations like `POST`.
* Persist the response associated with each key and return it for retries.
* Use database unique constraints as a safety net against duplicate writes.
* Wrap business logic in transactions.
* Handle concurrent requests with locking or unique constraints.
* Set a TTL for idempotency records to avoid unbounded storage growth.
* Consider Redis for fast idempotency key lookups in distributed systems.
* Ensure the same idempotency key is only reused for the same request payload; if the payload differs, reject the request (for example, with `409 Conflict` or `422 Unprocessable Entity`) to prevent accidental misuse.

This combination of **idempotency keys + transactional processing + unique constraints** is the most common and robust pattern for implementing idempotent `POST` endpoints in Spring Boot applications.


In Spring transactions, a rollback can fail, although it's relatively uncommon. When it does, it usually means the application could not successfully undo the database transaction. The consequences depend on when and why the rollback failed.

Here are the common scenarios:

### 1. Connection lost before rollback

If the database connection is lost before Spring calls `rollback()`:

```java
@Transactional
public void transfer() {
    accountRepository.withdraw();
    accountRepository.deposit();
    throw new RuntimeException();
}
```

Expected flow:

```
BEGIN
withdraw
deposit
Exception
ROLLBACK
```

If the connection drops before `ROLLBACK`:

```
BEGIN
withdraw
deposit
Connection lost
ROLLBACK fails
```

The transaction's final state depends on the database:

* Many databases automatically roll back uncommitted transactions when the connection is closed.
* If the database had already committed for some reason, rollback has no effect.
* If the database cannot determine the state immediately, the outcome may be uncertain until recovery.

---

### 2. Rollback exception masks the original exception

Suppose your business code throws:

```java
throw new IllegalArgumentException("Invalid input");
```

During rollback:

```java
connection.rollback(); // throws SQLException
```

Spring typically:

* Preserves the original exception.
* Wraps or logs the rollback failure.
* Throws a transaction-related exception such as:

```text
TransactionSystemException
```

The rollback exception is usually available as the cause.

---

### 3. Database crash during rollback

If the database crashes while rolling back:

* The transaction remains incomplete.
* During database recovery, the database's transaction log (WAL, redo/undo logs) determines whether to commit or roll back the transaction.
* ACID-compliant databases are designed to recover to a consistent state.

---

### 4. Rollback after commit

Once a transaction has committed:

```java
connection.commit();
connection.rollback(); // impossible
```

Rollback does nothing because the changes are already permanent.

If an exception occurs after commit:

```java
@Transactional
public void process() {
    repository.save(entity);   // committed later
}

// transaction commits

sendEmail(); // throws exception
```

The database changes remain committed. The email failure cannot be undone with a database rollback.

---

### 5. Nested transactions

Consider:

```java
@Transactional
public void outer() {
    service.inner();
}
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void inner() {
    ...
}
```

If the inner transaction commits successfully and the outer transaction later rolls back:

* The inner transaction stays committed.
* The outer transaction is rolled back independently.

This is because `REQUIRES_NEW` starts a separate physical transaction.

---

### 6. Rollback-only marker

Sometimes Spring marks a transaction as rollback-only:

```java
@Transactional
public void method() {
    try {
        repository.save(...);
        throw new RuntimeException();
    } catch (Exception e) {
        // swallowed
    }
}
```

Even though the exception is caught, the transaction may already be marked rollback-only.

When Spring attempts to commit:

```
UnexpectedRollbackException
```

This is not a rollback failure. It indicates that Spring intentionally rolled back the transaction because it had been marked as rollback-only.

---

## How Spring handles rollback internally

The simplified flow looks like this:

```
try {
    beginTransaction();

    businessMethod();

    commit();

} catch (Exception ex) {

    try {
        rollback();
    } catch (Exception rollbackEx) {
        // rollback itself failed
        throw new TransactionSystemException(
            "Rollback failed", rollbackEx);
    }

    throw ex;
}
```

---

## Best practices

* Avoid swallowing exceptions inside `@Transactional` methods unless you understand how Spring's rollback rules work.
* Let transaction management handle commit and rollback automatically.
* Log both the original exception and any rollback exception.
* Design external side effects (emails, messaging, HTTP calls) to be idempotent or use patterns like the Outbox pattern, since they cannot be rolled back with the database transaction.
* Monitor database connection health, because rollback failures are often caused by connection or infrastructure issues rather than application logic.

In practice, **rollback failures are rare**. More commonly, developers encounter:

* `UnexpectedRollbackException` due to a transaction being marked rollback-only.
* Exceptions after a transaction has already committed.
* Rollbacks triggered by checked vs. unchecked exception behavior (Spring rolls back by default only for unchecked exceptions unless configured otherwise).
A **memory leak** and a **`OutOfMemoryError`** (sometimes mistakenly called a "memory extension error") are related but different concepts in Java.

| Memory Leak                                                             | `OutOfMemoryError`                                                                      |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| A programming problem where objects are unintentionally kept in memory. | A runtime error thrown when the JVM cannot allocate more memory.                        |
| Memory usage gradually increases over time.                             | Happens when the JVM runs out of available memory.                                      |
| Caused by objects that are no longer needed but are still referenced.   | Can be caused by memory leaks, insufficient heap size, or legitimate high memory usage. |
| May not immediately crash the application.                              | Often causes the application or operation to fail.                                      |

### Memory Leak Example

```java
import java.util.ArrayList;
import java.util.List;

public class MemoryLeakExample {
    static List<byte[]> cache = new ArrayList<>();

    public static void main(String[] args) {
        while (true) {
            cache.add(new byte[1024 * 1024]); // Add 1 MB repeatedly
        }
    }
}
```

**Why it's a leak:**

* The `cache` list keeps references to every allocated array.
* The garbage collector cannot reclaim those arrays because they are still reachable.
* Memory usage keeps growing.

---

### `OutOfMemoryError` Example

```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

This error occurs when the JVM cannot allocate additional memory.

Common reasons:

* A memory leak.
* The heap is too small (e.g., running with `-Xmx128m` for a memory-intensive application).
* The application genuinely needs more memory than is available.
* Creating very large objects or too many objects.

---

### Relationship

A memory leak can eventually lead to an `OutOfMemoryError`, but not every `OutOfMemoryError` is caused by a memory leak.

```
Memory Leak
      ↓
Heap usage keeps increasing
      ↓
Garbage Collector cannot free enough memory
      ↓
OutOfMemoryError
```

However, this can also happen without a leak:

```java
byte[] hugeArray = new byte[5 * 1024 * 1024 * 1024L]; // Too large
```

Even though there is no leak, the allocation itself may exceed the available heap and trigger an `OutOfMemoryError`.

---

### How to Detect Them

**Memory Leak**

* Heap usage continuously grows over time.
* Analyze heap dumps using tools such as Eclipse MAT, VisualVM, or Java Flight Recorder.
* Look for objects that remain reachable unexpectedly.

**`OutOfMemoryError`**

* Inspect the exception message, for example:

  * `Java heap space`
  * `GC overhead limit exceeded`
  * `Metaspace`
  * `Direct buffer memory`
* Review GC logs and heap dumps.
* Determine whether the cause is a leak, configuration issue, or workload.

### Summary

* **Memory leak:** Objects that should be garbage-collected remain referenced, causing memory usage to grow unnecessarily.
* **`OutOfMemoryError`:** The JVM cannot allocate more memory. It may be caused by a memory leak, insufficient JVM memory settings, or an application that legitimately requires more memory than is available.

If you're having an issue looping through a `HashMap` in Java, here are the most common patterns and pitfalls.

### 1. Loop through key-value pairs (recommended)

```java
Map<String, Integer> map = new HashMap<>();
map.put("Alice", 25);
map.put("Bob", 30);

for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}
```

### 2. Loop through keys

```java
for (String key : map.keySet()) {
    System.out.println(key);
}
```

### 3. Loop through values

```java
for (Integer value : map.values()) {
    System.out.println(value);
}
```

### 4. Using `forEach` (Java 8+)

```java
map.forEach((key, value) -> {
    System.out.println(key + " -> " + value);
});
```

## Common issues

### ❌ ConcurrentModificationException

If you remove entries while looping like this:

```java
for (String key : map.keySet()) {
    if (key.equals("Alice")) {
        map.remove(key); // Throws ConcurrentModificationException
    }
}
```

Use an iterator instead:

```java
Iterator<Map.Entry<String, Integer>> iterator = map.entrySet().iterator();

while (iterator.hasNext()) {
    Map.Entry<String, Integer> entry = iterator.next();

    if (entry.getKey().equals("Alice")) {
        iterator.remove();
    }
}
```

---

### ❌ Calling `get()` repeatedly

Instead of:

```java
for (String key : map.keySet()) {
    System.out.println(key + " -> " + map.get(key));
}
```

Prefer:

```java
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}
```

This avoids an extra lookup.

---

### ❌ Expecting insertion order

`HashMap` does **not** guarantee iteration order.

If you need:

* Insertion order → `LinkedHashMap`
* Sorted keys → `TreeMap`

---

If you're encountering a specific problem, please share:

* Your loop code
* The expected output
* The actual output or exception (if any)

That will make it easier to pinpoint the issue.


If you're targeting companies like TCS, Infosys, Accenture, Capgemini, Cognizant, Wipro, HCL, Deloitte, or product-based companies, practicing reviews of real Spring Boot microservices is one of the best ways to prepare.

In a Spring Boot application, initial (seed) data can be loaded into the database in several common ways. The best approach depends on whether you want data only during development, for testing, or in production.

Option 1: data.sql (Simple and Recommended)

Create a file:

src/main/resources/data.sql

Example:

INSERT INTO users (id, name, email) VALUES (1, 'John Doe', 'john@example.com');
INSERT INTO users (id, name, email) VALUES (2, 'Jane Smith', 'jane@example.com');

If using JPA, also create schema.sql or let Hibernate generate the tables.


------------------------------------------------------------------------------------------------------------------------------------------------------------------
Achieving idempotency across **multiple payment gateways** in Java is less about Java itself and more about designing a consistent payment orchestration layer. Since each gateway (Stripe, Razorpay, Adyen, PayPal, etc.) handles idempotency differently, your application should enforce idempotency independently.

## Architecture

```
Client
   |
   | Payment Request (idempotencyKey)
   |
Payment API
   |
   +-----------------------------+
   | Idempotency Service         |
   | (DB/Redis)                  |
   +-----------------------------+
               |
       Already processed?
        /              \
      Yes              No
      |                 |
Return saved      Call Payment Gateway
response               |
                        |
                  Save response
                        |
                  Return response
```

## 1. Generate an Idempotency Key

The client should send an idempotency key.

```
POST /payments

Headers:
Idempotency-Key: 8b9f5c6f-d3ae-4c23-a4b2-123456789abc
```

If the client doesn't provide one, your backend can generate it for internal workflows.

---

## 2. Create an Idempotency Table

Example:

```sql
CREATE TABLE payment_idempotency (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key VARCHAR(100) UNIQUE,
    request_hash VARCHAR(64),
    status VARCHAR(20),
    response TEXT,
    created_at TIMESTAMP
);
```

Fields:

* idempotency_key
* request_hash
* status
* response
* timestamp

---

## 3. Hash the Request

This prevents someone from sending the same key with different payment details.

```java
String payload = objectMapper.writeValueAsString(paymentRequest);

String requestHash = DigestUtils.sha256Hex(payload);
```

If

```
idempotencyKey == same
```

but

```
requestHash != existingHash
```

return

```
409 Conflict
```

---

## 4. Use Database Locking

Example using Spring Data JPA:

```java
@Transactional
public PaymentResponse processPayment(
        String idempotencyKey,
        PaymentRequest request) {

    Optional<PaymentRecord> existing =
            repository.findByIdempotencyKey(idempotencyKey);

    if (existing.isPresent()) {
        return existing.get().getResponse();
    }

    PaymentRecord record = new PaymentRecord();
    record.setIdempotencyKey(idempotencyKey);
    record.setStatus("PROCESSING");

    repository.save(record);

    PaymentResponse response = gateway.charge(request);

    record.setStatus("SUCCESS");
    record.setResponse(response);

    repository.save(record);

    return response;
}
```

Better yet, rely on a **UNIQUE** constraint on `idempotency_key` and handle duplicate insert exceptions to avoid race conditions.

---

## 5. Gateway Abstraction

```java
public interface PaymentGateway {

    PaymentResponse charge(PaymentRequest request);
}
```

Implementations:

```java
StripeGateway

RazorpayGateway

PayPalGateway

AdyenGateway
```

Your payment service should not care which gateway is used.

---

## 6. Pass the Gateway's Idempotency Key

Many gateways support idempotency.

Example:

```java
HttpHeaders headers = new HttpHeaders();

headers.add("Idempotency-Key", idempotencyKey);
```

Even if the gateway supports idempotency, **still maintain your own idempotency layer**, because:

* Not all gateways support it.
* Different gateways retain keys for different time periods.
* You want consistent behavior regardless of provider.

---

## 7. Handle Failures

Consider these scenarios:

| Scenario                                  | Action                                                                             |
| ----------------------------------------- | ---------------------------------------------------------------------------------- |
| Request timed out before reaching gateway | Retry with same key                                                                |
| Gateway charged but response lost         | Retry with same key; if supported, gateway idempotency returns the original result |
| Server crashed after charging             | Recover using stored state and reconciliation/webhooks                             |
| Duplicate client request                  | Return cached response                                                             |
| Different payload with same key           | Return `409 Conflict`                                                              |

---

## 8. Handle Concurrent Requests

Suppose two requests arrive simultaneously with the same key.

Bad:

```
Thread A -> No record
Thread B -> No record

Both charge card
```

Good:

```
Thread A -> INSERT PROCESSING

Thread B -> Unique constraint violation

Thread B -> Fetch existing row

Return existing result
```

This can be implemented using:

* Database unique constraints.
* `SELECT ... FOR UPDATE`.
* Optimistic or pessimistic locking.
* Distributed locks (e.g., Redis) if you have multiple application instances, though a database unique constraint is often sufficient.

---

## 9. Store the Final Response

Example:

```json
{
  "paymentId": "pay_12345",
  "status": "SUCCESS",
  "amount": 1000
}
```

Subsequent requests with the same key simply return this stored response.

---

## 10. Example Flow

```
Request #1
------------
Key = abc123

DB -> Not found

↓

Gateway Charge

↓

SUCCESS

↓

Save response

↓

Return SUCCESS
```

```
Request #2 (same key)

↓

DB -> Found

↓

Return cached response

(No gateway call)
```

---

## 11. Multi-Gateway Considerations

If your system can retry on another gateway after a failure, the idempotency record should capture the orchestration state. For example:

```
PaymentRequest
    |
    | Key = abc123
    |
    +--> Stripe (failed before authorization)
    |
    +--> Razorpay (succeeded)
```

The idempotency record should store:

* `idempotencyKey`
* `merchantOrderId`
* `selectedGateway`
* `gatewayTransactionId`
* Overall payment status
* Serialized response

This ensures that retries with the same key return the successful Razorpay result rather than attempting another charge.

## Best Practices

* Generate or require a unique idempotency key per payment attempt.
* Enforce a unique constraint on the idempotency key in the database.
* Store a hash of the request to detect conflicting reuse of a key.
* Save the final response and return it for duplicate requests.
* Pass the same idempotency key to gateways that support it, but do not depend solely on gateway-level idempotency.
* Design payment state transitions (`PROCESSING`, `SUCCESS`, `FAILED`, `UNKNOWN`) carefully and use webhooks or reconciliation jobs to resolve uncertain outcomes.
* Make the orchestration layer—not individual gateway implementations—the source of truth for idempotency. This keeps behavior consistent even when gateways have different APIs or capabilities.
---------------------------------------------------------------------------------------------------------------------------------------------------------------------


`Collections.synchronizedMap()` is a method in Java that returns a **thread-safe (synchronized) wrapper** around an existing `Map`. It ensures that only one thread can access the map at a time.

### Syntax

```java
Map<K, V> syncMap = Collections.synchronizedMap(new HashMap<>());
```

### Example

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Map<Integer, String> map = Collections.synchronizedMap(new HashMap<>());

        map.put(1, "Apple");
        map.put(2, "Banana");

        System.out.println(map.get(1)); // Apple
    }
}
```

### Iterating over a synchronized map

Even though the map is synchronized, **iteration must be manually synchronized** to avoid concurrent modification issues.

```java
Map<Integer, String> map = Collections.synchronizedMap(new HashMap<>());

synchronized (map) {
    for (Map.Entry<Integer, String> entry : map.entrySet()) {
        System.out.println(entry.getKey() + " : " + entry.getValue());
    }
}
```

### When to use

* When multiple threads need to share a `Map`.
* When you already have a `HashMap` (or another `Map`) and want to make it thread-safe without changing its type.

### Limitations

* Uses a **single lock** for all operations, so only one thread can access the map at a time. This can reduce performance under high concurrency.
* Iteration still requires external synchronization.

### `Collections.synchronizedMap()` vs `ConcurrentHashMap`

| Feature       | `Collections.synchronizedMap()` | `ConcurrentHashMap`                                                |
| ------------- | ------------------------------- | ------------------------------------------------------------------ |
| Thread safety | Yes                             | Yes                                                                |
| Locking       | Single lock for entire map      | Fine-grained locking / high concurrency                            |
| Iteration     | Must synchronize manually       | Safe without external synchronization (weakly consistent iterator) |
| Performance   | Lower under heavy concurrency   | Better under heavy concurrency                                     |

### Recommendation

* Use **`Collections.synchronizedMap()`** for simple thread-safe access when contention is low.
* Use **`ConcurrentHashMap`** in highly concurrent applications, as it generally provides better scalability and performance.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------


Concurrency issues in Spring Boot are usually not caused by Spring Boot itself—they arise when multiple threads access shared resources (memory, caches, files, databases, etc.) at the same time. The right solution depends on where the shared state exists.

Here are the most common approaches.

### 1. Make beans stateless (Best Practice)

Spring `@Service` beans are singletons by default. If you store mutable state in them, multiple requests can modify it simultaneously.

**Problem:**

```java
@Service
public class CounterService {

    private int counter = 0;

    public int increment() {
        return ++counter;
    }
}
```

Multiple users calling `increment()` can cause race conditions.

**Solution:**
Keep services stateless.

```java
@Service
public class OrderService {

    public Order createOrder(OrderRequest request) {
        // No shared mutable variables
    }
}
```

---

## 2. Use synchronized

For small critical sections:

```java
public synchronized void updateInventory() {
    // only one thread executes at a time
}
```

or

```java
public void updateInventory() {
    synchronized(this) {
        // critical section
    }
}
```

**Pros**

* Easy
* Built into Java

**Cons**

* Can reduce performance
* Doesn't work across multiple application instances

---

## 3. Use Atomic classes

Instead of

```java
private int count;
```

Use

```java
private AtomicInteger count = new AtomicInteger();

public int increment() {
    return count.incrementAndGet();
}
```

Other useful classes:

* `AtomicInteger`
* `AtomicLong`
* `AtomicReference`
* `LongAdder` (better under heavy contention)

---

## 4. Use Locks

For more control than `synchronized`.

```java
private Lock lock = new ReentrantLock();

public void update() {
    lock.lock();
    try {
        // critical code
    } finally {
        lock.unlock();
    }
}
```

Useful when you need:

* timeout
* fairness
* interruptible locking
* multiple conditions

---

## 5. Database Locking

If multiple users update the same database row, use locking.

### Optimistic Locking

Best when conflicts are rare.

```java
@Entity
public class Product {

    @Id
    private Long id;

    @Version
    private Long version;
}
```

If two users update simultaneously:

* First update succeeds
* Second throws `OptimisticLockException`

This is the recommended approach for many business applications.

---

### Pessimistic Locking

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Product findByIdForUpdate(Long id);
```

The row is locked until the transaction completes.

Useful for:

* banking
* inventory
* ticket booking

---

## 6. Transaction Management

Use transactions to keep related operations atomic.

```java
@Transactional
public void transferMoney(...) {
    withdraw();
    deposit();
}
```

Choose appropriate isolation levels:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

Common isolation levels:

* READ_UNCOMMITTED
* READ_COMMITTED
* REPEATABLE_READ
* SERIALIZABLE (most strict, lowest concurrency)

---

## 7. Concurrent Collections

Instead of:

```java
HashMap<String, User>
```

Use:

```java
ConcurrentHashMap<String, User>
```

Other concurrent collections:

* `CopyOnWriteArrayList`
* `ConcurrentLinkedQueue`
* `BlockingQueue`

---

## 8. Thread-safe Caching

If using caches, use thread-safe implementations.

Example:

```java
ConcurrentHashMap<Long, Product> cache = new ConcurrentHashMap<>();
```

Or use Spring Cache with a thread-safe cache provider like Caffeine or Redis.

---

## 9. Distributed Locking

If your application runs on multiple servers, Java locks (`synchronized`, `ReentrantLock`) won't coordinate across instances.

Use a distributed locking mechanism such as:

* Redis-based locks (e.g., Redisson)
* Database locks
* ZooKeeper
* etcd

---

## 10. Asynchronous Processing

If many threads compete for the same resource, offload work using asynchronous processing.

Example:

```java
@Async
public CompletableFuture<Void> processOrder() {
    ...
}
```

Or use a message queue like RabbitMQ or Kafka to serialize processing for specific workloads.

---

## 11. Prevent Lost Updates

Example:

```
Inventory = 10

Thread A reads 10
Thread B reads 10

A writes 9
B writes 9
```

Expected:

```
8
```

Actual:

```
9
```

Solutions:

* Optimistic locking
* Pessimistic locking
* Atomic database updates:

```sql
UPDATE product
SET quantity = quantity - 1
WHERE id = ?
```

instead of:

```java
quantity = quantity - 1;
save(quantity);
```

---

## Example Interview Scenario

**Problem:** Two users place the last item in stock simultaneously.

**Poor implementation:**

```java
Product p = repository.findById(id);

if (p.getStock() > 0) {
    p.setStock(p.getStock() - 1);
    repository.save(p);
}
```

Both users may see stock = 1 and both purchase it.

**Better implementation:**

```java
@Transactional
public void purchase(Long id) {

    Product p = repository.findByIdForUpdate(id);

    if (p.getStock() <= 0)
        throw new OutOfStockException();

    p.setStock(p.getStock() - 1);
}
```

Or use optimistic locking with a `@Version` field and retry the transaction if an `OptimisticLockException` occurs.

---

## Which approach should you choose?

| Scenario                               | Recommended Solution                                   |
| -------------------------------------- | ------------------------------------------------------ |
| Stateless business logic               | Keep Spring beans stateless                            |
| Shared in-memory counter               | `AtomicInteger` or `LongAdder`                         |
| Protect a small critical section       | `synchronized`                                         |
| Advanced thread coordination           | `ReentrantLock`                                        |
| Multiple threads using collections     | `ConcurrentHashMap` and other concurrent collections   |
| Concurrent database updates            | `@Version` (optimistic locking)                        |
| High-conflict database operations      | Pessimistic locking                                    |
| Multiple DB operations as one unit     | `@Transactional`                                       |
| Multiple application instances         | Distributed locking (e.g., Redis)                      |
| High-throughput asynchronous workloads | Queues (Kafka, RabbitMQ) or `@Async` where appropriate |

### Interview tip

If asked, "How do you handle concurrency in Spring Boot?", a strong answer is:

> "I first avoid shared mutable state by keeping Spring beans stateless. For in-memory concurrency, I use thread-safe constructs like `AtomicInteger`, `ConcurrentHashMap`, or `ReentrantLock` as appropriate. For database concurrency, I rely on transactions and optimistic locking using `@Version`; if contention is high or strict consistency is required, I use pessimistic locking. In distributed deployments, I use distributed locks or database-level coordination rather than Java synchronization primitives."

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------


Custom exceptions in Java let you define application-specific error types that make your code easier to understand and handle.

## Why use custom exceptions?

Custom exceptions help you:

* Represent specific error conditions (e.g., `InvalidAgeException`, `InsufficientBalanceException`)
* Improve code readability
* Handle different errors separately
* Provide meaningful error messages

## Types of Custom Exceptions

There are two types:

1. **Checked Exceptions** (extend `Exception`)

   * Checked at compile time.
   * Must be handled using `try-catch` or declared with `throws`.

2. **Unchecked Exceptions** (extend `RuntimeException`)

   * Checked at runtime.
   * Handling is optional.

---

## 1. Creating a Checked Custom Exception

```java
class InvalidAgeException extends Exception {

    public InvalidAgeException(String message) {
        super(message);
    }
}
```

### Using it

```java
public class Voting {

    static void checkAge(int age) throws InvalidAgeException {
        if (age < 18) {
            throw new InvalidAgeException("Age must be at least 18 to vote.");
        }

        System.out.println("Eligible to vote.");
    }

    public static void main(String[] args) {
        try {
            checkAge(16);
        } catch (InvalidAgeException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

**Output:**

```
Age must be at least 18 to vote.
```

---

## 2. Creating an Unchecked Custom Exception

```java
class InsufficientBalanceException extends RuntimeException {

    public InsufficientBalanceException(String message) {
        super(message);
    }
}
```

### Using it

```java
public class BankAccount {

    static void withdraw(double balance, double amount) {
        if (amount > balance) {
            throw new InsufficientBalanceException("Insufficient balance.");
        }

        System.out.println("Withdrawal successful.");
    }

    public static void main(String[] args) {
        withdraw(1000, 1500);
    }
}
```

**Output:**

```
Exception in thread "main"
InsufficientBalanceException: Insufficient balance.
```

---

## Custom Exception with Multiple Constructors

```java
class MyException extends Exception {

    public MyException() {
        super();
    }

    public MyException(String message) {
        super(message);
    }

    public MyException(String message, Throwable cause) {
        super(message, cause);
    }

    public MyException(Throwable cause) {
        super(cause);
    }
}
```

---

## Best Practices

* Choose **checked exceptions** for recoverable conditions that callers are expected to handle.
* Choose **unchecked exceptions** (`RuntimeException`) for programming errors or invalid API usage.
* Use descriptive names ending with `Exception`.
* Include meaningful error messages.
* Preserve the original cause when wrapping another exception:

```java
try {
    // code that throws IOException
} catch (IOException e) {
    throw new MyException("Failed to read file.", e);
}
```

---

## Summary

| Feature                | Checked Exception  | Unchecked Exception               |
| ---------------------- | ------------------ | --------------------------------- |
| Base class             | `Exception`        | `RuntimeException`                |
| Compile-time checking  | Yes                | No                                |
| Must handle or declare | Yes                | No                                |
| Use case               | Recoverable errors | Programming errors, invalid input |

Custom exceptions make Java applications more maintainable by providing clear, domain-specific error types instead of relying only on generic exceptions like `Exception` or `RuntimeException`.

-------------------------------------------------------------------------------------------------------------------------------------------------------------------






