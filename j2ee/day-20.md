# Day 20 — Advanced EJB

**J2EE/Jakarta EE 30-Day Mastery**
**Date: October 22, 2026**

Day 19 gave you the fundamentals of EJB. Today we go deeper into the areas that matter for **enterprise production systems and solutions-architect-level design**:

* Advanced lifecycle
* Transaction propagation
* Concurrency
* Async execution
* Timers
* Security
* Interceptors
* Exception semantics
* Stateful bean passivation
* Remote EJB
* Pooling
* EJB + JPA
* EJB + messaging
* Production architecture
* Performance and failure scenarios

---

# 1. Advanced EJB Architecture

A useful mental model is:

```text
                    Client
                      |
                      v
             REST / Servlet / JMS
                      |
                      v
               EJB Container
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
  Security       Transactions    Interceptors
       |              |              |
       +--------------+--------------+
                      |
                      v
                  EJB Bean
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
         JPA        Async       Timer
          |
          v
       Database
```

The container is responsible for much more than creating Java objects.

---

# 2. EJB Invocation Flow

Suppose:

```java
orderService.placeOrder(order);
```

The conceptual flow is:

```text
Caller
  |
  v
EJB Proxy
  |
  v
Security Check
  |
  v
Transaction Handling
  |
  v
Interceptor
  |
  v
EJB Business Method
  |
  v
JPA
  |
  v
Database
  |
  v
Interceptor
  |
  v
Transaction Commit
  |
  v
Caller
```

This is why EJB can provide enterprise behavior without putting infrastructure code inside every business method.

---

# 3. EJB Proxy

When you inject an EJB:

```java
@Inject
private OrderService orderService;
```

you should not mentally think:

```text
new OrderService()
```

Instead, the container manages the component and invocation.

Conceptually:

```text
Caller
  |
  v
Proxy
  |
  +--> security
  +--> transaction
  +--> interception
  |
  v
Actual EJB instance
```

This proxy/container model explains many EJB behaviors.

---

# 4. Why You Shouldn't Use `new` for an EJB

Don't do:

```java
OrderService service = new OrderService();
```

Why?

You bypass container management.

You may lose:

* Dependency injection
* Transactions
* Security
* Interceptors
* Lifecycle management
* Other container services

Prefer:

```java
@Inject
private OrderService service;
```

or appropriate EJB/CDI injection.

---

# 5. Stateless EJB Lifecycle

A simplified lifecycle:

```text
                     Container
                        |
                        v
                 Create instance
                        |
                        v
              Dependency injection
                        |
                        v
                  @PostConstruct
                        |
                        v
                     Ready
                        |
                        v
                  Business calls
                        |
                        v
                     Pool
                        |
                        v
                 @PreDestroy
                        |
                        v
                    Destroy
```

Important:

> A stateless EJB instance can be reused for different invocations.

Therefore don't store client-specific state in fields.

---

# 6. Stateless Pooling

Imagine:

```text
              EJB Pool

        +------------------+
        | EmployeeService  |
        +------------------+
        | EmployeeService  |
        +------------------+
        | EmployeeService  |
        +------------------+
        | EmployeeService  |
        +------------------+
```

Requests:

```text
Request 1 ──> Instance A
Request 2 ──> Instance C
Request 3 ──> Instance B
Request 4 ──> Instance A
```

You shouldn't assume:

```text
User A always gets Instance A
```

That's not the purpose of stateless EJBs.

---

# 7. Stateful EJB Lifecycle

Stateful beans have conversational state.

Conceptually:

```text
Client
  |
  v
Create Stateful Bean
  |
  v
Initialize
  |
  v
Call 1
  |
  v
Call 2
  |
  v
Call 3
  |
  v
Remove
  |
  v
Destroy
```

Example:

```java
@Stateful
public class ShoppingCart {

    private final List<Long> products =
        new ArrayList<>();

    public void add(Long productId) {
        products.add(productId);
    }

    public List<Long> getProducts() {
        return products;
    }

    @Remove
    public void checkout() {
        // finalize cart
    }
}
```

The `@Remove` concept is associated with stateful session beans.

---

# 8. Stateful Passivation

Stateful session beans can have their conversational state passivated by the container.

Conceptually:

```text
Active
  |
  v
Passivation
  |
  v
Stored state
  |
  v
Activation
  |
  v
Active
```

Why?

Suppose you have:

```text
100,000
stateful conversations
```

Keeping every bean instance actively in memory could consume significant resources.

The container can manage lifecycle/resource usage.

---

# 9. Passivation Constraints

This creates an important design concern.

Stateful bean state must be compatible with the container's passivation requirements.

For example:

```java
@Stateful
public class ShoppingCart {

    private List<Product> products;
}
```

The objects involved need to satisfy the relevant passivation requirements.

Avoid putting arbitrary non-passivatable resources into conversational state.

Bad conceptual example:

```java
private Socket socket;
```

or:

```java
private Thread thread;
```

inside stateful conversational state.

---

# 10. Stateful EJB Design Rule

Keep state:

```text
Small
Serializable/passivation-compatible
Business-relevant
Bounded
```

Don't use a stateful EJB as a giant application cache.

---

# 11. Singleton EJB

Singleton:

```java
@Singleton
public class ConfigurationService {
}
```

One logical instance exists within the application.

This makes concurrency extremely important.

```text
Request A ----\
Request B -----+--> Singleton
Request C ----/
Request D ----/
```

Multiple callers can access it.

---

# 12. Singleton Concurrency Management

Two important models:

```text
Container-managed concurrency
Bean-managed concurrency
```

Container-managed concurrency allows the container to control concurrent access.

Example:

```java
@Singleton
@Lock(LockType.READ)
public class ConfigurationService {

    public String getValue() {
        return "...";
    }
}
```

---

# 13. Read vs Write Locks

Suppose:

```java
@Lock(LockType.READ)
public String getConfig() {
    ...
}
```

Multiple readers can potentially execute concurrently.

But:

```java
@Lock(LockType.WRITE)
public void updateConfig() {
    ...
}
```

requires exclusive access according to the container's concurrency semantics.

Think:

```text
READ + READ + READ
       ↓
Concurrent access

WRITE
       ↓
Exclusive access
```

---

# 14. Avoid Singleton Bottlenecks

This can be dangerous:

```java
@Singleton
public class GlobalService {

    @Lock(LockType.WRITE)
    public void process() {
        // expensive 5-second operation
    }
}
```

If every request needs the write lock:

```text
Request 1 → waits
Request 2 → waits
Request 3 → waits
Request 4 → waits
```

Your singleton becomes a bottleneck.

Always examine:

* Lock scope
* Critical section
* Execution time
* Contention
* Throughput

---

# 15. Bean-Managed Concurrency

A singleton can also use bean-managed concurrency.

The bean itself becomes responsible for thread safety.

That means you may need mechanisms such as:

```java
synchronized
```

or:

```java
AtomicInteger
```

or:

```java
ReentrantLock
```

But don't choose bean-managed concurrency simply because it looks more flexible.

Container-managed concurrency is usually easier to reason about in a Jakarta EE environment.

---

# 16. Transaction Attributes — Deep Dive

You learned these yesterday.

Now let's reason about them architecturally.

```text
REQUIRED
REQUIRES_NEW
MANDATORY
SUPPORTS
NOT_SUPPORTED
NEVER
```

---

# 17. REQUIRED

Suppose:

```text
A → B
```

A has a transaction.

B uses:

```java
@TransactionAttribute(
    TransactionAttributeType.REQUIRED
)
```

Then:

```text
A
 |
 | Transaction T1
 v
B
 |
 | Same T1
 v
Database
```

If B is called without a transaction:

```text
B
 |
 +--> Container starts T1
```

Therefore:

```text
REQUIRED
=
join existing
or
create new
```

---

# 18. REQUIRES_NEW

Suppose:

```text
A
 |
 | T1
 v
B
```

B uses:

```java
@TransactionAttribute(
    TransactionAttributeType.REQUIRES_NEW
)
```

Then:

```text
T1
 |
 | suspended
 v
T2
 |
 B
 |
 v
commit T2
 |
 v
resume T1
```

This is useful for independent operations such as certain audit scenarios.

---

# 19. Transaction Example

Imagine:

```java
public void placeOrder() {

    createOrder();

    saveAudit();

    chargePayment();
}
```

Suppose:

```text
placeOrder() → T1
saveAudit() → REQUIRES_NEW
```

The conceptual flow becomes:

```text
T1 START
   |
   +-- createOrder()
   |
   +-- suspend T1
          |
          +-- T2 saveAudit()
          |
          +-- COMMIT T2
   |
   +-- resume T1
   |
   +-- chargePayment()
   |
   +-- COMMIT T1
```

This is powerful but should be used deliberately.

---

# 20. Why `REQUIRES_NEW` Can Be Dangerous

Suppose:

```text
Main transaction fails
```

but:

```text
Audit transaction succeeds
```

Now you could have:

```text
Business operation → ROLLED BACK

Audit → COMMITTED
```

That may be exactly what you want—or it may create confusing business semantics.

Always ask:

> Should this operation succeed independently of the parent transaction?

---

# 21. `MANDATORY`

```java
@TransactionAttribute(
    TransactionAttributeType.MANDATORY
)
```

Means:

```text
A transaction MUST already exist.
```

If not:

```text
Exception
```

Useful when a method should never execute outside a transaction.

---

# 22. `SUPPORTS`

```text
Transaction exists
    ↓
Join it

No transaction
    ↓
Run without one
```

Useful when transaction participation is optional.

---

# 23. `NOT_SUPPORTED`

Means:

```text
If transaction exists:
    suspend it

Execute without transaction
```

Potential use cases:

* Long-running non-transactional work
* Certain read-only operations
* External operations that should not hold a DB transaction

Be careful: holding a database transaction while waiting for a remote service can be expensive.

---

# 24. `NEVER`

Requires:

```text
No transaction
```

If one exists:

```text
Exception
```

Less common, but important for interviews.

---

# 25. Transaction Attribute Cheat Sheet

| Attribute     | Existing TX      | No TX  |
| ------------- | ---------------- | ------ |
| REQUIRED      | Join             | Create |
| REQUIRES_NEW  | Suspend + create | Create |
| MANDATORY     | Join             | Error  |
| SUPPORTS      | Join             | No TX  |
| NOT_SUPPORTED | Suspend          | No TX  |
| NEVER         | Error            | No TX  |

Memorize this table.

---

# 26. EJB Exception Semantics

You need to distinguish:

```text
Application Exception
System Exception
```

Business exception:

```text
InsufficientBalanceException
```

Infrastructure failure:

```text
Database connection failure
```

They have different implications for transaction handling and bean lifecycle.

---

# 27. Application Exceptions

Example:

```java
@ApplicationException(rollback = true)
public class InsufficientBalanceException
        extends Exception {
}
```

This communicates that the exception represents an application/business condition.

The `rollback` configuration is important because some application exceptions do not automatically imply rollback in the same way system exceptions do.

---

# 28. System Exceptions

Examples:

```text
Unexpected infrastructure failure
RuntimeException
Database failure
Container-level failure
```

These can cause transaction rollback and may cause the EJB instance to be discarded depending on the circumstances.

Important architectural lesson:

> Don't use exceptions casually. Understand whether the failure is business-level or infrastructure-level.

---

# 29. Exception Design

Good:

```text
InsufficientBalanceException
EmployeeNotFoundException
DuplicateOrderException
InvalidOrderStateException
```

Avoid exposing low-level errors:

```text
SQLException
NullPointerException
Internal database details
```

directly through REST responses.

Instead:

```text
EJB
 ↓
Business exception
 ↓
REST exception mapper
 ↓
Safe HTTP error
```

---

# 30. EJB Security

You can secure business methods.

```java
@RolesAllowed("ADMIN")
public void deleteEmployee(Long id) {
    ...
}
```

Multiple roles:

```java
@RolesAllowed({"ADMIN", "HR"})
public void updateEmployee(Employee employee) {
    ...
}
```

---

# 31. Security at the Business Boundary

Don't rely only on:

```text
Frontend button hidden
```

or:

```text
REST endpoint hidden
```

The business operation should have authorization controls.

Think:

```text
Client
  ↓
Authentication
  ↓
REST
  ↓
Authorization
  ↓
Business Service
```

---

# 32. Method-Level Security

Imagine:

```java
@Stateless
public class AccountService {

    @RolesAllowed("CUSTOMER")
    public Account getAccount(Long id) {
        ...
    }

    @RolesAllowed("ADMIN")
    public void closeAccount(Long id) {
        ...
    }
}
```

This expresses business authorization directly.

However, role checks alone may not be enough.

You also need **object-level authorization**.

For example:

```text
Customer A requests:

GET /accounts/999
```

Even if authenticated as CUSTOMER, you must determine whether account 999 belongs to Customer A.

---

# 33. Interceptors

Interceptors allow cross-cutting behavior.

Architecture:

```text
EJB Method
   |
   v
Interceptor
   |
   v
Business Method
   |
   v
Interceptor after-processing
```

Example:

```java
@AroundInvoke
public Object log(InvocationContext context)
        throws Exception {

    long start = System.nanoTime();

    try {
        return context.proceed();
    } finally {
        long duration =
            System.nanoTime() - start;

        System.out.println(
            context.getMethod().getName()
            + ": "
            + duration
            + " ns");
    }
}
```

---

# 34. What Belongs in an Interceptor?

Good examples:

```text
Logging
Metrics
Tracing
Audit
Security-related cross-cutting checks
Performance measurement
Correlation IDs
```

Bad examples:

```text
Core order calculation
Payment business rules
Inventory rules
Customer eligibility
```

Those belong in business components.

---

# 35. Interceptor Execution

Suppose:

```text
LoggingInterceptor
AuditInterceptor
MetricsInterceptor
OrderService
```

Conceptually:

```text
Request
  |
  v
Logging
  |
  v
Audit
  |
  v
Metrics
  |
  v
OrderService
  |
  v
Metrics
  |
  v
Audit
  |
  v
Logging
  |
  v
Response
```

This is effectively a chain around the business invocation.

---

# 36. Async EJB

You can use:

```java
@Asynchronous
public void sendEmail(String email) {
    ...
}
```

This allows the caller to continue without waiting for completion in the normal synchronous manner.

Useful for:

```text
Email
Notification
Non-critical background work
Secondary processing
```

---

# 37. Async Does NOT Mean Durable

This is extremely important.

Consider:

```text
OrderService
    |
    +---- @Asynchronous sendEmail()
```

If the application crashes:

```text
Order committed
Email task may not complete
```

If the requirement is:

> "The message must not be lost."

then you should consider durable messaging:

```text
OrderService
    |
    v
JMS / Kafka
    |
    v
Notification Consumer
```

This distinction matters enormously in system design interviews.

---

# 38. EJB Timers

Timers allow scheduled execution.

Example:

```java
@Singleton
public class CleanupService {

    @Schedule(
        hour = "2",
        minute = "0",
        second = "0"
    )
    public void cleanup() {
        // cleanup expired records
    }
}
```

Architecture:

```text
Timer
  ↓
CleanupService
  ↓
JPA
  ↓
Database
```

---

# 39. Timer Use Cases

Examples:

```text
Expire abandoned orders
Generate reports
Clean temporary records
Reconcile payments
Process subscriptions
Archive old data
```

For complex enterprise scheduling, also evaluate dedicated scheduling/workflow infrastructure.

---

# 40. EJB + JPA Transaction Boundary

One of the most useful patterns:

```java
@Stateless
public class OrderService {

    @PersistenceContext
    private EntityManager entityManager;

    public void placeOrder(Order order) {

        entityManager.persist(order);

        // other database operations
    }
}
```

The service operation can define the business transaction.

Think:

```text
One business operation
        ↓
One intentional transaction boundary
```

---

# 41. Don't Put Transactions Around Everything

Bad architectural design:

```text
HTTP Request
   ↓
Transaction
   ↓
Remote API call
   ↓
Wait 20 seconds
   ↓
Database
```

Now your database transaction may remain open while waiting on a remote service.

That can cause:

* Connection pool exhaustion
* Lock retention
* Increased latency
* Reduced throughput

Prefer designing the transaction boundary around the actual atomic work.

---

# 42. Distributed Transaction Problem

Suppose:

```text
Order DB
Payment DB
Inventory DB
```

You might want:

```text
BEGIN
  |
  +-- Order
  +-- Payment
  +-- Inventory
  |
COMMIT
```

If they're separate transactional resources, you may enter distributed transaction territory.

This is why:

```text
EJB transactions
```

do not automatically solve:

```text
distributed system consistency
```

Days 21–23 will go much deeper into messaging and distributed transactions.

---

# 43. Remote EJB

Remote EJB allows a client in another JVM/environment to invoke an enterprise bean using the supported remote EJB mechanism.

Conceptually:

```text
Application A
     |
     | Remote invocation
     v
Application B
     |
     v
EJB
```

Remote EJB can propagate enterprise context according to the platform/container capabilities.

---

# 44. Remote EJB vs REST

### Remote EJB

```text
Java/Jakarta EE oriented
Strong enterprise invocation model
```

### REST

```text
HTTP
Language-independent
Internet-friendly
Broad interoperability
```

If you have:

```text
Java EE Server A
       |
       v
Java EE Server B
```

remote EJB may be relevant.

If you have:

```text
Java
Node.js
Python
Mobile
External partners
```

REST or messaging is often a more interoperable choice.

---

# 45. EJB + JMS

EJB and Jakarta Messaging work very well together.

Example:

```text
OrderService
    |
    v
JMS Queue
    |
    v
NotificationConsumer
```

Consumer:

```java
@MessageDriven
public class OrderNotificationConsumer {

    public void onMessage(Message message) {
        // process message
    }
}
```

This leads into Day 21.

---

# 46. Message-Driven Beans

MDBs are EJB components designed to consume messages.

Conceptually:

```text
Producer
   |
   v
Queue
   |
   v
Message-Driven Bean
   |
   v
Business Logic
```

Unlike session beans, MDBs aren't normally called through ordinary synchronous business method invocation.

They're triggered by messages.

---

# 47. MDB Use Cases

Excellent use cases:

```text
Order events
Email processing
Payment events
Inventory events
Background workflows
Integration events
```

Example:

```text
REST POST /orders
       |
       v
OrderService
       |
       v
JMS Queue
       |
       v
OrderProcessor MDB
```

---

# 48. EJB and Idempotency

Suppose a message arrives twice:

```text
OrderCreated #123
OrderCreated #123
```

Your consumer should ideally avoid processing the same business operation twice.

Possible strategy:

```text
Message ID
   ↓
Idempotency table
   ↓
Already processed?
   ├── Yes → ignore
   └── No  → process
```

EJB transactions can help make the processing and idempotency record atomic when using compatible transactional resources.

---

# 49. Production Architecture — Order Processing

Let's combine everything.

```text
                  Client
                    |
                    v
             Load Balancer
                    |
                    v
             Jakarta REST
                    |
                    v
              OrderService
                 EJB
                    |
        +-----------+-----------+
        |                       |
        v                       v
      JPA                    JMS
        |                       |
        v                       v
   Order DB               Queue/Topic
                                |
                                v
                         Message Consumer
                                |
                         +------+------+
                         |             |
                         v             v
                    Notification   Integration
```

Cross-cutting:

```text
Security
Logging
Metrics
Tracing
Validation
Transactions
```

---

# 50. Advanced EJB Design Exercise

Design:

## Banking Transaction Platform

Requirements:

```text
Transfer money
Deposit
Withdraw
Transaction history
Audit
Notifications
Scheduled reconciliation
```

Architecture:

```text
REST
 ↓
AccountResource
 ↓
AccountService EJB
 ↓
JPA
 ↓
Database
```

Additional components:

```text
AuditInterceptor
NotificationService
ReconciliationTimer
JMS
```

---

# 51. Banking Transfer Flow

```text
POST /transfers
       |
       v
Security
       |
       v
AccountService
       |
       v
BEGIN TRANSACTION
       |
       +--> Debit Account A
       |
       +--> Credit Account B
       |
       +--> Save Transaction
       |
       v
COMMIT
       |
       v
Publish notification/event
```

Important question:

> Should notification publishing be inside the same database transaction?

The answer depends on the delivery and consistency requirements.

This leads naturally to:

```text
Transactional messaging
Outbox pattern
Distributed transactions
Event-driven architecture
```

---

# 52. Advanced Coding Challenge

Implement:

```java
@Stateless
public class TransferService {
}
```

Methods:

```text
transfer()
deposit()
withdraw()
getBalance()
```

Requirements:

* Transaction boundary
* Authorization
* Validation
* Business exceptions
* JPA
* Audit
* Logging interceptor
* Notification
* Idempotency

---

# 53. Architect-Level Exercise

You have:

```text
10,000 requests/sec
```

and:

```text
OrderService
PaymentService
InventoryService
NotificationService
```

Evaluate:

### Question 1

Should every operation be synchronous?

Consider:

```text
Latency
Coupling
Failure propagation
User experience
```

### Question 2

Should notification be synchronous?

Consider:

```text
Order success
Notification failure
```

### Question 3

Should payment and inventory be in one transaction?

Consider:

```text
Separate databases
Separate services
Distributed transactions
Compensation
```

### Question 4

Should you use Stateful EJB?

Consider:

```text
State size
Number of users
Horizontal scaling
Passivation
External state
```

These are the kinds of questions that move you from **Java developer → system designer**.

---

# 54. EJB Performance Considerations

Watch:

```text
Database connection pool
EJB pool
Thread pool
Transaction duration
Lock contention
Remote calls
JPA query performance
N+1 queries
Memory usage
Async queue capacity
```

A slow application is rarely fixed by simply increasing EJB pool size.

Always find the bottleneck first.

---

# 55. Common EJB Anti-Patterns

### Anti-pattern 1

Huge Stateful EJB:

```text
Millions of objects
+
Large conversational state
```

Potential memory/scaling problem.

---

### Anti-pattern 2

Singleton as global mutable cache.

```java
@Singleton
public class GlobalCache {
    private Map<String, Object> data;
}
```

You now have to reason carefully about:

* Concurrency
* Memory
* Consistency
* Eviction
* Horizontal deployment

---

### Anti-pattern 3

Long database transactions.

```text
BEGIN
 ↓
DB
 ↓
Remote API
 ↓
Wait 30 sec
 ↓
DB
 ↓
COMMIT
```

Avoid this unless the architecture specifically requires it.

---

### Anti-pattern 4

Using async EJB as a message broker.

```text
@Asynchronous
```

doesn't provide the same durable delivery semantics as a messaging system.

---

### Anti-pattern 5

Putting business logic in interceptors.

Keep cross-cutting logic separate from domain behavior.

---

# 56. EJB Production Checklist

Before deploying an EJB-based application, ask:

### Transactions

* What is the transaction boundary?
* What causes rollback?
* Are remote calls inside transactions?
* Is `REQUIRES_NEW` really necessary?

### Concurrency

* Is the bean stateless?
* Does it have mutable fields?
* Is singleton state protected?
* Is there lock contention?

### Security

* Who can invoke the method?
* Are roles enforced?
* Is object-level authorization implemented?

### Performance

* How many DB connections?
* How long are transactions?
* Are queries optimized?
* Are remote calls blocking?

### Reliability

* What happens if a database fails?
* What happens if a message is delivered twice?
* What happens if async processing fails?

---

# 57. Day 20 Interview Questions

## EJB

1. What happens when an EJB method is invoked?
2. What is an EJB proxy?
3. Why shouldn't we instantiate EJB using `new`?
4. How does stateless EJB pooling work?
5. How does stateful EJB differ from stateless?
6. What is passivation?
7. Why does stateful EJB support passivation?
8. What are passivation requirements?
9. How does singleton EJB concurrency work?
10. What is `@Lock`?

## Transactions

11. Explain all EJB transaction attributes.
12. `REQUIRED` vs `REQUIRES_NEW`.
13. What happens when `REQUIRES_NEW` is called?
14. What does `MANDATORY` mean?
15. What does `SUPPORTS` mean?
16. What does `NOT_SUPPORTED` mean?
17. What does `NEVER` mean?
18. How does transaction rollback happen?
19. What are application exceptions?
20. What are system exceptions?

## Security

21. What is `@RolesAllowed`?
22. `@PermitAll` vs `@DenyAll`.
23. Why should authorization exist at the business layer?
24. What is object-level authorization?

## Async/Timers

25. What is `@Asynchronous`?
26. When would you use asynchronous EJB?
27. Why isn't async EJB equivalent to durable messaging?
28. What are EJB timers?
29. When should you use timers?

## Architecture

30. EJB vs CDI?
31. EJB vs Spring `@Service`?
32. EJB vs REST?
33. EJB vs microservices?
34. Remote EJB vs REST?
35. EJB + JPA architecture?
36. EJB + JMS architecture?
37. What is an MDB?
38. How would you implement idempotent message processing?
39. How would you design an order-processing system using EJB?
40. How would you prevent a singleton EJB from becoming a bottleneck?

---

# 58. 10 Advanced Scenario Questions

Try answering these yourself.

### Scenario 1

A stateless bean has:

```java
private User currentUser;
```

Why is this dangerous?

---

### Scenario 2

A singleton method takes 10 seconds and uses:

```java
@Lock(LockType.WRITE)
```

What scalability issue can occur?

---

### Scenario 3

A transaction calls an external payment API that takes 15 seconds.

What problems could this create?

---

### Scenario 4

You use:

```text
REQUIRES_NEW
```

for audit logging.

The parent transaction rolls back.

What happens to the audit transaction?

---

### Scenario 5

An asynchronous email task executes after an order transaction commits.

The application crashes while sending email.

What guarantees do you have?

---

### Scenario 6

A JMS message is delivered twice.

How should the consumer behave?

---

### Scenario 7

A Stateful EJB contains a large object graph.

What concerns should you investigate?

---

### Scenario 8

A customer is authenticated as `CUSTOMER`, but requests another customer's account.

Is authentication sufficient?

---

### Scenario 9

Three EJBs access three different databases.

Can you automatically assume one transaction provides perfect distributed consistency?

---

### Scenario 10

You have 20,000 concurrent users.

Would Stateful EJB automatically be your session-management solution?

Explain the scalability implications.

---

# 59. Day 20 Mastery Project

Upgrade yesterday's **Enterprise Order Management System**.

Build:

```text
                   REST API
                      |
                      v
               OrderService EJB
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    Payment       Inventory      Notification
      EJB            EJB             EJB
       |              |              |
       +--------------+--------------+
                      |
                      v
                     JPA
                      |
                      v
                  PostgreSQL
```

Add:

```text
@RolesAllowed
@TransactionAttribute
@Asynchronous
@Schedule
@AroundInvoke
@Singleton
@Stateless
```

And implement:

```text
Order creation
Payment
Inventory reservation
Authorization
Audit
Async notification
Scheduled cleanup
Exception handling
Idempotency
Transaction management
```

---

# 60. Day 20 Mental Model

Memorize this architecture:

```text
                    EJB CONTAINER
                         |
      +------------------+------------------+
      |                  |                  |
      v                  v                  v
 Lifecycle           Security          Transactions
      |                  |                  |
      +------------------+------------------+
                         |
                         v
                    EJB Proxy
                         |
                         v
                    EJB Instance
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
         JPA            JMS            Async
          |              |              |
          v              v              v
      Database         Queue          Worker
```

And remember the three major session bean models:

```text
@Stateless
    ↓
Independent business operations

@Stateful
    ↓
Conversational state

@Singleton
    ↓
Application-wide shared component
```

## Day 20 Golden Rule

> **Advanced EJB is primarily about understanding what the container does around your business code—and deliberately designing transaction boundaries, concurrency, security, lifecycle, and failure behavior.**

For your solutions-architect journey, don't stop at knowing annotations such as `@Stateless` or `@Transactional`. Be able to answer:

```text
Where is the transaction boundary?
What happens when a dependency fails?
Where does state live?
How does this scale horizontally?
What happens under concurrency?
What happens when a message is duplicated?
What happens after a crash?
Which operations must be atomic?
Which operations can be asynchronous?
Which boundaries should be synchronous?
```

Those questions are the bridge from **enterprise Java knowledge to architecture design**.

**Day 21 → Jakarta Messaging (JMS): queues, topics, producers, consumers, MDBs, acknowledgements, transactions, delivery semantics, retries, dead-letter queues, ordering, idempotency, and enterprise asynchronous architecture.**
