# Day 18 — CDI & Dependency Injection in Jakarta EE

**Date:** October 20, 2026
**Goal:** Master **CDI (Contexts and Dependency Injection)** so you can build loosely coupled, testable, maintainable enterprise Java applications.

CDI is one of the most important Jakarta EE concepts because it connects many enterprise components:

```text
REST Resource
      ↓
Service
      ↓
Repository / DAO
      ↓
Other Services
```

without every class manually creating its dependencies.

---

# 1. What You Must Master Today

By the end of Day 18, understand:

1. Dependency Injection
2. Inversion of Control
3. CDI
4. CDI container
5. CDI beans
6. Bean discovery
7. `@Inject`
8. Constructor injection
9. Field injection
10. Setter injection
11. CDI scopes
12. `@ApplicationScoped`
13. `@RequestScoped`
14. `@SessionScoped`
15. `@Dependent`
16. `@ConversationScoped`
17. Qualifiers
18. `@Named`
19. Producer methods
20. Producer fields
21. Disposers
22. Alternatives
23. `@Priority`
24. Lifecycle callbacks
25. CDI events
26. Interceptors
27. Decorators
28. Proxies
29. Dependency graphs
30. CDI testing
31. CDI vs Spring DI
32. Production architecture

---

# 2. The Problem CDI Solves

Consider:

```java
public class EmployeeService {

    private EmployeeRepository repository;

    public EmployeeService() {
        repository = new EmployeeRepository();
    }
}
```

This creates tight coupling.

Now imagine:

```text
EmployeeService
      ↓
EmployeeRepository
      ↓
Database
```

You can't easily replace the repository with:

```text
MockEmployeeRepository
```

for testing.

CDI solves this by allowing the container to provide dependencies.

---

# 3. Dependency

A dependency is something a class needs to perform its work.

Example:

```java
public class EmployeeService {

    private EmployeeRepository repository;

}
```

Here:

```text
EmployeeService
      ↓
depends on
      ↓
EmployeeRepository
```

---

# 4. Dependency Injection

Instead of creating the dependency:

```java
new EmployeeRepository()
```

the container supplies it.

Conceptually:

```text
CDI Container
     │
     ├── EmployeeRepository
     │
     └── EmployeeService
              │
              ▼
       injected repository
```

---

# 5. Inversion of Control

Without IoC:

```text
Application
    ↓
creates objects
    ↓
creates dependencies
    ↓
controls lifecycle
```

With IoC:

```text
CDI Container
      ↓
creates objects
      ↓
resolves dependencies
      ↓
manages lifecycle
      ↓
injects dependencies
```

Therefore:

> **IoC means control of object creation/lifecycle is moved away from your application code to a container/framework.**

---

# 6. What Is CDI?

CDI stands for:

> **Contexts and Dependency Injection**

It is the Jakarta EE dependency-management and contextual lifecycle model.

CDI provides:

```text
Dependency Injection
+
Lifecycle Management
+
Scopes
+
Qualifiers
+
Events
+
Interceptors
+
Decorators
+
Producer methods
```

---

# 7. CDI Container

Think of the CDI container as an object manager.

```text
                 CDI Container
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
 EmployeeService  EmployeeDAO   NotificationService
       │
       ▼
 EmployeeDAO
```

The container:

```text
creates beans
resolves dependencies
manages scopes
manages lifecycle
applies interceptors
fires events
```

---

# 8. CDI Bean

A CDI bean is an object managed by the CDI container according to CDI's rules.

Example:

```java
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class EmployeeService {

    public void createEmployee() {
        // business logic
    }
}
```

Now CDI can manage this class.

---

# 9. `@Inject`

Basic dependency injection:

```java
import jakarta.inject.Inject;

public class EmployeeResource {

    @Inject
    private EmployeeService employeeService;
}
```

Conceptually:

```text
CDI Container
     ↓
EmployeeResource
     ↓
@Inject
     ↓
EmployeeService
```

---

# 10. Constructor Injection

Constructor injection makes dependencies explicit.

```java
@ApplicationScoped
public class EmployeeService {

    private final EmployeeRepository repository;

    @Inject
    public EmployeeService(
            EmployeeRepository repository) {

        this.repository = repository;
    }
}
```

Mental model:

```text
EmployeeService
       ↑
       │
EmployeeRepository
```

The dependency is required to construct the object.

---

# 11. Why Constructor Injection Is Valuable

It provides:

* explicit dependencies
* easier testing
* immutable references
* fewer partially initialized objects
* easier reasoning about class design

Example test conceptually:

```java
EmployeeRepository mockRepository =
        new MockEmployeeRepository();

EmployeeService service =
        new EmployeeService(mockRepository);
```

This is much cleaner than manually manipulating injected fields.

---

# 12. Field Injection

Example:

```java
@Inject
private EmployeeService employeeService;
```

Advantages:

```text
simple
short
convenient
```

Disadvantages:

```text
dependencies are less visible
harder to instantiate manually
fields cannot be final
```

For maintainability, prefer constructor injection where practical.

---

# 13. Setter Injection

Conceptually:

```java
private EmployeeService service;

@Inject
public void setService(
        EmployeeService service) {

    this.service = service;
}
```

Useful when a dependency is optional or when setter-based configuration is appropriate.

But required dependencies are usually better represented through construction.

---

# 14. CDI Scopes

One of CDI's most important concepts.

Common scopes:

```text
@ApplicationScoped
@RequestScoped
@SessionScoped
@ConversationScoped
@Dependent
```

They define how long a contextual bean instance lives.

---

# 15. `@ApplicationScoped`

```java
@ApplicationScoped
public class EmployeeService {
}
```

Conceptually:

```text
Application starts
       ↓
ApplicationScoped bean available
       ↓
Many requests use contextual instance
       ↓
Application stops
       ↓
Bean destroyed
```

Use it for appropriate application-wide services that don't hold per-user/request mutable state.

---

# 16. `@RequestScoped`

```java
@RequestScoped
public class EmployeeRequestContext {
}
```

Conceptually:

```text
Request 1
 ↓
Instance A

Request 2
 ↓
Instance B

Request 3
 ↓
Instance C
```

Useful for request-specific state.

---

# 17. `@SessionScoped`

A session-scoped bean generally lives across requests associated with a user's HTTP session.

```java
@SessionScoped
public class UserSession
        implements Serializable {

    private String username;
}
```

Use carefully.

Don't put large objects or unnecessary mutable state into HTTP sessions.

---

# 18. `@Dependent`

`@Dependent` is CDI's default pseudo-scope when no other scope is specified.

Conceptually, the dependent object's lifecycle is tied to the bean into which it is injected/used under CDI lifecycle rules.

Example:

```java
public class EmployeeMapper {
}
```

If discovered as a CDI bean without another scope, it has dependent scope.

---

# 19. Scope Comparison

| Scope                 | Typical lifetime                 |
| --------------------- | -------------------------------- |
| `@ApplicationScoped`  | Application                      |
| `@RequestScoped`      | Request                          |
| `@SessionScoped`      | HTTP session                     |
| `@ConversationScoped` | Conversation                     |
| `@Dependent`          | Dependent bean/context lifecycle |

The key question isn't:

> "Which scope is easiest?"

Ask:

> **"What lifecycle should this state have?"**

---

# 20. The Dangerous Mistake

Don't do:

```java
@ApplicationScoped
public class EmployeeService {

    private Employee currentEmployee;

}
```

if `currentEmployee` represents request/user-specific state.

Multiple requests could interact with shared mutable state.

Instead:

```text
ApplicationScoped
→ shared stateless service behavior

RequestScoped
→ request-specific state

SessionScoped
→ user-session state
```

---

# 21. CDI and Thread Safety

Suppose:

```text
100 requests
      ↓
ApplicationScoped EmployeeService
```

The same contextual service instance may handle concurrent requests.

Therefore:

```text
Avoid mutable shared fields.
```

Prefer:

```java
@ApplicationScoped
public class EmployeeService {

    public Employee find(Long id) {
        // local variables
        // stateless operation
    }
}
```

rather than:

```java
private Employee currentEmployee;
```

---

# 22. Dependency Graph

Consider:

```text
EmployeeResource
       ↓
EmployeeService
       ↓
EmployeeRepository
       ↓
EntityManager
```

CDI builds/resolves the object graph.

Conceptually:

```text
CDI Container
     ↓
Create EmployeeResource
     ↓
Needs EmployeeService
     ↓
Create/obtain EmployeeService
     ↓
Needs EmployeeRepository
     ↓
Create/obtain Repository
     ↓
Inject dependencies
```

---

# 23. Interface-Based Design

Prefer:

```java
public interface EmployeeRepository {

    Employee findById(Long id);

}
```

Implementation:

```java
@ApplicationScoped
public class JpaEmployeeRepository
        implements EmployeeRepository {

    @Override
    public Employee findById(Long id) {
        ...
    }
}
```

Then:

```java
@Inject
EmployeeRepository repository;
```

But now CDI must know **which implementation** to inject if multiple beans implement the interface.

That leads to qualifiers.

---

# 24. Qualifiers

Suppose:

```text
PaymentService
      ↑
      │
 ┌────┴─────┐
 │          │
Stripe     PayPal
```

CDI needs to know which implementation you want.

Define:

```java
import jakarta.inject.Qualifier;

import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.*;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

@Qualifier
@Retention(RUNTIME)
@Target({
    TYPE,
    FIELD,
    PARAMETER,
    METHOD
})
public @interface Stripe {
}
```

---

# 25. Applying a Qualifier

```java
@Stripe
@ApplicationScoped
public class StripePaymentService
        implements PaymentService {

    @Override
    public void pay() {
        ...
    }
}
```

Another implementation:

```java
@PayPal
@ApplicationScoped
public class PayPalPaymentService
        implements PaymentService {

    @Override
    public void pay() {
        ...
    }
}
```

Then inject:

```java
@Inject
@Stripe
private PaymentService paymentService;
```

Now CDI knows exactly which bean you want.

---

# 26. Why Qualifiers Matter

They enable:

```text
Interface
   ↓
Multiple implementations
   ↓
Qualifier
   ↓
Specific implementation
```

This is extremely useful for:

```text
payment providers
notification providers
storage providers
cloud providers
database strategies
business strategies
```

---

# 27. `@Named`

You can expose a CDI bean using a name:

```java
@Named("employeeService")
@ApplicationScoped
public class EmployeeService {
}
```

This is particularly useful when integrating CDI beans with technologies that refer to beans by name, such as some UI/view technologies.

Don't use `@Named` merely because it "looks like dependency injection."

---

# 28. CDI Producer

Sometimes you need to create an object using custom construction logic.

Example:

```java
@Produces
public Date createDate() {
    return new Date();
}
```

Then:

```java
@Inject
private Date date;
```

The producer tells CDI:

> "Use this method to produce this dependency."

---

# 29. Producer with Qualifier

Example:

```java
@Produces
@ApplicationDatabase
public DataSource dataSource() {

    return createDataSource();
}
```

Then:

```java
@Inject
@ApplicationDatabase
private DataSource dataSource;
```

This is useful for objects whose construction isn't simply:

```java
new SomeClass()
```

---

# 30. Producer Fields

CDI can also use producer fields.

Conceptually:

```java
@Produces
@ApplicationDatabase
private DataSource dataSource;
```

Producer methods are often more flexible because they allow construction/configuration logic.

---

# 31. Disposer Methods

Sometimes CDI-managed resources require cleanup.

Conceptually:

```java
public void dispose(
        @Disposes
        @ApplicationDatabase
        DataSource dataSource) {

    // cleanup
}
```

The disposer corresponds to a producer and is used for cleanup when the produced resource's lifecycle ends according to CDI rules.

---

# 32. CDI Lifecycle

A simplified lifecycle:

```text
Bean discovery
     ↓
Bean resolution
     ↓
Bean creation
     ↓
Dependency injection
     ↓
@PostConstruct
     ↓
Bean used
     ↓
@PreDestroy
     ↓
Bean destroyed
```

---

# 33. `@PostConstruct`

Use:

```java
@PostConstruct
public void initialize() {
    ...
}
```

Example:

```java
@ApplicationScoped
public class ConfigurationService {

    @PostConstruct
    public void initialize() {
        System.out.println(
            "Configuration loaded"
        );
    }
}
```

Use it for appropriate initialization after dependency injection.

---

# 34. `@PreDestroy`

```java
@PreDestroy
public void shutdown() {
    ...
}
```

Use it for lifecycle cleanup when appropriate.

Avoid treating it as a universal shutdown mechanism for every external resource; resources such as containers, pools, and managed infrastructure should generally be managed according to their own lifecycle contracts.

---

# 35. Initialization Order

Don't assume arbitrary CDI beans initialize in a particular order simply because one exists in the application.

If bean B genuinely depends on bean A's initialization, model that dependency explicitly rather than relying on incidental startup order.

---

# 36. Alternatives

Suppose:

```text
NotificationService
       │
 ┌─────┴──────┐
 │            │
Email        SMS
```

You can define alternative implementations.

Conceptually:

```java
@Alternative
@ApplicationScoped
public class SmsNotificationService
        implements NotificationService {
}
```

Then enable the alternative according to your CDI configuration/deployment model.

---

# 37. `@Priority`

CDI also provides priority mechanisms for certain alternatives/interceptors/decorators.

Conceptually:

```java
@Alternative
@Priority(100)
public class SmsNotificationService
        implements NotificationService {
}
```

The exact use depends on what CDI element you're prioritizing.

---

# 38. CDI Events

CDI provides an event mechanism for loosely coupled communication inside an application.

Suppose:

```text
Employee created
```

Instead of:

```text
EmployeeService
     ↓
EmailService
     ↓
AuditService
     ↓
NotificationService
```

you can publish an event:

```text
EmployeeService
     ↓
EmployeeCreatedEvent
     ↓
CDI Event Bus
   ┌────┼────┐
   ↓    ↓    ↓
Audit Email Notification
```

---

# 39. Firing an Event

```java
@Inject
Event<EmployeeCreatedEvent> employeeCreated;

public void create(Employee employee) {

    repository.save(employee);

    employeeCreated.fire(
        new EmployeeCreatedEvent(
            employee.getId()
        )
    );
}
```

---

# 40. Observing an Event

```java
public void onEmployeeCreated(
        @Observes EmployeeCreatedEvent event) {

    auditService.record(event);
}
```

Now the producer doesn't need to know which components consume the event.

This reduces coupling.

---

# 41. CDI Events and Transactions

Events can participate in transaction-aware processing.

You should understand the distinction between:

```text
immediate observer
transactional observer
```

Transactional observers allow reactions to be tied to transaction phases, such as after successful transaction completion.

This becomes extremely useful for enterprise workflows.

---

# 42. Example

Suppose:

```text
Create Employee
      ↓
Database transaction
      ↓
COMMIT
```

You don't necessarily want:

```text
Send "employee created"
```

before the transaction successfully commits.

Otherwise:

```text
Database rollback
       +
Email already sent
```

could produce inconsistent external behavior.

Transaction-aware event handling can help design around this problem.

---

# 43. CDI Interceptors

Interceptors allow cross-cutting behavior.

Examples:

```text
logging
metrics
auditing
security
performance measurement
transaction-related concerns
```

Conceptually:

```text
Method
  ↓
Interceptor
  ↓
Method
```

---

# 44. Interceptor Binding

Define an interceptor binding annotation.

Conceptually:

```java
@InterceptorBinding
@Retention(RUNTIME)
@Target({
    TYPE,
    METHOD
})
public @interface Audited {
}
```

Then:

```java
@Audited
public void createEmployee() {
    ...
}
```

An interceptor can react to this annotation.

---

# 45. Interceptor Example

Conceptually:

```java
@Audited
@Interceptor
public class AuditInterceptor {

    @AroundInvoke
    public Object audit(
            InvocationContext context)
            throws Exception {

        System.out.println(
            "Calling: "
            + context.getMethod().getName()
        );

        return context.proceed();
    }
}
```

Flow:

```text
createEmployee()
       ↓
AuditInterceptor
       ↓
context.proceed()
       ↓
createEmployee()
```

---

# 46. Why Interceptors Are Powerful

Without interceptors:

```text
Service A
 → logging code

Service B
 → logging code

Service C
 → logging code
```

With interceptor:

```text
              LoggingInterceptor
                     │
        ┌────────────┼────────────┐
        ↓            ↓            ↓
     Service A    Service B    Service C
```

Cross-cutting concerns become centralized.

---

# 47. Decorators

CDI decorators are used to add behavior to a bean while maintaining the bean's business interface.

Example:

```text
PaymentService
      ↓
PaymentDecorator
      ↓
Real PaymentService
```

A decorator can add:

```text
metrics
notifications
business auditing
additional behavior
```

while still representing the same business abstraction.

---

# 48. Interceptor vs Decorator

Important interview question.

### Interceptor

Generally used for:

```text
cross-cutting technical concerns
```

Examples:

```text
logging
metrics
security
auditing
```

### Decorator

Generally used to enhance:

```text
business behavior
```

through the bean's business interface.

Mental model:

```text
Technical concern → Interceptor

Business behavior extension → Decorator
```

---

# 49. CDI Proxy

CDI scopes often involve contextual references/proxies.

For example:

```text
@RequestScoped Bean
```

may be represented to an injecting bean through a proxy so that the correct contextual instance can be resolved for the current request.

This enables code such as:

```java
@Inject
private RequestContext requestContext;
```

inside a longer-lived bean, subject to CDI's contextual rules.

---

# 50. Scope Compatibility

Consider:

```text
@ApplicationScoped
      ↓
@RequestScoped
```

This is possible because CDI can use contextual references/proxies.

But:

```text
ApplicationScoped
```

must not simply store request-specific state in an ordinary field.

The proxy/context mechanism does **not** make arbitrary mutable state thread-safe.

---

# 51. Circular Dependencies

Consider:

```text
A → B
B → A
```

This creates a circular dependency.

Example:

```text
EmployeeService
      ↓
AuditService
      ↓
EmployeeService
```

Avoid circular dependencies through better architecture.

Possible strategies:

```text
extract shared abstraction
introduce event
split responsibilities
move logic to another service
```

Don't blindly use lazy tricks to hide poor architecture.

---

# 52. CDI and Layered Architecture

Your Employee application becomes:

```text
             REST Client
                  │
                  ▼
          EmployeeResource
                  │
               @Inject
                  ▼
          EmployeeService
                  │
               @Inject
                  ▼
        EmployeeRepository
                  │
               @Inject
                  ▼
            EntityManager
```

CDI handles object wiring.

---

# 53. Production Package Structure

```text
com.example.employee
│
├── config
│   └── ApplicationConfig.java
│
├── resource
│   └── EmployeeResource.java
│
├── service
│   └── EmployeeService.java
│
├── repository
│   └── EmployeeRepository.java
│
├── entity
│   └── Employee.java
│
├── dto
│   ├── EmployeeRequest.java
│   └── EmployeeResponse.java
│
├── mapper
│   └── EmployeeMapper.java
│
├── exception
│   └── EmployeeNotFoundException.java
│
├── event
│   └── EmployeeCreatedEvent.java
│
├── interceptor
│   └── AuditInterceptor.java
│
└── qualifier
    └── PrimaryRepository.java
```

---

# 54. Production Example

## Repository

```java
@ApplicationScoped
public class EmployeeRepository {

    public Employee findById(Long id) {
        // persistence logic
        return null;
    }

    public void save(Employee employee) {
        // persistence logic
    }
}
```

## Service

```java
@ApplicationScoped
public class EmployeeService {

    private final EmployeeRepository repository;

    @Inject
    public EmployeeService(
            EmployeeRepository repository) {

        this.repository = repository;
    }

    public Employee findById(Long id) {
        return repository.findById(id);
    }
}
```

## Resource

```java
@Path("/employees")
@RequestScoped
public class EmployeeResource {

    private final EmployeeService service;

    @Inject
    public EmployeeResource(
            EmployeeService service) {

        this.service = service;
    }

    @GET
    @Path("/{id}")
    @Produces(MediaType.APPLICATION_JSON)
    public Response get(
            @PathParam("id") Long id) {

        Employee employee =
                service.findById(id);

        return Response.ok(employee).build();
    }
}
```

Notice:

```text
Resource
  ↓
does HTTP work

Service
  ↓
does business work

Repository
  ↓
does persistence work

CDI
  ↓
wires them together
```

---

# 55. CDI Does NOT Mean Everything Should Be `@ApplicationScoped`

This is a common beginner mistake.

Don't do:

```java
@ApplicationScoped
public class EverythingService {
    
    // hundreds of unrelated operations
}
```

Instead design around responsibilities:

```text
EmployeeService
DepartmentService
PayrollService
NotificationService
AuditService
```

Good dependency injection does not compensate for poor boundaries.

---

# 56. CDI vs Manual `new`

Manual:

```java
EmployeeService service =
        new EmployeeService(
            new EmployeeRepository()
        );
```

CDI:

```java
@Inject
EmployeeService service;
```

Container:

```text
create
 ↓
resolve
 ↓
inject
 ↓
manage lifecycle
 ↓
apply interceptors
 ↓
manage context
```

---

# 57. CDI vs Spring Dependency Injection

Since you're targeting Java backend roles, know this distinction.

| CDI                                      | Spring                                                         |
| ---------------------------------------- | -------------------------------------------------------------- |
| Jakarta EE standard                      | Spring ecosystem                                               |
| `jakarta.inject.Inject` / CDI APIs       | `org.springframework.beans.factory.annotation.Autowired`, etc. |
| CDI scopes                               | Spring scopes                                                  |
| CDI interceptors/decorators/events       | Spring AOP/events/decorators through different mechanisms      |
| Container-managed enterprise environment | Spring ApplicationContext                                      |
| Jakarta EE standard                      | Spring-specific ecosystem                                      |

The concepts overlap heavily:

```text
DI
IoC
Scopes
Qualifiers
Lifecycle
Interceptors
Events
```

But the APIs and container semantics differ.

---

# 58. CDI vs EJB

CDI and EJB are not identical.

CDI provides:

```text
dependency injection
contexts
events
interceptors
decorators
```

EJB provides enterprise component capabilities such as:

```text
container-managed transactions
security integration
concurrency-related container services
remote/local enterprise interfaces
scheduled enterprise tasks
```

Modern Jakarta EE applications often use CDI heavily alongside other Jakarta EE specifications rather than treating EJB as the only enterprise component model.

---

# 59. CDI + REST

Your architecture:

```text
Jakarta REST
      ↓
CDI
      ↓
Service
      ↓
JPA
      ↓
Database
```

This is one of the most important Jakarta EE combinations to understand.

---

# 60. CDI + JPA

A repository can receive an `EntityManager` through container-managed mechanisms depending on your persistence setup.

Conceptually:

```java
@ApplicationScoped
public class EmployeeRepository {

    @PersistenceContext
    private EntityManager entityManager;
}
```

Then:

```text
CDI-managed Repository
        ↓
Persistence Context
        ↓
EntityManager
        ↓
Database
```

Transaction management must be configured correctly around the operation.

---

# 61. CDI + Transactions

Suppose:

```java
public void createEmployee() {

    repository.save(employee);

    auditService.record(...);
}
```

You need to decide:

```text
Where is the transaction boundary?
```

Generally:

```text
REST Resource
      ↓
Service method
      ↓
Transaction
      ├── repository operation
      └── related database operations
```

Don't scatter transaction boundaries arbitrarily throughout your architecture.

---

# 62. CDI + Events + Transactions

A strong enterprise pattern:

```text
REST
 ↓
Service
 ↓
Transaction
 ↓
Database
 ↓
Transaction-aware event
 ↓
Audit / notification
```

This helps separate the core business operation from secondary reactions.

For reliable cross-system delivery, however, CDI events alone are not a substitute for durable messaging/outbox architecture.

---

# 63. CDI Testing

Because dependencies are injected, you can design testable services.

Interface:

```java
public interface EmployeeRepository {
    Employee findById(Long id);
}
```

Service:

```java
public class EmployeeService {

    private final EmployeeRepository repository;

    public EmployeeService(
            EmployeeRepository repository) {

        this.repository = repository;
    }
}
```

Unit test concept:

```java
EmployeeRepository repository =
        mock(EmployeeRepository.class);

EmployeeService service =
        new EmployeeService(repository);
```

This is one reason constructor injection is so valuable.

---

# 64. Unit Test vs CDI Integration Test

### Unit test

Test:

```text
EmployeeService
```

without starting the CDI container.

### Integration test

Test:

```text
CDI container
 +
injection
 +
JPA
 +
transactions
 +
REST
```

Both are necessary.

---

# 65. CDI Dependency Resolution

When CDI sees:

```java
@Inject
PaymentService paymentService;
```

it needs to answer:

```text
Which bean?
Which qualifier?
Which scope?
Which implementation?
```

Conceptually:

```text
Injection Point
      ↓
Type
      ↓
Qualifiers
      ↓
Resolvable beans
      ↓
Bean selected
```

If no suitable bean exists:

```text
Unsatisfied dependency
```

If multiple equally suitable beans exist:

```text
Ambiguous dependency
```

These are important CDI deployment/startup errors.

---

# 66. Unsatisfied Dependency

Example:

```java
@Inject
PaymentService paymentService;
```

but CDI cannot find an appropriate bean.

Result:

```text
Unsatisfied dependency
```

Fix:

```text
discover bean
correct scope
correct qualifier
correct bean archive/configuration
```

---

# 67. Ambiguous Dependency

Suppose:

```text
StripePaymentService
PayPalPaymentService
```

both implement:

```text
PaymentService
```

and you inject:

```java
@Inject
PaymentService service;
```

CDI may have multiple candidates.

Solution:

```text
Qualifier
```

Example:

```java
@Inject
@Stripe
PaymentService service;
```

---

# 68. CDI Bean Discovery

For CDI to manage your classes, they need to be discoverable according to the CDI/Jakarta EE deployment model and bean archive configuration.

Historically, applications used:

```text
WEB-INF/beans.xml
```

or:

```text
META-INF/beans.xml
```

depending on packaging.

Modern CDI versions support implicit bean archives under defined rules, so don't assume every deployment requires the same `beans.xml` configuration.

---

# 69. CDI Lifecycle Mental Model

Memorize:

```text
Discovery
   ↓
Bean metadata
   ↓
Dependency resolution
   ↓
Context
   ↓
Creation
   ↓
Injection
   ↓
@PostConstruct
   ↓
Business method
   ↓
Interceptors
   ↓
Events
   ↓
@PreDestroy
```

---

# 70. Day 18 Architecture Exercise

Design this:

## Payment System

Requirements:

```text
PaymentService
      ↓
PaymentProvider
```

Providers:

```text
Stripe
PayPal
BankTransfer
```

Use:

```text
CDI
@Inject
Qualifier
@ApplicationScoped
Producer
Interceptor
Event
```

Architecture:

```text
PaymentResource
      ↓
PaymentService
      ↓
PaymentProvider
      │
 ┌────┼──────────┐
 ↓    ↓          ↓
Stripe PayPal BankTransfer
```

Then publish:

```text
PaymentCompletedEvent
```

and let:

```text
AuditService
NotificationService
AnalyticsService
```

observe it.

---

# 71. Day 18 Coding Exercises

### Beginner

1. Create a CDI bean.
2. Inject a service.
3. Use constructor injection.
4. Create a request-scoped bean.
5. Create an application-scoped bean.
6. Create a session-scoped bean.
7. Add `@PostConstruct`.
8. Add `@PreDestroy`.

### Intermediate

9. Create an interface with two implementations.
10. Create a qualifier.
11. Inject a qualified implementation.
12. Create a producer method.
13. Create a disposer method.
14. Create an alternative.
15. Fire a CDI event.
16. Observe a CDI event.

### Advanced

17. Create an interceptor.
18. Create an interceptor binding.
19. Create a decorator.
20. Build a complete REST → CDI → Service → Repository application.

---

# 72. Interview Questions

### Fundamentals

**1. What is CDI?**

Contexts and Dependency Injection, the Jakarta EE programming model for dependency injection, contextual lifecycle management, events, interceptors, decorators, and related services.

**2. What is dependency injection?**

Providing a class's dependencies from outside rather than having the class construct them directly.

**3. What is IoC?**

Moving object creation/lifecycle control from application code to a container/framework.

**4. What is a CDI bean?**

An object managed according to CDI's bean and contextual lifecycle rules.

**5. What does `@Inject` do?**

It marks an injection point where CDI should resolve and provide a dependency.

---

# 73. Scope Questions

**6. What is `@ApplicationScoped`?**

A CDI scope representing application-wide contextual lifetime.

**7. What is `@RequestScoped`?**

A scope associated with an individual request.

**8. What is `@SessionScoped`?**

A scope associated with an HTTP session.

**9. What is `@Dependent`?**

CDI's default pseudo-scope whose lifecycle is dependent on its injection/contextual owner.

---

# 74. Qualifier Questions

**10. Why do we need qualifiers?**

To distinguish between multiple beans that satisfy the same type.

**11. What happens if CDI finds multiple beans?**

If multiple beans are equally eligible and no qualifier/other resolution mechanism distinguishes them, dependency resolution becomes ambiguous.

**12. What happens if CDI finds none?**

The injection point is unsatisfied.

---

# 75. Advanced Questions

**13. Producer vs normal bean?**

A normal bean is discovered and constructed by CDI according to its bean definition. A producer allows application-defined logic to supply an object for injection.

**14. What is a disposer?**

A method associated with a producer for cleanup of a produced object when its lifecycle ends.

**15. Interceptor vs decorator?**

```text
Interceptor
→ technical cross-cutting behavior

Decorator
→ business behavior enhancement
```

**16. Why are CDI proxies important?**

They allow CDI to provide contextual references whose actual instance can depend on the active scope/context.

---

# 76. Architect-Level Questions

### Q1. Why should services generally be stateless?

Because stateless services are easier to:

```text
scale
test
reason about
reuse
run concurrently
```

and don't require synchronization around per-request mutable state.

---

### Q2. How does CDI reduce coupling?

Instead of:

```text
Service
 ↓
new Repository()
```

you have:

```text
Service
 ↓
Repository interface
 ↑
CDI supplies implementation
```

Therefore implementation choices can be changed with less impact.

---

### Q3. How would you implement multiple payment providers?

```text
PaymentProvider
      ↑
 ┌────┼────┐
 │    │    │
Stripe PayPal Bank
```

Use:

```text
qualifiers
```

to select implementations.

---

### Q4. How would you implement auditing?

A cross-cutting technical audit concern can use:

```text
Interceptor
```

For domain-level events:

```text
CDI Event
```

For durable cross-service audit delivery:

```text
Messaging / Outbox
```

may be more appropriate.

---

# 77. Production CDI Rules

### Rule 1

Prefer constructor injection for required dependencies.

### Rule 2

Keep application-scoped beans stateless where practical.

### Rule 3

Choose scopes based on lifecycle requirements.

### Rule 4

Use qualifiers for multiple implementations.

### Rule 5

Don't use CDI to hide bad architecture.

### Rule 6

Don't create circular dependencies.

### Rule 7

Keep business logic in services/domain components.

### Rule 8

Use interceptors for cross-cutting technical concerns.

### Rule 9

Use events to reduce coupling between components.

### Rule 10

Use durable messaging when events must survive crashes and cross process boundaries.

---

# 78. Day 18 Mastery Challenge

Build:

```text
Employee Management Platform
```

with:

```text
EmployeeResource
       ↓
EmployeeService
       ↓
EmployeeRepository
       ↓
JPA
```

Use CDI for:

```text
@Inject
@ApplicationScoped
@RequestScoped
Qualifier
Producer
@PostConstruct
@PreDestroy
Event
Interceptor
```

Implement:

```text
EmployeeCreatedEvent
EmployeeUpdatedEvent
EmployeeDeletedEvent
```

Observers:

```text
AuditObserver
NotificationObserver
AnalyticsObserver
```

Architecture:

```text
                         REST API
                            │
                            ▼
                   EmployeeResource
                            │
                          CDI
                            │
                            ▼
                    EmployeeService
                            │
                   ┌────────┴────────┐
                   │                 │
                   ▼                 ▼
           EmployeeRepository    AuditService
                   │                 ▲
                   ▼                 │
                  JPA          CDI Events
                   │
                   ▼
               Database
```

---

# 79. Day 18 Final Cheat Sheet

```text
CDI
→ Contexts and Dependency Injection

IoC
→ Container controls object lifecycle

@Inject
→ Dependency injection

@ApplicationScoped
→ Application lifecycle

@RequestScoped
→ Request lifecycle

@SessionScoped
→ HTTP session lifecycle

@Dependent
→ Dependent lifecycle

@Qualifier
→ Select implementation

@Named
→ Give bean a name

@Produces
→ Produce injectable object

@Disposes
→ Cleanup produced object

@PostConstruct
→ Initialization callback

@PreDestroy
→ Destruction callback

@Alternative
→ Alternative bean implementation

@Priority
→ Priority mechanism for supported CDI elements

Event<T>
→ Publish CDI event

@Observes
→ Consume CDI event

@Interceptor
→ Cross-cutting behavior

@AroundInvoke
→ Interceptor method

@Decorator
→ Business behavior enhancement
```

---

# 80. The Big Picture

You've now built an important part of the Jakarta EE stack:

```text
Day 1
HTTP
 ↓
Day 2–5
Servlets / Filters / Sessions
 ↓
Day 6–8
JSP / JSTL / MVC
 ↓
Day 9
JDBC
 ↓
Day 10–12
JPA
 ↓
Day 13
Transactions
 ↓
Day 14
Enterprise Integration Project
 ↓
Day 15–16
REST / Jakarta REST
 ↓
Day 17
REST Security
 ↓
Day 18
CDI
```

Your architecture is becoming:

```text
                    CLIENT
                       │
                     HTTPS
                       │
                       ▼
                 Jakarta REST
                       │
                    CDI DI
                       │
                       ▼
                  Controller
                       │
                       ▼
                    Service
                       │
                Transaction
                       │
                       ▼
                 Repository
                       │
                    JPA
                       │
                       ▼
                   Database
```

### Day 18 golden rule

> **CDI should make dependencies explicit, lifecycles intentional, components loosely coupled, and cross-cutting behavior reusable.**

The key architectural distinction to remember is:

```text
CDI
 ↓
"Wires and manages components"

Service
 ↓
"Implements business behavior"

JPA
 ↓
"Manages persistence"

Jakarta REST
 ↓
"Exposes HTTP API"

Transaction
 ↓
"Controls atomic business operations"
```

**Day 19:** EJB Fundamentals — session beans, stateless/stateful/singleton beans, EJB lifecycle, business interfaces, container-managed transactions, security, timers, asynchronous methods, pooling, concurrency, and when EJB is still useful in modern Jakarta EE.
