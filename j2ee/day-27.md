# Day 27 — Testing Enterprise Jakarta EE Applications

**Date:** October 29, 2026
**J2EE/Jakarta EE 30-Day Mastery — Day 27**

Today you move from:

> **“I can build an enterprise application.”**

to:

> **“I can prove that the enterprise application works correctly under realistic conditions.”**

For a solutions architect, testing is not just about writing JUnit tests. You need to understand **what to test, where to test it, how much to mock, how to test infrastructure, and how to test distributed behavior.**

---

# 1. Day 27 Learning Objectives

By the end of today, you should understand:

* Testing pyramid
* Unit testing
* Integration testing
* Component testing
* System testing
* End-to-end testing
* JUnit
* Mockito
* CDI testing
* Servlet testing
* REST API testing
* JPA testing
* JDBC testing
* Transaction testing
* JMS testing
* EJB testing
* Database testing
* Test databases
* Testcontainers
* Mocking
* Stubbing
* Contract testing
* Test fixtures
* Test data builders
* Assertions
* Exception testing
* Security testing
* Performance testing
* Test isolation
* CI/CD testing
* Test strategy

---

# 2. The Enterprise Testing Pyramid

Start with this model:

```text
                    /\
                   /  \
                  / E2E\
                 /------\
                / System \
               /----------\
              / Integration\
             /--------------\
            /   Unit Tests   \
           /------------------\
```

Generally:

```text
Many fast tests
       ↓
Unit tests

Fewer
       ↓
Integration tests

Even fewer
       ↓
End-to-end tests
```

Why?

Because E2E tests are usually:

* slower
* more expensive
* harder to diagnose
* more dependent on infrastructure

---

# 3. Unit Test

A unit test tests a small piece of behavior in isolation.

Example:

```java
class EmployeeServiceTest {

    @Test
    void shouldCalculateAnnualSalary() {

        Employee employee =
                new Employee("Pradeep", 50000);

        double result =
                employee.getMonthlySalary() * 12;

        assertEquals(600000, result);
    }
}
```

The test should be:

```text
Fast
Deterministic
Independent
Repeatable
```

---

# 4. What Is the Unit?

A unit could be:

* method
* class
* small component
* business rule

Example:

```text
EmployeeService
      |
      +-- validateEmployee()
      +-- calculateSalary()
      +-- determineBonus()
```

You might unit-test each business behavior without connecting to:

```text
Database
JMS
HTTP
External APIs
```

---

# 5. Integration Testing

Integration tests verify that multiple components work together.

Example:

```text
EmployeeService
      ↓
Repository
      ↓
JPA
      ↓
Database
```

This tests integration between layers.

---

# 6. Unit vs Integration

| Unit                        | Integration                  |
| --------------------------- | ---------------------------- |
| Small scope                 | Larger scope                 |
| Usually mocked dependencies | Real dependencies often used |
| Very fast                   | Slower                       |
| Isolated                    | Components interact          |
| No DB usually               | DB may be used               |
| Easy debugging              | More realistic               |

---

# 7. System Testing

System testing validates the complete application behavior.

Example:

```text
HTTP
 ↓
REST
 ↓
Service
 ↓
JPA
 ↓
Database
```

You verify the system as a whole.

---

# 8. End-to-End Testing

E2E tests simulate a real user/business flow.

Example:

```text
Login
 ↓
Create Employee
 ↓
Persist Employee
 ↓
Publish Event
 ↓
Consumer Processes Event
 ↓
Audit Created
 ↓
Verify Result
```

This validates the complete business journey.

---

# 9. Test Pyramid for Your Employee Platform

Your project might have:

```text
                10 E2E
              -----------
             50 Integration
           ----------------
          300 Unit Tests
        --------------------
```

The exact numbers aren't universal.

The important principle is:

> Put most tests at the fastest useful level.

---

# 10. JUnit

JUnit is the foundation for Java testing.

Typical annotations:

```java
@Test
@BeforeEach
@AfterEach
@BeforeAll
@AfterAll
@DisplayName
@Disabled
```

Example:

```java
class CalculatorTest {

    @Test
    void shouldAddNumbers() {

        int result = 10 + 20;

        assertEquals(30, result);
    }
}
```

---

# 11. Assertions

Common assertions:

```java
assertEquals(expected, actual);

assertNotEquals(expected, actual);

assertTrue(condition);

assertFalse(condition);

assertNull(value);

assertNotNull(value);
```

Exception:

```java
assertThrows(
    IllegalArgumentException.class,
    () -> service.create(null)
);
```

---

# 12. `@BeforeEach`

Use this for test setup.

```java
@BeforeEach
void setUp() {
    service = new EmployeeService();
}
```

Runs before each test.

---

# 13. `@AfterEach`

Use for cleanup.

```java
@AfterEach
void tearDown() {
    // cleanup
}
```

---

# 14. Parameterized Tests

Instead of duplicating tests:

```java
@ParameterizedTest
@ValueSource(strings = {
    "a@example.com",
    "b@example.com",
    "c@example.com"
})
void shouldAcceptValidEmail(String email) {

    assertTrue(email.contains("@"));
}
```

Useful for validation and business rules.

---

# 15. Mockito

Mockito is commonly used for mocking dependencies.

Suppose:

```text
EmployeeService
      ↓
EmployeeRepository
```

Unit test:

```java
@Mock
EmployeeRepository repository;
```

Then:

```java
when(repository.findById(1L))
    .thenReturn(employee);
```

---

# 16. Why Mock Dependencies?

Suppose your service:

```java
class EmployeeService {

    private EmployeeRepository repository;

}
```

You want to test:

```text
Business logic
```

without testing:

```text
Database
```

So:

```text
EmployeeService
      |
      v
Mock Repository
```

---

# 17. Mockito Example

```java
@ExtendWith(MockitoExtension.class)
class EmployeeServiceTest {

    @Mock
    private EmployeeRepository repository;

    @InjectMocks
    private EmployeeService service;

    @Test
    void shouldFindEmployee() {

        Employee employee =
                new Employee(1L, "Pradeep");

        when(repository.findById(1L))
                .thenReturn(Optional.of(employee));

        Employee result =
                service.findById(1L);

        assertEquals(
                "Pradeep",
                result.getName()
        );
    }
}
```

---

# 18. Verify Interactions

You can verify:

```java
verify(repository)
    .findById(1L);
```

Or:

```java
verify(repository, times(1))
    .findById(1L);
```

But don't overuse interaction verification.

Prefer testing meaningful behavior.

---

# 19. Mock vs Stub

### Stub

Provides predetermined behavior.

```text
Repository
   ↓
return employee
```

### Mock

Can additionally verify interactions.

```text
Was repository called?
How many times?
With what argument?
```

---

# 20. Mock vs Real Dependency

A common architecture mistake is mocking everything.

Suppose:

```text
REST
 ↓
Service
 ↓
JPA
 ↓
PostgreSQL
```

If you mock all of it:

```text
REST Mock
Service Mock
JPA Mock
DB Mock
```

you may end up proving only that your mocks behave as configured.

Therefore:

> Mock at boundaries when isolation provides value; use real infrastructure in integration tests.

---

# 21. Testing a Repository

Repository:

```java
public Optional<Employee> findById(Long id) {
    // database operation
}
```

A unit test shouldn't necessarily connect to PostgreSQL.

Instead, repository integration testing should use:

```text
Repository
 ↓
JPA/JDBC
 ↓
Test Database
```

---

# 22. Test Database Options

Common approaches:

### Option 1 — H2

```text
Application
 ↓
H2
```

Fast, but behavior may differ from production DB.

### Option 2 — Dedicated test database

```text
Application
 ↓
PostgreSQL Test DB
```

More realistic.

### Option 3 — Testcontainers

```text
Application
 ↓
Docker Container
 ↓
Real PostgreSQL
```

This can provide excellent environment fidelity.

---

# 23. H2 vs Production Database

Suppose production uses PostgreSQL.

Your test uses H2.

Potential differences:

```text
SQL syntax
Indexes
Transactions
Isolation
JSON types
Sequences
Extensions
Lock behavior
```

Therefore:

> A test passing against H2 does not prove it will behave identically against PostgreSQL.

---

# 24. Testcontainers

Testcontainers allows tests to start disposable infrastructure containers.

Conceptually:

```text
JUnit
  |
  v
Testcontainers
  |
  v
Docker
  |
  +-- PostgreSQL
  +-- Redis
  +-- Kafka
  +-- Message Broker
```

Now integration tests can use real infrastructure.

---

# 25. Example Testcontainers Concept

Conceptually:

```java
@Testcontainers
class EmployeeRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:...");
}
```

Your application can then connect to the temporary database.

The exact dependency/version should match your project's Java and Testcontainers setup.

---

# 26. Why Testcontainers Is Valuable

Instead of:

```text
Developer machine
   ↓
Manually configured DB
```

you can have:

```text
Test
 ↓
Start PostgreSQL
 ↓
Run migrations
 ↓
Run tests
 ↓
Destroy PostgreSQL
```

This improves reproducibility.

---

# 27. Database Integration Test

Architecture:

```text
JUnit
 ↓
EmployeeRepository
 ↓
JPA
 ↓
PostgreSQL
```

Test:

```text
Create employee
 ↓
Persist
 ↓
Read
 ↓
Verify
 ↓
Delete
```

This tests actual persistence behavior.

---

# 28. Transaction Testing

Transactions must be tested explicitly.

Example:

```text
Create Employee
+
Create Audit
```

Expected:

```text
Both succeed
    ↓
COMMIT
```

Failure:

```text
Employee succeeds
Audit fails
    ↓
ROLLBACK
```

Test:

```text
Database before
       ↓
Transaction
       ↓
Failure
       ↓
Database should remain consistent
```

---

# 29. Testing Rollback

Example scenario:

```text
employee created
audit insertion fails
```

Expected:

```text
Employee NOT persisted
```

This is much more valuable than simply testing:

```text
assertNotNull(employee)
```

because you're testing the transactional contract.

---

# 30. Testing REST APIs

REST testing verifies:

```text
HTTP request
      ↓
REST resource
      ↓
Service
      ↓
Response
```

Test:

```text
POST /employees
```

Request:

```json
{
  "name": "Pradeep",
  "email": "pradeep@example.com"
}
```

Expected:

```text
201 Created
```

---

# 31. REST Test Cases

Test:

### Success

```text
POST /employees
→ 201
```

### Invalid input

```text
POST /employees
→ 400
```

### Missing authentication

```text
GET /employees
→ 401
```

### Insufficient authorization

```text
DELETE /employees/10
→ 403
```

### Missing employee

```text
GET /employees/999
→ 404
```

### Conflict

```text
POST duplicate employee
→ 409
```

---

# 32. REST Testing Matrix

| Scenario               | Expected |
| ---------------------- | -------- |
| Valid request          | 2xx      |
| Invalid JSON           | 400      |
| Validation failure     | 400      |
| Unauthenticated        | 401      |
| Unauthorized           | 403      |
| Resource missing       | 404      |
| Duplicate resource     | 409      |
| Unsupported media type | 415      |
| Rate limit             | 429      |
| Server failure         | 5xx      |

---

# 33. Testing Headers

Don't test only the body.

Test:

```text
Content-Type
Location
Authorization
Correlation-ID
Cache-Control
```

For example, after creation:

```text
HTTP/1.1 201 Created
Location: /employees/101
```

Verify the `Location` header.

---

# 34. Testing Validation

Suppose:

```java
@NotBlank
private String name;

@Email
private String email;
```

Tests should include:

```text
null name
empty name
blank name
invalid email
valid email
```

Don't test only the happy path.

---

# 35. Testing Exception Handling

Suppose:

```java
throw new EmployeeNotFoundException();
```

Verify that the REST layer maps it to:

```text
404 Not Found
```

Example:

```text
Service Exception
      ↓
ExceptionMapper
      ↓
HTTP 404
```

---

# 36. Testing Exception Contracts

Verify the response:

```json
{
  "type": "employee-not-found",
  "title": "Employee Not Found",
  "status": 404,
  "detail": "Employee does not exist",
  "traceId": "abc123"
}
```

You should verify both:

```text
HTTP status
```

and:

```text
Error response structure
```

---

# 37. Testing CDI

Suppose:

```java
@ApplicationScoped
public class EmployeeService {
}
```

You need to verify:

```text
Bean discovery
Dependency injection
Scopes
Qualifiers
Lifecycle
Interceptors
Events
```

Example architecture:

```text
CDI Container
      ↓
EmployeeService
      ↓
EmployeeRepository
```

An integration test should validate that CDI actually wires the components.

---

# 38. Testing Dependency Injection

Unit test:

```text
Mock dependency
 ↓
Service
```

CDI integration test:

```text
CDI Container
 ↓
Real dependency
 ↓
Service
```

This catches problems such as:

```text
Unsatisfied dependency
Ambiguous dependency
Incorrect qualifier
Missing bean
Wrong scope
```

---

# 39. Testing EJB

If your application uses EJB:

```text
EJB Container
      ↓
EJB
      ↓
Business logic
```

Tests should verify container-provided behavior such as:

* transactions
* lifecycle
* security
* asynchronous behavior
* timers
* concurrency

A plain unit test cannot fully prove container behavior.

---

# 40. Testing JMS

Suppose:

```text
OrderService
      ↓
JMS Queue
      ↓
OrderConsumer
```

Test:

```text
Send message
 ↓
Consumer receives
 ↓
Business processing
 ↓
Database update
```

---

# 41. JMS Test Cases

Test:

### Successful message

```text
Message
 ↓
Processed
 ↓
ACK
```

### Processing failure

```text
Message
 ↓
Exception
 ↓
Rollback
 ↓
Redelivery
```

### Poison message

```text
Invalid message
 ↓
Repeated failure
 ↓
DLQ
```

### Duplicate message

```text
Same message twice
 ↓
Business effect occurs once
```

This tests idempotency.

---

# 42. Testing Idempotency

Suppose:

```text
OrderCreated(orderId=100)
```

arrives twice.

Expected:

```text
Database
Order 100 = created once
```

Not:

```text
Order 100
Order 100 duplicate effect
```

Test:

```text
send event
send same event again
verify business state
```

---

# 43. Testing Concurrency

Some bugs appear only under concurrent execution.

Example:

```text
100 requests
      ↓
same employee
      ↓
concurrent updates
```

Test:

* race conditions
* lost updates
* optimistic locking
* duplicate creation
* thread safety

---

# 44. Example Concurrent Test

Conceptually:

```java
ExecutorService executor =
        Executors.newFixedThreadPool(10);

for (int i = 0; i < 100; i++) {
    executor.submit(() ->
        service.updateEmployee(1L)
    );
}
```

Then verify the final state.

Be careful: concurrency tests can be nondeterministic if poorly designed.

---

# 45. Testing Optimistic Locking

Entity:

```java
@Version
private long version;
```

Scenario:

```text
Transaction A reads version 1
Transaction B reads version 1

A updates → version 2

B updates
    ↓
Conflict
```

Test should verify that the second update is handled correctly.

---

# 46. Testing Security

Security tests should cover:

```text
Authentication
Authorization
Session handling
CSRF
CORS
Input validation
Object-level authorization
Password handling
Token validation
```

---

# 47. Authentication Test Matrix

```text
No credentials
       ↓
401

Invalid credentials
       ↓
401

Valid credentials
       ↓
Authenticated

Valid user + insufficient permission
       ↓
403
```

---

# 48. Authorization Testing

Suppose:

```text
ADMIN
MANAGER
EMPLOYEE
```

Endpoint:

```text
DELETE /employees/100
```

Tests should verify:

```text
ADMIN → allowed
MANAGER → according to policy
EMPLOYEE → denied
Unauthenticated → denied
```

The exact permissions depend on your application requirements.

---

# 49. IDOR / Object-Level Authorization Test

Suppose:

```text
GET /employees/100
```

User owns employee 99.

Trying:

```text
GET /employees/100
```

should not automatically succeed.

Test:

```text
Authenticated user
       ↓
Requests another user's resource
       ↓
Authorization check
       ↓
Denied
```

This is a very important security test.

---

# 50. Testing Passwords

Never assert that:

```text
password == storedPassword
```

Instead verify:

```text
Password
   ↓
Hash
   ↓
Stored hash
```

and:

```text
Correct password → authentication succeeds
Wrong password → authentication fails
```

---

# 51. Contract Testing

Suppose Service A calls Service B.

```text
Employee Service
       ↓
Payment Service
```

Service A expects:

```json
{
  "employeeId": 100,
  "salary": 50000
}
```

Contract testing checks that the provider continues to satisfy the consumer's expected API contract.

This is especially useful in microservice environments.

---

# 52. Test Fixtures

Test fixture = predefined data/environment required by a test.

Example:

```text
Employee:
id = 1
name = Pradeep
email = p@example.com
```

Fixtures should be:

```text
Small
Readable
Relevant
Deterministic
```

---

# 53. Test Data Builder

Instead of:

```java
new Employee(
    1L,
    "Pradeep",
    "p@example.com",
    "IT",
    50000,
    ...
);
```

you can create:

```java
Employee employee =
    EmployeeTestData.builder()
        .name("Pradeep")
        .email("p@example.com")
        .salary(50000)
        .build();
```

This makes tests easier to read.

---

# 54. Test Isolation

Bad:

```text
Test A modifies DB
       ↓
Test B depends on Test A
```

If Test A fails:

```text
Test B also fails
```

Better:

```text
Test A → independent state
Test B → independent state
Test C → independent state
```

Tests should be independently executable.

---

# 55. Deterministic Tests

Avoid relying on:

```text
Current time
Random numbers
External internet
Production database
Other tests
Thread scheduling
```

unless the test specifically intends to verify those behaviors.

For time:

```java
Clock
```

can be injected.

Then tests can use a fixed clock.

---

# 56. Testing Time

Production:

```java
Instant.now()
```

Test:

```text
Fixed Clock
```

Conceptually:

```java
Clock clock =
    Clock.fixed(
        instant,
        ZoneOffset.UTC
    );
```

Then:

```text
Production → real clock
Test       → fixed clock
```

---

# 57. Testing External APIs

Suppose:

```text
EmployeeService
      ↓
Payment API
```

Unit test:

```text
Mock Payment API
```

Integration test:

```text
Mock server / contract environment
```

E2E:

```text
Real controlled external environment
```

Don't make every unit test depend on the internet.

---

# 58. Testing Timeouts

External dependency:

```text
Payment API
```

Test:

```text
Payment API takes 30 seconds
```

Your application should:

```text
timeout
 ↓
handle failure
 ↓
return appropriate response
```

not hang indefinitely.

---

# 59. Testing Retry

Suppose:

```text
Attempt 1 → failure
Attempt 2 → failure
Attempt 3 → success
```

Test that:

```text
Retry occurs
```

But also verify:

```text
Maximum retry count
Backoff
No infinite loop
Correct final failure behavior
```

---

# 60. Testing Circuit Breaker

Conceptually:

```text
External API
     ↓
Failures increase
     ↓
Circuit opens
     ↓
Requests fail fast
```

Then:

```text
Recovery period
     ↓
Half-open
     ↓
Test request
     ↓
Success
     ↓
Closed
```

Testing should verify state transitions.

---

# 61. Testing Caching

Suppose:

```text
GET employee
```

First request:

```text
Cache miss
 ↓
DB
 ↓
Cache
```

Second request:

```text
Cache hit
 ↓
Return
```

Tests should verify:

```text
Cache hit
Cache miss
Expiration
Invalidation
Stale data handling
Failure behavior
```

---

# 62. Testing Transactions + Messaging

This is a classic enterprise problem.

Suppose:

```text
DB update
+
JMS event
```

You need to define the consistency requirement.

Possible architecture:

```text
Transaction
 ├── DB change
 └── Outbox event
```

Then:

```text
Outbox Relay
 ↓
JMS
```

Tests should verify that a failed transaction doesn't produce an invalid business state.

---

# 63. Testing the Outbox Pattern

Test:

```text
Create Order
 ↓
DB transaction
 ├── Order
 └── Outbox Event
 ↓
Commit
```

Then:

```text
Relay
 ↓
Publish event
```

Failure scenarios:

```text
DB rollback
Relay failure
Duplicate publish
Consumer failure
```

---

# 64. Performance Testing

Functional correctness isn't enough.

You should measure:

```text
Throughput
Latency
Concurrency
CPU
Memory
DB connections
JMS queue depth
Error rate
```

Important metrics:

```text
p50
p95
p99
```

---

# 65. Why p95/p99 Matter

Suppose:

```text
1000 requests
```

Average:

```text
100 ms
```

But:

```text
p99 = 2 seconds
```

The average hides the slow tail.

For production systems, tail latency can be very important.

---

# 66. Load Testing

Example:

```text
100 users
 ↓
500 users
 ↓
1000 users
 ↓
5000 users
```

Measure:

```text
Response time
Error rate
DB pool usage
CPU
Memory
```

Find the point where the system degrades.

---

# 67. Stress Testing

Push beyond expected capacity.

```text
Expected:
1000 req/sec

Stress:
2000
3000
5000
```

Goal:

> Understand failure behavior and capacity limits.

---

# 68. Soak Testing

Run the application for a long period.

Example:

```text
24 hours
```

Look for:

* memory leaks
* connection leaks
* thread leaks
* growing queues
* resource exhaustion

---

# 69. Security Testing

Your enterprise testing strategy should include:

```text
SAST
DAST
Dependency scanning
Authentication testing
Authorization testing
Input validation
Secret scanning
API security testing
```

---

# 70. CI/CD Test Pipeline

A typical pipeline:

```text
Developer Push
      ↓
Compile
      ↓
Unit Tests
      ↓
Static Analysis
      ↓
Integration Tests
      ↓
Security Tests
      ↓
Package
      ↓
Deploy Test Environment
      ↓
E2E Tests
      ↓
Production Deployment
```

---

# 71. Test Pyramid in CI/CD

Fast feedback:

```text
Commit
 ↓
Unit tests
```

Then:

```text
Build
 ↓
Integration tests
```

Then:

```text
Deployment
 ↓
E2E
```

This prevents expensive tests from becoming the first feedback mechanism.

---

# 72. Test Naming

Bad:

```java
@Test
void test1() {}
```

Better:

```java
@Test
void shouldRejectEmployeeWhenEmailIsInvalid() {}
```

Even better:

```java
@Test
void createEmployee_shouldRejectDuplicateEmail() {}
```

A test name should describe behavior.

---

# 73. Arrange–Act–Assert

A useful structure:

```text
Arrange
   ↓
Act
   ↓
Assert
```

Example:

```java
@Test
void shouldRejectInvalidEmployee() {

    // Arrange
    Employee employee =
        invalidEmployee();

    // Act
    Exception exception =
        assertThrows(
            ValidationException.class,
            () -> service.create(employee)
        );

    // Assert
    assertEquals(
        "Email is invalid",
        exception.getMessage()
    );
}
```

---

# 74. Given–When–Then

Another useful style:

```text
Given
 ↓
When
 ↓
Then
```

Example:

```text
Given an employee already exists

When another employee is created
with the same email

Then the API returns 409 Conflict
```

This is especially useful for business-focused tests.

---

# 75. Don't Test Implementation Details

Bad:

```text
Verify private method X was called
Verify internal variable Y
Verify exact internal algorithm
```

Prefer:

```text
Given input
When operation occurs
Then business behavior is correct
```

Tests should survive reasonable refactoring.

---

# 76. Test Boundary Strategy

For each layer:

```text
REST
 ↓
Service
 ↓
Repository
 ↓
Database
```

Test differently.

### REST

Test:

```text
HTTP contract
```

### Service

Test:

```text
Business rules
```

### Repository

Test:

```text
Persistence behavior
```

### Database

Test:

```text
Schema/integration
```

---

# 77. Complete Testing Architecture

For your Employee Management Platform:

```text
                         Tests
                           |
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
     Unit Tests       Integration Tests     E2E
        |                  |                  |
        ↓                  ↓                  ↓
   Service Logic      DB/JMS/CDI          HTTP Flow
        |                  |                  |
        ↓                  ↓                  ↓
    Mockito            Containers        Full System
```

---

# 78. Employee Platform Test Plan

### Unit Tests

```text
EmployeeService
Validation
Business rules
DTO mapping
Exception handling
```

### Integration Tests

```text
JPA
PostgreSQL
Datasource
Transactions
CDI
JMS
```

### API Tests

```text
GET
POST
PUT
DELETE
Validation
Security
Error responses
```

### E2E

```text
Login
Create Employee
Persist
Publish Event
Consume Event
Audit
Query Employee
```

---

# 79. Example Service Unit Test

```java
@ExtendWith(MockitoExtension.class)
class EmployeeServiceTest {

    @Mock
    EmployeeRepository repository;

    @InjectMocks
    EmployeeService service;

    @Test
    void shouldRejectDuplicateEmail() {

        Employee employee =
            EmployeeTestData.valid();

        when(repository.existsByEmail(
            employee.getEmail()
        )).thenReturn(true);

        assertThrows(
            DuplicateEmployeeException.class,
            () -> service.create(employee)
        );
    }
}
```

---

# 80. Example Repository Integration Test

Conceptually:

```java
@Test
void shouldPersistAndLoadEmployee() {

    Employee employee =
        new Employee(
            "Pradeep",
            "pradeep@example.com"
        );

    repository.save(employee);

    Optional<Employee> result =
        repository.findById(employee.getId());

    assertTrue(result.isPresent());
    assertEquals(
        "Pradeep",
        result.get().getName()
    );
}
```

Here the important difference is that the repository interacts with a real test database.

---

# 81. Testing Database Constraints

Don't test only application validation.

Suppose database contains:

```sql
UNIQUE(email)
```

Test the database constraint too.

Why?

Because:

```text
Application validation
```

doesn't eliminate:

```text
Concurrent requests
```

Two requests could both pass:

```text
email doesn't exist
```

and then race to insert.

The database constraint is still important.

---

# 82. Testing Race Conditions

Consider:

```text
Request A → email available
Request B → email available

A → INSERT
B → INSERT
```

If the DB has:

```sql
UNIQUE(email)
```

one succeeds and one fails.

Your application should translate that correctly, for example:

```text
Database constraint violation
        ↓
Domain exception
        ↓
409 Conflict
```

---

# 83. Testing Pagination

For:

```text
GET /employees?page=0&size=20
```

Test:

```text
No employees
1 employee
20 employees
21 employees
1000 employees
```

Also test:

```text
Invalid page
Invalid size
Maximum size
Sorting
Filtering
```

---

# 84. Testing Idempotent APIs

Suppose:

```text
PUT /employees/100
```

called twice with the same request.

Expected final state should be the same.

Test:

```text
Request 1
Request 2
 ↓
same final state
```

For POST operations requiring idempotency:

```text
Idempotency-Key
```

can be tested explicitly.

---

# 85. Testing File Upload

If your application supports files, test:

```text
Valid file
Empty file
Large file
Wrong MIME type
Wrong extension
Malformed content
Duplicate file
Unauthorized upload
Path traversal attempts
```

Never trust:

```text
filename
Content-Type
extension
```

provided by the client.

---

# 86. Testing Logging

Don't normally assert every log message.

Instead test important observability behavior where needed:

```text
Correlation ID exists
Security event is audited
Critical failure is recorded
```

Avoid making tests fragile because wording of logs changes.

---

# 87. Testing Health Endpoints

Test:

```text
Liveness
Readiness
```

Example:

```text
Database healthy
→ READY

Database unavailable
→ NOT READY
```

Depending on your application's health policy, the exact behavior may differ.

---

# 88. Testing Graceful Shutdown

Test:

```text
Shutdown initiated
       ↓
No new requests
       ↓
Existing requests finish
       ↓
Resources released
```

Especially important for:

```text
JMS consumers
Scheduled jobs
Long-running transactions
```

---

# 89. Testing Configuration

Configuration errors can cause production failures.

Test:

```text
Missing datasource
Invalid JNDI name
Missing required property
Invalid timeout
Invalid URL
Invalid security configuration
```

Fail fast where appropriate.

---

# 90. Contract of Configuration

Example:

```text
EmployeeDS must exist
OrderQueue must exist
Security realm must exist
```

Treat infrastructure requirements as part of the deployable application's contract.

---

# 91. Day 27 Mini Project

Build:

# Production-Grade Employee Testing Suite

Your project should contain:

```text
employee-platform/
│
├── src/test/java/
│   │
│   ├── unit/
│   │   ├── EmployeeServiceTest.java
│   │   └── EmployeeValidatorTest.java
│   │
│   ├── integration/
│   │   ├── EmployeeRepositoryIT.java
│   │   ├── TransactionIT.java
│   │   └── JmsIT.java
│   │
│   ├── api/
│   │   ├── EmployeeResourceIT.java
│   │   └── SecurityIT.java
│   │
│   └── e2e/
│       └── EmployeeWorkflowIT.java
```

---

# 92. Required Test Scenarios

Implement at least:

### Employee

```text
Create employee
Read employee
Update employee
Delete employee
```

### Validation

```text
Invalid name
Invalid email
Missing required fields
```

### Database

```text
Persistence
Unique constraint
Transaction rollback
```

### REST

```text
201
400
401
403
404
409
500
```

### Messaging

```text
Publish event
Consume event
Retry
Duplicate event
```

### Security

```text
Unauthenticated
Authenticated
Unauthorized
```

---

# 93. Test Automation Goal

Your goal should be:

```text
git push
    ↓
Build
    ↓
Unit Tests
    ↓
Integration Tests
    ↓
Security Tests
    ↓
Package
```

No manual testing should be required for basic regression validation.

---

# 94. 30-Day Testing Challenge

Today, implement these **30 tests**:

### Unit — 10

1. Valid employee
2. Invalid employee
3. Duplicate email
4. Employee not found
5. Salary validation
6. Department validation
7. DTO conversion
8. Business rule
9. Exception mapping
10. Service behavior

### Integration — 10

11. Repository save
12. Repository find
13. Repository update
14. Repository delete
15. Unique constraint
16. Transaction commit
17. Transaction rollback
18. JPA relationship
19. Datasource
20. Database migration

### API/E2E — 10

21. Create API
22. Read API
23. Update API
24. Delete API
25. Validation
26. Authentication
27. Authorization
28. JMS event
29. Duplicate event
30. Complete employee workflow

---

# 95. 40 Important Interview Questions

### Fundamentals

1. What is unit testing?
2. What is integration testing?
3. What is system testing?
4. What is E2E testing?
5. Explain the testing pyramid.
6. What makes a good unit test?
7. What makes a test deterministic?
8. What is test isolation?
9. What is a test fixture?
10. What is Arrange-Act-Assert?

### JUnit

11. What is JUnit?
12. `@BeforeEach` vs `@BeforeAll`?
13. `@AfterEach` vs `@AfterAll`?
14. What is parameterized testing?
15. How do you test exceptions?
16. How do you organize test classes?

### Mockito

17. What is mocking?
18. What is stubbing?
19. Mock vs spy?
20. `when()` vs `verify()`?
21. Why shouldn't everything be mocked?
22. When should you use a real dependency?

### Database

23. How do you test JPA?
24. How do you test transactions?
25. H2 vs PostgreSQL test database?
26. What is Testcontainers?
27. How do you test database constraints?
28. How do you test rollback?

### REST

29. How do you test REST APIs?
30. What HTTP statuses should be tested?
31. How do you test validation?
32. How do you test authentication?
33. How do you test authorization?
34. How do you test idempotency?

### Enterprise

35. How do you test JMS?
36. How do you test CDI?
37. How do you test EJB?
38. How do you test asynchronous processing?
39. How do you test distributed workflows?
40. What belongs in a CI/CD test pipeline?

---

# 96. Architect-Level Question

Imagine you are designing an enterprise banking application.

Architecture:

```text
                 API
                  |
              AccountService
             /             \
            /               \
       PostgreSQL           JMS
            |                 |
            ↓                 ↓
       Transaction        Notification
```

Your testing strategy should not be:

```text
"Write 1000 unit tests."
```

Instead:

```text
Business rules
       ↓
Unit tests

Database behavior
       ↓
Integration tests

Messaging behavior
       ↓
Integration tests

REST contract
       ↓
API tests

Complete money-transfer workflow
       ↓
E2E tests

Concurrency
       ↓
Concurrency tests

Capacity
       ↓
Performance tests

Security
       ↓
Security tests
```

This is the mindset of an architect.

---

# 97. What Not to Do

### ❌ Test only happy paths

```text
Valid request → success
```

You also need:

```text
Invalid
Unauthorized
Duplicate
Timeout
Failure
Concurrent
Unavailable dependency
```

---

### ❌ Mock everything

You can accidentally create tests that don't resemble production.

---

### ❌ Use production database

Never make normal automated tests depend on production.

---

### ❌ Depend on test execution order

Tests should be independent.

---

### ❌ Ignore concurrency

Many enterprise bugs are race conditions.

---

### ❌ Ignore infrastructure

If your application depends on:

```text
PostgreSQL
JMS
Redis
External APIs
```

you need tests at the appropriate integration boundary.

---

# 98. Testing Strategy for Your 2-Year Architect Goal

When designing any system, ask:

```text
1. What are the business rules?
        ↓
2. What should be unit tested?
        ↓
3. What infrastructure needs integration testing?
        ↓
4. What APIs need contract testing?
        ↓
5. What workflows need E2E testing?
        ↓
6. What failures need testing?
        ↓
7. What security properties need testing?
        ↓
8. What concurrency risks exist?
        ↓
9. What performance limits exist?
        ↓
10. How will tests run in CI/CD?
```

This is much more valuable than memorizing testing frameworks.

---

# 99. Day 27 Mastery Checklist

You should now be able to explain:

* [ ] Testing pyramid
* [ ] Unit testing
* [ ] Integration testing
* [ ] System testing
* [ ] E2E testing
* [ ] JUnit
* [ ] Assertions
* [ ] Parameterized tests
* [ ] Mockito
* [ ] Mock vs stub
* [ ] Test fixtures
* [ ] Test isolation
* [ ] Testcontainers
* [ ] H2 limitations
* [ ] Repository testing
* [ ] JPA testing
* [ ] JDBC testing
* [ ] Transaction testing
* [ ] REST testing
* [ ] CDI testing
* [ ] EJB testing
* [ ] JMS testing
* [ ] Idempotency testing
* [ ] Concurrency testing
* [ ] Security testing
* [ ] Authorization testing
* [ ] Contract testing
* [ ] Performance testing
* [ ] Load testing
* [ ] Stress testing
* [ ] Soak testing
* [ ] CI/CD testing
* [ ] Test strategy

---

# 100. Final Mental Model

Memorize:

```text
                         ENTERPRISE APPLICATION
                                  |
            ┌─────────────────────┼─────────────────────┐
            ↓                     ↓                     ↓
       Business Logic       Infrastructure          APIs
            |                     |                     |
            ↓                     ↓                     ↓
       UNIT TESTS          INTEGRATION TESTS       API TESTS
                                  |
                    ┌─────────────┼─────────────┐
                    ↓             ↓             ↓
                  JPA           JMS           CDI
                    ↓             ↓             ↓
                Database      Broker        Container
                                  |
                                  ↓
                           SYSTEM / E2E TESTS
                                  |
                                  ↓
                       PERFORMANCE + SECURITY
                                  |
                                  ↓
                              CI/CD
```

### Golden Rule

> **Unit tests prove business behavior, integration tests prove components work with real infrastructure, API tests prove contracts, and end-to-end tests prove complete business workflows.**

For your solutions-architect goal, always ask:

> **“What failure or incorrect behavior would be expensive in production, and what test would catch it before deployment?”**

---

## Day 28 → Production Deployment & DevOps

Tomorrow you'll combine the entire J2EE/Jakarta EE journey with production operations:

```text
Source Code
    ↓
Maven Build
    ↓
Automated Tests
    ↓
WAR/JAR/EAR
    ↓
Docker
    ↓
Application Server
    ↓
AWS / Azure / GCP
    ↓
Load Balancer
    ↓
Database
    ↓
JMS
    ↓
Monitoring
    ↓
Logging
    ↓
CI/CD
    ↓
Production
```

You'll focus on **Dockerizing Jakarta EE, CI/CD, deployment strategies, environment configuration, secrets, HTTPS, reverse proxies, cloud deployment, observability, rollback, and production troubleshooting.**
