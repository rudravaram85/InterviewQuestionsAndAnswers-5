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




