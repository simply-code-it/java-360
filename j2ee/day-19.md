# Day 19 — EJB Fundamentals

**Date: October 21, 2026**
**J2EE/Jakarta EE 30-Day Mastery**

Today you move into one of the most important parts of traditional enterprise Java: **Enterprise JavaBeans (EJB)**.

The goal is not just to memorize EJB annotations. You should understand **why EJB existed, what the container provides, how transactions/security/concurrency are handled, and when EJB still makes sense in modern Jakarta EE applications.**

---

# 1. What is EJB?

**EJB = Enterprise JavaBeans.**

EJB is a server-side component model for building enterprise applications.

An EJB container can provide infrastructure services such as:

* Dependency injection
* Transaction management
* Security
* Concurrency management
* Lifecycle management
* Instance pooling
* Asynchronous execution
* Timers
* Remote invocation
* Interceptors

Conceptually:

```text
Client
   |
   v
REST / Servlet / Messaging
   |
   v
EJB Container
   |
   +---- Security
   +---- Transactions
   +---- Lifecycle
   +---- Pooling
   +---- Concurrency
   +---- Interceptors
   |
   v
EJB
   |
   v
JPA / JDBC
   |
   v
Database
```

The important idea is:

> **EJB is not just a Java class. It is a managed enterprise component whose execution is controlled by the application server.**

---

# 2. Why was EJB created?

Enterprise applications have difficult infrastructure requirements.

Imagine a banking operation:

```text
Transfer ₹10,000
```

You need:

1. Authenticate the user
2. Authorize the operation
3. Start transaction
4. Debit account A
5. Credit account B
6. Commit transaction
7. Roll back if something fails
8. Log the operation
9. Handle concurrent requests
10. Possibly execute asynchronously

Without a container, developers would repeatedly implement this infrastructure.

EJB introduced container-managed services.

Instead of:

```java
beginTransaction();

try {
    debit();
    credit();

    commit();
} catch (Exception e) {
    rollback();
}
```

you can define the transaction semantics declaratively.

For example:

```java
@Stateless
public class TransferService {

    public void transfer(...) {
        debit();
        credit();
    }
}
```

The container can manage the transaction around the method.

---

# 3. EJB Evolution

EJB has gone through several generations.

```text
EJB 1.x
   ↓
EJB 2.x
   ↓
EJB 3.x
   ↓
Java EE
   ↓
Jakarta EE
```

Older EJB versions were much more complicated.

Modern EJB 3.x dramatically simplified development using:

* Annotations
* POJOs
* Dependency injection
* Declarative transactions
* Interceptors

Legacy:

```text
Home Interface
Remote Interface
Bean class
Deployment descriptors
Container-generated artifacts
```

Modern:

```java
@Stateless
public class OrderService {
    ...
}
```

This is a major architectural improvement.

---

# 4. Types of Session Beans

There are three important session bean types.

```text
Session Beans
│
├── Stateless Session Bean
│
├── Stateful Session Bean
│
└── Singleton Session Bean
```

---

# 5. Stateless Session Bean

Annotation:

```java
@Stateless
public class EmployeeService {
}
```

A stateless bean does **not maintain conversational state for a particular client between method calls**.

Example:

```java
@Stateless
public class EmployeeService {

    public Employee findEmployee(Long id) {
        // database lookup
        return ...;
    }

    public void createEmployee(Employee employee) {
        // persist employee
    }
}
```

Each invocation is independent.

---

# 6. Stateless Bean Mental Model

Suppose there are several bean instances:

```text
EJB Pool

+----------------------+
| EmployeeService #1   |
+----------------------+
| EmployeeService #2   |
+----------------------+
| EmployeeService #3   |
+----------------------+
| EmployeeService #4   |
+----------------------+
```

Requests arrive:

```text
Request A ──> Bean #1

Request B ──> Bean #3

Request C ──> Bean #2
```

The client shouldn't assume that every request reaches the same instance.

Therefore:

### Don't do this

```java
@Stateless
public class ShoppingService {

    private Long userId;

    public void setUser(Long userId) {
        this.userId = userId;
    }
}
```

The bean instance is managed by the container and should not be used as per-user conversational storage.

---

# 7. Stateful Session Bean

Annotation:

```java
@Stateful
public class ShoppingCart {
}
```

A stateful session bean can maintain conversational state for a client.

Example:

```java
@Stateful
public class ShoppingCart {

    private final List<Long> productIds = new ArrayList<>();

    public void addProduct(Long productId) {
        productIds.add(productId);
    }

    public void removeProduct(Long productId) {
        productIds.remove(productId);
    }

    public List<Long> getProducts() {
        return productIds;
    }
}
```

The cart maintains state across method invocations associated with the conversational instance.

Conceptually:

```text
Client
  |
  +--> addProduct(101)
  |
  +--> addProduct(202)
  |
  +--> getProducts()
             |
             v
       [101, 202]
```

---

# 8. Stateful vs Stateless

| Feature              | Stateless                | Stateful                      |
| -------------------- | ------------------------ | ----------------------------- |
| Conversational state | No                       | Yes                           |
| State between calls  | No client-specific state | Yes                           |
| Pooling              | Common                   | Different lifecycle semantics |
| Scalability          | Generally easier         | More complex                  |
| Typical use          | Services                 | Conversational workflows      |
| Example              | OrderService             | ShoppingCart                  |

For most business services:

```text
Stateless
```

is the natural starting point.

---

# 9. Singleton Session Bean

Annotation:

```java
@Singleton
public class ConfigurationService {
}
```

A singleton bean has one logical bean instance per application.

Example:

```java
@Singleton
public class ApplicationConfiguration {

    private String applicationName;

    public String getApplicationName() {
        return applicationName;
    }

    public void setApplicationName(String name) {
        this.applicationName = name;
    }
}
```

Conceptually:

```text
Application
     |
     v
+----------------------+
| ConfigurationService |
|       Singleton      |
+----------------------+
     ^
     |
All callers
```

---

# 10. Singleton Concurrency

Singleton beans are especially important because multiple requests may access them.

You need to understand concurrency management.

Example:

```java
@Singleton
@Lock(LockType.READ)
public class ConfigurationService {

    public String getConfig() {
        return "production";
    }
}
```

For write operations:

```java
@Lock(LockType.WRITE)
public void updateConfig(String value) {
    ...
}
```

Conceptually:

```text
READ
READ
READ
READ

can execute concurrently
```

while:

```text
WRITE
```

can provide exclusive access depending on the locking configuration.

---

# 11. `@Lock`

Common values:

```java
@Lock(LockType.READ)
```

and:

```java
@Lock(LockType.WRITE)
```

Example:

```java
@Singleton
public class CounterService {

    private long count;

    @Lock(LockType.READ)
    public long getCount() {
        return count;
    }

    @Lock(LockType.WRITE)
    public void increment() {
        count++;
    }
}
```

Default singleton locking behavior is important to understand when diagnosing concurrency and performance.

---

# 12. EJB Lifecycle

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
Business method calls
   |
   v
@PreDestroy
   |
   v
Destroy
```

Example:

```java
@Stateless
public class EmployeeService {

    @PostConstruct
    public void initialize() {
        System.out.println("Bean initialized");
    }

    public void createEmployee() {
        ...
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Bean destroyed");
    }
}
```

---

# 13. Dependency Injection into EJB

EJB works naturally with CDI.

```java
@Stateless
public class OrderService {

    @Inject
    private PaymentService paymentService;
}
```

You can also inject other resources.

For example:

```java
@PersistenceContext
private EntityManager entityManager;
```

---

# 14. EJB + JPA

This is a very important enterprise combination.

```java
@Stateless
public class EmployeeService {

    @PersistenceContext
    private EntityManager entityManager;

    public void create(Employee employee) {
        entityManager.persist(employee);
    }
}
```

Architecture:

```text
REST Resource
      |
      v
EJB Service
      |
      v
EntityManager
      |
      v
JPA Provider
      |
      v
Database
```

---

# 15. EJB and Transactions

One of EJB's biggest strengths historically has been declarative transaction management.

Example:

```java
@Stateless
public class BankService {

    public void transfer(
            Long from,
            Long to,
            BigDecimal amount) {

        debit(from, amount);
        credit(to, amount);
    }
}
```

With container-managed transactions, the container can conceptually perform:

```text
BEGIN
   |
debit()
   |
credit()
   |
COMMIT
```

If an appropriate failure occurs:

```text
BEGIN
   |
debit()
   |
credit()
   |
ERROR
   |
ROLLBACK
```

---

# 16. Container-Managed Transactions

CMT = **Container-Managed Transactions**.

The container manages:

* Begin
* Commit
* Rollback
* Transaction propagation
* Transaction context

Example:

```java
@Stateless
public class OrderService {

    public void placeOrder(Order order) {
        saveOrder(order);
        reserveInventory(order);
        createPayment(order);
    }
}
```

The entire business operation can execute within the transaction configured for the EJB method.

---

# 17. Bean-Managed Transactions

BMT = **Bean-Managed Transactions**.

The bean explicitly controls transactions.

Example conceptually:

```java
@Stateless
@TransactionManagement(TransactionManagementType.BEAN)
public class PaymentService {

    @Resource
    private UserTransaction transaction;

    public void processPayment() throws Exception {

        transaction.begin();

        try {
            debit();
            credit();

            transaction.commit();

        } catch (Exception e) {
            transaction.rollback();
            throw e;
        }
    }
}
```

Use BMT only when you genuinely need programmatic transaction control.

For most standard business operations:

```text
CMT > BMT
```

as a design starting point.

---

# 18. CMT vs BMT

| Feature                    | CMT       | BMT         |
| -------------------------- | --------- | ----------- |
| Transaction controlled by  | Container | Developer   |
| Boilerplate                | Low       | Higher      |
| Declarative                | Yes       | No          |
| Fine-grained control       | Lower     | Higher      |
| Typical enterprise service | Common    | Specialized |

---

# 19. Transaction Attributes

EJB supports transaction attributes such as:

```text
REQUIRED
REQUIRES_NEW
MANDATORY
SUPPORTS
NOT_SUPPORTED
NEVER
```

The most important two:

### REQUIRED

```text
Use existing transaction
OR
create one if none exists
```

### REQUIRES_NEW

```text
Suspend existing transaction
Create a new transaction
```

Example:

```java
@TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
public void audit() {
    ...
}
```

This can be useful when an audit operation needs its own transaction boundary.

---

# 20. EJB Security

EJB integrates with Jakarta EE security.

You can restrict access based on roles.

Example:

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

Deny access:

```java
@DenyAll
public void internalOperation() {
}
```

Allow authenticated users:

```java
@PermitAll
public void getEmployee(Long id) {
}
```

The important distinction:

```text
Authentication
      ↓
Who are you?

Authorization
      ↓
What are you allowed to do?
```

---

# 21. EJB Interceptors

Interceptors are useful for cross-cutting concerns.

Example:

```java
@AroundInvoke
public Object logMethod(InvocationContext context)
        throws Exception {

    long start = System.currentTimeMillis();

    try {
        return context.proceed();
    } finally {
        long duration =
            System.currentTimeMillis() - start;

        System.out.println(
            context.getMethod().getName()
            + " took "
            + duration
            + " ms");
    }
}
```

Potential use cases:

```text
Logging
Metrics
Auditing
Tracing
Security checks
Performance measurement
```

---

# 22. EJB Asynchronous Methods

EJB can support asynchronous business operations.

Example:

```java
@Stateless
public class NotificationService {

    @Asynchronous
    public void sendEmail(String email) {
        // expensive operation
    }
}
```

Caller:

```java
notificationService.sendEmail(email);
```

The caller doesn't necessarily need to wait for the operation to complete.

Conceptually:

```text
Request
  |
  +----> Main operation
  |
  +----> Async notification
              |
              v
          Background execution
```

This is useful for operations such as:

* Notifications
* Background processing
* Non-critical secondary tasks

But for durable messaging between systems, **Jakarta Messaging/Kafka/etc.** is generally a different architectural concern.

---

# 23. EJB Timers

EJB also supports scheduled execution.

Example concept:

```java
@Schedule(
    hour = "2",
    minute = "0",
    second = "0"
)
public void runDailyJob() {
    ...
}
```

Conceptually:

```text
02:00 AM
   |
   v
EJB Timer
   |
   v
Cleanup job
```

Potential applications:

```text
Database cleanup
Expired order processing
Report generation
Subscription processing
Scheduled reconciliation
```

---

# 24. EJB Remote vs Local Access

EJB can support different invocation models.

### Local

Components within the same application/server environment can interact locally.

Conceptually:

```text
Application
   |
   +--> EJB A
   |
   +--> EJB B
```

### Remote

Remote EJB invocation allows a client in another JVM/environment to invoke an EJB through supported remote mechanisms.

Conceptually:

```text
JVM A
Client
  |
  | remote invocation
  v
JVM B
EJB
```

Remote EJB is an important historical enterprise Java technology.

However, don't automatically choose remote EJB simply because two components are in different services.

For modern distributed systems, APIs and messaging are often architectural alternatives.

---

# 25. Business Interfaces

An EJB can expose a business interface.

Example:

```java
public interface EmployeeService {

    Employee findById(Long id);

    void create(Employee employee);
}
```

Implementation:

```java
@Stateless
public class EmployeeServiceBean
        implements EmployeeService {

    @Override
    public Employee findById(Long id) {
        ...
    }

    @Override
    public void create(Employee employee) {
        ...
    }
}
```

This supports interface-driven design.

---

# 26. EJB + REST

A very useful architecture is:

```text
HTTP Client
    |
    v
Jakarta REST Resource
    |
    v
EJB Service
    |
    v
JPA
    |
    v
Database
```

Example:

```java
@Path("/employees")
@Produces(MediaType.APPLICATION_JSON)
public class EmployeeResource {

    @Inject
    private EmployeeService service;

    @GET
    @Path("/{id}")
    public Employee getEmployee(
            @PathParam("id") Long id) {

        return service.findById(id);
    }
}
```

The REST resource should not contain all business logic.

---

# 27. EJB + CDI — Important Distinction

You learned CDI on Day 18.

Think of them this way:

### CDI

Primarily provides:

```text
Dependency Injection
Bean lifecycle
Scopes
Qualifiers
Events
Interceptors
Decorators
```

### EJB

Adds enterprise component semantics such as:

```text
Container-managed transactions
Security
Concurrency management
Asynchronous methods
Timers
EJB-specific lifecycle/container behavior
```

They are complementary technologies.

---

# 28. CDI Bean vs EJB

| Feature                           | CDI Bean                   | EJB                     |
| --------------------------------- | -------------------------- | ----------------------- |
| Dependency injection              | Yes                        | Yes                     |
| Scopes                            | Yes                        | Yes/integrates with CDI |
| Interceptors                      | Yes                        | Yes                     |
| Transactions                      | Via Jakarta Transactions   | Strong EJB integration  |
| Security                          | Yes                        | Strong integration      |
| Async methods                     | CDI mechanisms differ      | `@Asynchronous`         |
| Timers                            | Different mechanisms       | EJB Timer               |
| Stateful conversational component | CDI scopes can model state | `@Stateful`             |
| Enterprise container services     | General                    | Strong EJB model        |

Don't think:

```text
EJB = obsolete
```

A better understanding is:

```text
EJB is a specialized enterprise component model.
```

Modern Jakarta EE applications may use CDI heavily and EJB selectively.

---

# 29. Stateless EJB Example — Employee Service

Let's build a realistic example.

### Entity

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    private String department;

    private BigDecimal salary;

    // getters/setters
}
```

### EJB

```java
@Stateless
public class EmployeeService {

    @PersistenceContext
    private EntityManager entityManager;

    public Employee create(Employee employee) {
        entityManager.persist(employee);
        return employee;
    }

    public Employee findById(Long id) {
        return entityManager.find(Employee.class, id);
    }

    public void delete(Long id) {

        Employee employee =
            entityManager.find(Employee.class, id);

        if (employee != null) {
            entityManager.remove(employee);
        }
    }
}
```

This is a clean enterprise service.

---

# 30. Add Authorization

```java
@Stateless
public class EmployeeService {

    @RolesAllowed("HR")
    public void deleteEmployee(Long id) {
        ...
    }
}
```

Now the operation has an explicit security boundary.

---

# 31. Add Transaction Semantics

```java
@Stateless
public class EmployeeService {

    @TransactionAttribute(
        TransactionAttributeType.REQUIRED
    )
    public void createEmployee(Employee employee) {

        // business operation
    }
}
```

Conceptually:

```text
HTTP Request
     |
     v
REST Resource
     |
     v
EJB
     |
     +---- Security
     |
     +---- Transaction
     |
     +---- Business Logic
     |
     v
JPA
     |
     v
Database
```

---

# 32. EJB Exception Handling

You need to understand application exceptions.

For example:

```java
public class EmployeeNotFoundException
        extends Exception {

    public EmployeeNotFoundException(String message) {
        super(message);
    }
}
```

An application exception can communicate business-level failure.

Examples:

```text
EmployeeNotFound
InsufficientBalance
DuplicateOrder
InvalidStateTransition
PaymentRejected
```

This is different from infrastructure failures such as:

```text
SQLException
Database connection failure
Network failure
```

---

# 33. EJB Pooling

One important EJB container responsibility is managing bean instances.

For stateless beans:

```text
EJB Pool

+-----------+
| Bean #1   |
+-----------+
| Bean #2   |
+-----------+
| Bean #3   |
+-----------+
| Bean #4   |
+-----------+
```

Requests can be dispatched to available instances.

This helps the container manage resources efficiently.

Therefore, don't assume:

```java
private int counter;
```

is a safe global request counter in a stateless bean.

It isn't.

---

# 34. EJB Thread Safety

This is an important interview topic.

### Stateless EJB

Don't use mutable instance state as request-specific state.

Bad:

```java
@Stateless
public class OrderService {

    private Order currentOrder;

    public void process(Order order) {
        currentOrder = order;
    }
}
```

Better:

```java
public void process(Order order) {
    // use local variable
}
```

Use method-local state:

```java
public void process(Order order) {

    Payment payment = calculatePayment(order);

    ...
}
```

---

# 35. EJB and Database Connection Management

Avoid manually managing database connections when container-managed resources are available.

Prefer:

```java
@PersistenceContext
private EntityManager entityManager;
```

or an injected datasource where appropriate.

Avoid:

```java
DriverManager.getConnection(...);
```

inside every business method.

The container/server can manage:

* Connection pooling
* Datasources
* Resource lifecycle
* Transaction integration

---

# 36. Production Architecture

A production Jakarta EE application might look like:

```text
                    Internet
                       |
                       v
                Load Balancer
                       |
                       v
              Jakarta EE Server
                       |
          +------------+------------+
          |                         |
          v                         v
   REST Resources             Security
          |
          v
     CDI / EJB Layer
          |
    +-----+------+
    |            |
    v            v
 Business      Validation
 Services
    |
    v
 JPA / DAO
    |
    v
 Connection Pool
    |
    v
 Database
```

Cross-cutting:

```text
Logging
Metrics
Tracing
Security
Configuration
```

---

# 37. Where EJB Fits in Your Architecture

For your solutions-architect goal, think in layers:

```text
Presentation
    |
    +-- Jakarta REST
    +-- Servlet
    +-- JSP
    |
Application / Business
    |
    +-- CDI
    +-- EJB
    +-- Domain Services
    |
Persistence
    |
    +-- JPA
    +-- JDBC
    |
Infrastructure
    |
    +-- Database
    +-- Messaging
    +-- Cache
```

The key architectural question is not:

> "Where can I use EJB?"

Instead ask:

> **"Which container-managed enterprise capability does this component actually need?"**

---

# 38. When to Use Stateless EJB

Good candidates:

```text
Order processing
Payment processing
Employee management
Account operations
Business workflows
Transactional services
```

Example:

```java
@Stateless
public class OrderService {
    public void placeOrder(Order order) {
        ...
    }
}
```

---

# 39. When to Use Stateful EJB

Useful when a genuine conversational workflow exists.

Example:

```text
Shopping Cart
Loan Application Wizard
Multi-step Business Process
Long-running User Interaction
```

Example:

```java
@Stateful
public class LoanApplication {

    private Applicant applicant;
    private List<Document> documents;

    public void addDocument(Document document) {
        documents.add(document);
    }
}
```

But always consider whether the state could instead live in:

* Database
* Cache
* HTTP session
* External workflow system

---

# 40. When to Use Singleton EJB

Good candidates:

```text
Application-wide configuration
Shared coordination state
Scheduled jobs
Application-wide caches
Startup initialization
```

Be extremely careful with shared mutable state.

---

# 41. EJB Startup

A singleton can be initialized when the application starts.

Example:

```java
@Singleton
@Startup
public class ApplicationInitializer {

    @PostConstruct
    public void initialize() {
        System.out.println(
            "Application initialization"
        );
    }
}
```

Potential uses:

```text
Load configuration
Initialize caches
Validate environment
Register application resources
```

---

# 42. Important EJB Annotations

Memorize these:

```text
@Stateless
@Stateful
@Singleton
@Startup

@Asynchronous
@Schedule

@RolesAllowed
@PermitAll
@DenyAll

@TransactionAttribute
@TransactionManagement

@Local
@Remote

@Lock
@ConcurrencyManagement
```

Lifecycle:

```text
@PostConstruct
@PreDestroy
```

Injection:

```text
@Inject
@EJB
@PersistenceContext
@Resource
```

---

# 43. `@EJB` vs `@Inject`

Traditional EJB injection:

```java
@EJB
private EmployeeService employeeService;
```

CDI injection:

```java
@Inject
private EmployeeService employeeService;
```

Modern Jakarta EE applications frequently use CDI injection because CDI provides a broader dependency-injection programming model.

Understand both for interviews and legacy enterprise applications.

---

# 44. EJB vs Spring Service

You will almost certainly encounter this comparison in Java interviews.

### EJB

```java
@Stateless
public class OrderService {
}
```

### Spring

```java
@Service
public class OrderService {
}
```

They are not literally identical technologies.

EJB is part of the Jakarta EE enterprise component model.

Spring's `@Service` is a Spring-managed component.

Both can participate in architectures such as:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

The infrastructure and programming models differ.

---

# 45. EJB vs Microservice

EJB:

```text
Application Server
      |
      +--- EJB
      +--- EJB
      +--- REST
      +--- JPA
```

Microservices:

```text
Service A
   |
   +--- Database A

Service B
   |
   +--- Database B

Service C
   |
   +--- Database C
```

EJB is a component model.

Microservices are an architectural style.

Therefore:

> **EJB and microservices are not direct alternatives.**

An EJB can exist inside a service-oriented or microservice-style application.

---

# 46. EJB Request Flow

Let's put everything together.

Suppose:

```http
POST /api/orders
```

Flow:

```text
Client
  |
  v
HTTP
  |
  v
Jakarta REST
  |
  v
Security
  |
  v
OrderResource
  |
  v
OrderService EJB
  |
  +--> Transaction begins
  |
  +--> Validate order
  |
  +--> Persist order
  |
  +--> Reserve inventory
  |
  +--> Create payment
  |
  v
Transaction commit
  |
  v
HTTP 201
```

If something fails:

```text
OrderService
     |
     v
Exception
     |
     v
Rollback
     |
     v
Exception Mapper
     |
     v
HTTP Error Response
```

This is the mental model you should master.

---

# 47. Production EJB Rules

### Rule 1 — Prefer stateless services

```java
@Stateless
```

for ordinary business services.

### Rule 2 — Don't store request-specific state

Avoid mutable instance fields in stateless beans.

### Rule 3 — Define transaction boundaries intentionally

Know exactly which operation should be atomic.

### Rule 4 — Secure business operations

Don't rely only on UI restrictions.

Use authorization at the business boundary.

### Rule 5 — Don't put SQL everywhere

Use:

```text
EJB → Service → JPA/DAO
```

### Rule 6 — Don't use EJB for everything

Use CDI where CDI is sufficient.

### Rule 7 — Be careful with singleton state

Concurrency bugs can be subtle.

### Rule 8 — Don't confuse async execution with durable messaging

For reliable cross-system delivery, consider messaging infrastructure.

---

# 48. Hands-On Exercise 1 — Employee EJB

Create:

```text
Employee
EmployeeService
EmployeeResource
```

Requirements:

```text
POST /employees
GET /employees/{id}
DELETE /employees/{id}
```

Architecture:

```text
REST
 ↓
EJB
 ↓
JPA
 ↓
Database
```

---

# 49. Hands-On Exercise 2 — Banking

Create:

```java
@Stateless
public class BankService {

    public void transfer(
        Long sourceAccount,
        Long destinationAccount,
        BigDecimal amount) {

        // debit
        // credit
    }
}
```

Requirements:

* Transactional transfer
* Rollback on failure
* Authorization
* Validation
* Audit

---

# 50. Hands-On Exercise 3 — Stateful Shopping Cart

Create:

```java
@Stateful
public class ShoppingCart {
}
```

Implement:

```text
addItem()
removeItem()
getItems()
clear()
getTotal()
```

Understand exactly how conversational state differs from stateless service execution.

---

# 51. Hands-On Exercise 4 — Singleton Configuration

Create:

```java
@Singleton
@Startup
public class ConfigurationService {
}
```

Implement:

```text
loadConfiguration()
getConfiguration()
updateConfiguration()
```

Add appropriate concurrency handling.

---

# 52. Hands-On Exercise 5 — Async Notification

Create:

```java
@Stateless
public class NotificationService {

    @Asynchronous
    public void sendNotification(String message) {
        ...
    }
}
```

Call it from:

```java
OrderService
```

Understand:

```text
Main business operation
        |
        +---- async notification
```

---

# 53. Hands-On Exercise 6 — Scheduled Cleanup

Create an EJB timer that periodically removes expired records.

Concept:

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

# 54. Mini Project — Enterprise Order Service

Build:

## Enterprise Order Management System

### Components

```text
OrderResource
       |
       v
OrderService EJB
       |
       +---- PaymentService EJB
       |
       +---- InventoryService EJB
       |
       +---- NotificationService EJB
       |
       v
JPA
       |
       v
PostgreSQL
```

### Requirements

#### Customer

```text
id
name
email
```

#### Product

```text
id
name
price
stock
```

#### Order

```text
id
customerId
status
total
createdAt
```

#### OrderItem

```text
id
orderId
productId
quantity
price
```

---

# 55. Order Workflow

```text
POST /orders
      |
      v
Validate request
      |
      v
OrderService
      |
      +--> Check inventory
      |
      +--> Reserve inventory
      |
      +--> Create order
      |
      +--> Process payment
      |
      +--> Commit transaction
      |
      +--> Async notification
      |
      v
201 Created
```

Think about what should happen if:

```text
Inventory succeeds
Payment fails
```

You need to reason about:

```text
Transaction boundaries
Compensation
Distributed transactions
Messaging
Idempotency
```

These concepts become particularly important on Days 21–23.

---

# 56. Day 19 Interview Questions

You should be able to answer these without notes.

### Beginner

1. What is EJB?
2. Why was EJB created?
3. What is an EJB container?
4. What are session beans?
5. What is a stateless session bean?
6. What is a stateful session bean?
7. What is a singleton bean?
8. What is EJB lifecycle?
9. What does `@Stateless` mean?
10. What does `@Stateful` mean?

### Intermediate

11. Stateless vs stateful?
12. Stateless vs singleton?
13. How does EJB pooling work?
14. How does EJB handle transactions?
15. What is CMT?
16. What is BMT?
17. CMT vs BMT?
18. What is `REQUIRED`?
19. What is `REQUIRES_NEW`?
20. What is `@TransactionAttribute`?
21. What is `@Asynchronous`?
22. What are EJB timers?
23. What is `@RolesAllowed`?
24. What is `@PermitAll`?
25. What is `@DenyAll`?

### Advanced

26. How does a stateless EJB scale?
27. Why shouldn't stateless EJBs maintain conversational state?
28. How does singleton EJB concurrency work?
29. What is `@Lock`?
30. How does EJB integrate with JPA?
31. EJB vs CDI?
32. EJB vs Spring `@Service`?
33. EJB vs REST?
34. EJB vs microservices?
35. Local vs remote EJB?
36. What are application exceptions?
37. How do interceptors work with EJB?
38. How does security integrate with EJB?
39. How does transaction propagation work?
40. When should you choose EJB over a CDI bean?

---

# 57. Architect-Level Questions

These are more important for your long-term solutions-architect goal.

### Scenario 1

You have:

```text
REST API
   ↓
Business Service
   ↓
Database
```

Would you use EJB, CDI, or another framework?

You should evaluate:

```text
Transaction requirements
Security requirements
Async requirements
Timer requirements
Application-server environment
Existing platform
Operational requirements
Team expertise
```

Don't choose technology merely because it is available.

---

### Scenario 2

You have:

```text
10,000 concurrent users
```

Should you use Stateful EJB?

You should investigate:

```text
Where state lives
Memory consumption
Session affinity
Passivation
Horizontal scaling
Failover
External state storage
```

---

### Scenario 3

You have:

```text
OrderService
PaymentService
InventoryService
```

Should they all be EJBs?

The real architectural question is:

```text
Same application?
Same transaction boundary?
Same deployment unit?
Separate services?
Separate databases?
Independent scaling?
Independent ownership?
```

That is the beginning of architecture thinking.

---

# 58. EJB Mental Model

Memorize this:

```text
                 EJB CONTAINER
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
   Lifecycle      Security     Transactions
        |             |             |
        +-------------+-------------+
                      |
                      v
                  EJB Bean
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
       JPA          Async         Timers
        |
        v
    Database
```

---

# 59. The Three EJB Types

Memorize:

```text
@Stateless
    ↓
No conversational state

@Stateful
    ↓
Client-specific conversational state

@Singleton
    ↓
One logical application-wide instance
```

---

# 60. Day 19 Golden Rule

The most important thing to understand today is:

> **EJB provides container-managed enterprise behavior around business components.**

Think:

```text
EJB
 |
 +-- Lifecycle
 |
 +-- Transactions
 |
 +-- Security
 |
 +-- Concurrency
 |
 +-- Async execution
 |
 +-- Timers
 |
 +-- Interceptors
 |
 +-- Pooling
 |
 v
Business Logic
```

And your overall Jakarta EE architecture now becomes:

```text
                    Client
                      |
                      v
              Jakarta REST / Servlet
                      |
                      v
                 CDI / EJB
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      Security   Transactions   Events
          |           |
          +-----------+
                      |
                      v
                     JPA
                      |
                      v
                   JDBC
                      |
                      v
                  Database
```

## Day 19 Mastery Checklist

Before moving to Day 20, you should be able to explain and implement:

* [ ] What EJB is
* [ ] Why EJB exists
* [ ] EJB container responsibilities
* [ ] Stateless session beans
* [ ] Stateful session beans
* [ ] Singleton beans
* [ ] EJB lifecycle
* [ ] EJB pooling
* [ ] EJB concurrency
* [ ] `@Lock`
* [ ] CMT
* [ ] BMT
* [ ] Transaction attributes
* [ ] `REQUIRED`
* [ ] `REQUIRES_NEW`
* [ ] EJB security
* [ ] `@RolesAllowed`
* [ ] EJB interceptors
* [ ] Async EJB
* [ ] EJB timers
* [ ] Local vs remote EJB
* [ ] EJB + CDI
* [ ] EJB + JPA
* [ ] EJB + REST
* [ ] EJB vs Spring Service
* [ ] EJB vs CDI
* [ ] EJB vs microservices
* [ ] Stateless vs stateful vs singleton
* [ ] Enterprise transaction boundaries

### Your Day 19 target

Build a **production-style Order Management API** using:

```text
Jakarta REST
     ↓
EJB
     ↓
CDI
     ↓
JPA
     ↓
PostgreSQL
```

with:

```text
Authentication
Authorization
Transactions
Validation
Exception handling
Async notification
EJB interceptor
Scheduled cleanup
```

**Day 20** will take EJB much deeper: **advanced EJB lifecycle, transaction propagation, concurrency, asynchronous processing, timers, security, interceptors, exception semantics, remote EJB, pooling, passivation, and production architecture.**
