# Day 23 — JTA & Distributed Transactions

**J2EE/Jakarta EE 30-Day Mastery**
**Date:** October 25, 2026
**Focus:** Transactions, JTA, JTS concepts, XA, two-phase commit, distributed transactions, transaction propagation, synchronization, failure handling, and Saga-based alternatives.

Today connects **JPA/JDBC + EJB + JMS + distributed systems** into one enterprise transaction model.

---

# 1. What Is a Transaction?

A transaction is a logical unit of work that should move the system from one consistent state to another.

Example:

```text
Place Order
   │
   ├── Create Order
   ├── Create Order Items
   ├── Reduce Inventory
   └── Record Payment
```

Ideally:

```text
ALL SUCCESS → COMMIT
ANY FAILURE → ROLLBACK
```

---

# 2. ACID

Every enterprise developer must understand ACID.

## Atomicity

Either everything succeeds or everything rolls back.

```text
A
B
C
```

If C fails:

```text
A → rollback
B → rollback
C → rollback
```

---

## Consistency

The transaction must preserve database/application invariants.

Example:

```text
account.balance >= 0
```

If an operation violates the invariant, the transaction should not commit.

---

## Isolation

Concurrent transactions should not interfere incorrectly.

For example:

```text
Transaction A
Transaction B
```

should behave according to the configured isolation guarantees.

---

## Durability

After commit:

```text
COMMIT
  ↓
Data survives failure
```

---

# 3. Local vs Distributed Transaction

This distinction is critical.

### Local transaction

One transactional resource:

```text
Application
    ↓
Database
```

For example:

```text
JPA → PostgreSQL
```

or:

```text
JDBC → MySQL
```

### Distributed transaction

Multiple transactional resources participate:

```text
Application
   ├──→ Database
   └──→ JMS Broker
```

or:

```text
Application
   ├──→ Database A
   └──→ Database B
```

Now coordination becomes much harder.

---

# 4. What Is JTA?

**JTA = Jakarta Transactions**

It provides a standard API for managing transactions across enterprise components and resources.

The key idea:

```text
Application
    ↓
Transaction Manager
    ↓
┌───────────────┬───────────────┐
│ Database      │ Messaging     │
│ Resource      │ Resource      │
└───────────────┴───────────────┘
```

The transaction manager coordinates participating resources.

---

# 5. JTA Architecture

Think:

```text
                  Application
                       │
                       ▼
               Transaction Manager
                 /            \
                /              \
               ▼                ▼
          DB Resource       JMS Resource
```

The application doesn't necessarily coordinate every resource itself.

The transaction infrastructure does.

---

# 6. Important JTA Components

Know these concepts:

* `jakarta.transaction.UserTransaction`
* `jakarta.transaction.TransactionManager`
* `jakarta.transaction.Transaction`
* `jakarta.transaction.Synchronization`
* `jakarta.transaction.Status`
* `jakarta.transaction.RollbackException`
* `jakarta.transaction.HeuristicMixedException`

And the annotation:

```java
jakarta.transaction.Transactional
```

Don't confuse this with Spring's:

```java
org.springframework.transaction.annotation.Transactional
```

They belong to different frameworks/specifications.

---

# 7. `UserTransaction`

Programmatic transaction management can use `UserTransaction`.

Conceptually:

```java
@Resource
UserTransaction transaction;

public void createOrder() {

    try {
        transaction.begin();

        // database operations

        transaction.commit();

    } catch (Exception e) {

        try {
            transaction.rollback();
        } catch (Exception rollbackException) {
            // handle rollback failure
        }
    }
}
```

The application explicitly controls:

```text
begin()
commit()
rollback()
```

This is **programmatic transaction management**.

---

# 8. Declarative Transaction Management

Enterprise applications commonly prefer declarative transaction boundaries where appropriate.

Conceptually:

```java
@Transactional
public void createOrder() {

    createOrderRecord();
    createOrderItems();
    reserveInventory();
}
```

The infrastructure manages the transaction boundary.

The application expresses:

> This operation should execute transactionally.

---

# 9. Container-Managed Transactions

In Jakarta EE, EJBs can use **Container-Managed Transactions (CMT)**.

Example:

```java
@Stateless
public class OrderService {

    public void createOrder(Order order) {

        // transactional work

    }
}
```

The container manages the transaction lifecycle according to the configured transaction attributes.

Conceptually:

```text
Client
  ↓
EJB Container
  ↓
BEGIN
  ↓
Business Method
  ↓
COMMIT / ROLLBACK
```

---

# 10. Bean-Managed Transactions

With **Bean-Managed Transactions (BMT)**, the enterprise bean explicitly manages transaction boundaries.

Conceptually:

```java
@Resource
UserTransaction tx;

public void process() {

    try {
        tx.begin();

        // business work

        tx.commit();

    } catch (Exception e) {

        try {
            tx.rollback();
        } catch (Exception ignored) {
        }
    }
}
```

Now:

```text
Bean
 ↓
UserTransaction
 ↓
Transaction
```

---

# 11. CMT vs BMT

| Feature                   | CMT        | BMT               |
| ------------------------- | ---------- | ----------------- |
| Transaction controlled by | Container  | Bean/application  |
| Explicit begin/commit     | Usually no | Yes               |
| Boilerplate               | Low        | Higher            |
| Fine-grained control      | Lower      | Higher            |
| Typical enterprise choice | Common     | Specialized cases |

General rule:

> Prefer declarative/container-managed transactions unless the business workflow genuinely requires programmatic control.

---

# 12. Transaction Propagation

Suppose:

```text
OrderService
    ↓
PaymentService
    ↓
AuditService
```

What happens to the transaction when methods call each other?

Possible behavior:

```text
Transaction T1
   ↓
OrderService
   ↓
PaymentService
   ↓
same T1
```

or:

```text
Transaction T1
   ↓
OrderService
   ↓
new Transaction T2
```

Transaction propagation defines these semantics.

---

# 13. Common Transaction Attributes

For EJB transaction management, know:

* `REQUIRED`
* `REQUIRES_NEW`
* `MANDATORY`
* `NOT_SUPPORTED`
* `SUPPORTS`
* `NEVER`

---

# 14. REQUIRED

The common default-style behavior.

```text
Existing transaction?
       │
    ┌──┴──┐
   Yes    No
    │      │
 join    create
    │      │
    └──┬───┘
       ▼
      T1
```

Example:

```text
OrderService
     ↓
PaymentService
```

If OrderService already has T1:

```text
OrderService → T1
PaymentService → T1
```

---

# 15. REQUIRES_NEW

Suspends the current transaction and starts a new one.

```text
T1
 ↓
Service A
 ↓
suspend T1
 ↓
T2
 ↓
Service B
 ↓
commit T2
 ↓
resume T1
```

Useful for certain independent operations such as audit logging where you intentionally want a separate transaction.

But don't use it blindly—it changes consistency semantics.

---

# 16. MANDATORY

The method requires an existing transaction.

```text
Existing T?
 ├── Yes → execute
 └── No  → exception
```

Useful when a method must never run outside an existing transaction.

---

# 17. NOT_SUPPORTED

The method executes without a transaction.

If a transaction exists:

```text
T1
 ↓
suspend
 ↓
non-transactional operation
 ↓
resume T1
```

---

# 18. SUPPORTS

```text
Transaction exists?
 ├── Yes → participate
 └── No  → execute without transaction
```

---

# 19. NEVER

The method must not execute inside a transaction.

If one exists:

```text
T1
 ↓
NEVER method
 ↓
exception
```

---

# 20. Transaction Propagation Mental Model

Memorize:

```text
REQUIRED
→ join existing or create new

REQUIRES_NEW
→ suspend existing and create new

MANDATORY
→ existing required

NOT_SUPPORTED
→ execute without transaction

SUPPORTS
→ use transaction if one exists

NEVER
→ transaction must not exist
```

---

# 21. Transaction Isolation

Isolation controls how concurrent transactions interact.

Common levels:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

Some databases provide additional or differently implemented semantics, so always consider the actual database.

---

# 22. Dirty Read

Transaction A changes data:

```text
A:
balance = 500
```

Before A commits:

```text
B reads balance = 500
```

Then A rolls back.

B read data that never became committed.

That's a **dirty read**.

---

# 23. Non-Repeatable Read

Transaction A:

```text
SELECT balance
→ 1000
```

Transaction B changes it:

```text
UPDATE balance = 500
COMMIT
```

Transaction A reads again:

```text
SELECT balance
→ 500
```

Same transaction, different value.

---

# 24. Phantom Read

Transaction A:

```text
SELECT employees WHERE salary > 100000
```

returns:

```text
10 rows
```

Transaction B inserts a matching employee and commits.

Transaction A executes the same query:

```text
11 rows
```

The new row is a phantom.

---

# 25. JPA + JTA

JPA can participate in JTA-managed transactions.

Architecture:

```text
REST / Servlet
      ↓
Service
      ↓
JTA Transaction
      ↓
EntityManager
      ↓
Database
```

Example concept:

```java
@PersistenceContext
private EntityManager entityManager;

public void createEmployee(Employee employee) {

    entityManager.persist(employee);
}
```

If the surrounding transaction is managed by the container, the persistence operation participates in that transaction.

---

# 26. JDBC + JTA

A JDBC connection can participate in a JTA transaction when the datasource/resource is configured for JTA.

Conceptually:

```text
Application
     ↓
JTA Transaction
     ↓
JDBC Resource
     ↓
Database
```

Don't manually call:

```java
connection.commit();
```

inside a JTA-managed transaction unless the environment explicitly supports the behavior you're relying on.

The transaction manager should control the global transaction.

---

# 27. JMS + JTA

Messaging can also participate in transactions.

Example:

```text
Receive JMS Message
       ↓
Update Database
       ↓
Commit
```

With coordinated transactional resources:

```text
JTA Transaction
    ├── JMS
    └── Database
```

The goal is to coordinate both operations.

---

# 28. The Dual-Write Problem

Consider:

```text
BEGIN

INSERT order

SEND OrderCreated

COMMIT
```

Potential failure:

```text
INSERT order → SUCCESS

SEND event → SUCCESS

COMMIT → FAILURE
```

Now the message may exist even though the database transaction didn't commit.

The reverse problem is also possible.

This is one reason distributed transactions become difficult.

---

# 29. Two-Phase Commit

A classic solution for multiple XA resources is **Two-Phase Commit (2PC)**.

Architecture:

```text
             Transaction Coordinator
                  /          \
                 /            \
                ▼              ▼
              DB A           DB B
```

Two phases:

```text
Phase 1 → Prepare
Phase 2 → Commit
```

---

# 30. Phase 1 — Prepare

Coordinator asks:

```text
"Can you commit?"
```

Resource A:

```text
READY
```

Resource B:

```text
READY
```

If everyone says yes:

```text
Proceed to phase 2
```

---

# 31. Phase 2 — Commit

Coordinator says:

```text
COMMIT
```

Resources commit.

```text
DB A → COMMIT
DB B → COMMIT
```

Conceptually:

```text
Coordinator
   │
   ├── prepare → DB A
   ├── prepare → DB B
   │
   ├── commit → DB A
   └── commit → DB B
```

---

# 32. What Is XA?

**XA** is a standard interface/model used for coordinating distributed transactions across transactional resources.

Typical participants:

```text
XA Database
XA JMS Resource
XA Database
```

Conceptually:

```text
JTA Transaction Manager
          ↓
      XA Protocol
      /        \
     ↓          ↓
 Database     JMS
```

---

# 33. XA Transaction Example

Suppose:

```text
Order DB
+
Payment DB
```

One global transaction:

```text
BEGIN GLOBAL TX

Order DB → prepare
Payment DB → prepare

Order DB → commit
Payment DB → commit
```

This can provide atomic coordination across participating XA resources.

---

# 34. Why 2PC Is Expensive

Distributed transactions introduce overhead.

```text
Application
     ↓
Transaction Manager
     ↓
Network
     ↓
Resource 1
     ↓
Network
     ↓
Resource 2
```

Problems include:

* network latency
* coordination overhead
* locking
* resource contention
* failure recovery
* transaction manager complexity
* operational complexity
* reduced availability in some failure scenarios

Therefore:

> Don't use distributed transactions merely because they are technically possible.

---

# 35. Blocking Problem

Imagine:

```text
Coordinator
   ↓
DB A
   ↓
Prepared
```

Then coordinator fails before the final commit decision reaches all participants.

A participant may have to retain resources while waiting for recovery/decision.

This can create:

```text
locks
connections
memory
```

being held for longer periods.

---

# 36. Heuristic Outcomes

Distributed transactions can have unusual failure outcomes.

For example:

```text
DB A → COMMIT
DB B → ROLLBACK
```

This is a **heuristic outcome**—the global transaction no longer has the clean all-or-nothing result expected from normal 2PC coordination.

Enterprise transaction managers provide mechanisms for detecting and recovering from such situations, but application design should minimize the need to encounter them.

---

# 37. Local Transaction vs XA

| Feature                | Local             | XA/Distributed        |
| ---------------------- | ----------------- | --------------------- |
| Resources              | Usually one       | Multiple              |
| Complexity             | Lower             | Higher                |
| Performance overhead   | Lower             | Higher                |
| Coordination           | Database/resource | Transaction manager   |
| Typical use            | Single DB         | Multiple XA resources |
| Operational complexity | Lower             | Higher                |

---

# 38. Why Microservices Avoid Distributed Transactions

Imagine:

```text
Order DB
Payment DB
Inventory DB
Shipping DB
```

Trying to put all of them into one global transaction:

```text
Global Transaction
 ├── Order
 ├── Payment
 ├── Inventory
 └── Shipping
```

creates significant coupling.

Instead, microservices frequently use:

```text
Local Transactions
+
Events
+
Outbox
+
Idempotency
+
Saga
+
Compensation
```

---

# 39. Saga

A Saga breaks a distributed business transaction into a sequence of local transactions.

Example:

```text
T1: Create Order
      ↓
T2: Reserve Inventory
      ↓
T3: Charge Payment
      ↓
T4: Create Shipment
```

If T4 fails:

```text
Compensate T3
      ↓
Refund Payment
```

If T2 fails:

```text
Cancel Order
```

---

# 40. Saga vs 2PC

### 2PC

```text
Global transaction
       ↓
Atomic coordination
       ↓
Commit/Rollback
```

### Saga

```text
Local TX
   ↓
Event
   ↓
Local TX
   ↓
Event
   ↓
Compensation if required
```

The key difference:

**2PC seeks coordinated atomic commit.
Saga achieves business consistency through local transactions and compensating actions.**

---

# 41. Saga Orchestration

One coordinator controls the workflow.

```text
             Order Saga
             Orchestrator
            /     |      \
           ▼      ▼       ▼
       Payment Inventory Shipping
```

Example:

```text
Order
 ↓
Orchestrator
 ↓
Reserve Inventory
 ↓
Charge Payment
 ↓
Create Shipment
```

Failure:

```text
Shipment failed
     ↓
Orchestrator
     ↓
Refund Payment
     ↓
Release Inventory
     ↓
Cancel Order
```

---

# 42. Saga Choreography

No central orchestrator.

Services react to events.

```text
OrderCreated
     ↓
Inventory Service
     ↓
InventoryReserved
     ↓
Payment Service
     ↓
PaymentCompleted
     ↓
Shipping Service
```

If something fails, another event triggers compensation.

---

# 43. Transaction Synchronization

Applications sometimes need callbacks around transaction completion.

Conceptually:

```java
transaction.registerSynchronization(
    new Synchronization() {

        @Override
        public void beforeCompletion() {
            // before commit
        }

        @Override
        public void afterCompletion(int status) {
            // after completion
        }
    }
);
```

This can be useful for integration with transaction-aware infrastructure.

Don't use transaction callbacks as a replacement for durable messaging.

---

# 44. Transaction Rollback

Suppose:

```text
BEGIN
 ↓
Create Order
 ↓
Reserve Inventory
 ↓
Payment Failure
```

Rollback:

```text
Payment Failure
      ↓
ROLLBACK
      ↓
Order creation undone
Inventory reservation undone
```

assuming those operations participate in the same transaction.

---

# 45. Important Rollback Question

Interview question:

> What happens if a transaction fails after some database statements have already executed?

Answer:

The transaction manager/database can roll back the uncommitted changes made within that transaction.

Example:

```text
INSERT A → success
INSERT B → success
UPDATE C → failure

ROLLBACK

A → undone
B → undone
C → unchanged
```

assuming all operations participated in the same transaction.

---

# 46. Transaction Timeout

Long-running transactions are dangerous.

Example:

```text
BEGIN
 ↓
10-minute external API call
 ↓
DB update
 ↓
COMMIT
```

The transaction may hold:

* database connections
* locks
* memory
* resources

for too long.

Prefer:

```text
short transaction
```

and avoid external network calls inside a transaction when possible.

---

# 47. Bad Transaction Design

Avoid:

```java
begin();

callPaymentAPI();     // 5 seconds
callShippingAPI();    // 3 seconds
callFraudAPI();       // 2 seconds

updateDatabase();

commit();
```

Now your database transaction depends on three external networks.

Better architecture:

```text
Order Transaction
     ↓
Commit
     ↓
Event
     ↓
Payment
     ↓
Inventory
     ↓
Shipping
```

This is often better suited to distributed systems.

---

# 48. Transaction Boundary Design

A transaction should normally represent a meaningful business operation.

Good:

```text
createOrder()
```

Potentially problematic:

```text
entire HTTP session
```

or:

```text
entire workflow lasting minutes
```

Think:

```text
Business Operation
       ↓
Transaction Boundary
```

---

# 49. Transaction + Outbox

A powerful architecture is:

```text
BEGIN
   │
   ├── INSERT order
   │
   └── INSERT outbox_event
   │
 COMMIT
   │
   ▼
Outbox Relay
   │
   ▼
Message Broker
```

This keeps the database state and the intent to publish the event in the same local transaction.

---

# 50. Transaction + Idempotency

Consumer:

```text
BEGIN
   │
   ├── Check message ID
   │
   ├── Business update
   │
   └── Record processed ID
   │
 COMMIT
```

Now duplicate delivery can be safely detected.

This is one of the most important patterns in modern enterprise systems.

---

# 51. Complete Order Architecture

Combine Days 21–23:

```text
                         Client
                           │
                           ▼
                     Jakarta REST
                           │
                           ▼
                     Order Service
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
                 Orders        Outbox
                    │             │
                    └──────┬──────┘
                           │
                       COMMIT
                           │
                           ▼
                     Message Broker
                    /      |       \
                   ▼       ▼        ▼
              Payment  Inventory  Notification
                 │         │
                 ▼         ▼
                DB        DB
```

Failure handling:

```text
Consumer
   ↓
Idempotency
   ↓
Local Transaction
   ↓
Success → ACK
   │
   └── Failure → Retry → DLQ
```

Distributed workflow:

```text
Order
 ↓
Payment
 ↓
Inventory
 ↓
Shipping
```

Use Saga/compensation when the business workflow spans independent services.

---

# 52. Jakarta EE Transaction Stack

You should visualize Jakarta EE like this:

```text
                  Application
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       REST         EJB/CDI      JMS
          │           │           │
          └───────────┼───────────┘
                      ▼
              Jakarta Transactions
                      │
             Transaction Manager
                 /           \
                ▼             ▼
              JPA            JDBC
                │             │
                └──────┬──────┘
                       ▼
                    Database
```

---

# 53. Practical Coding Exercise

Create:

```text
OrderService
PaymentService
InventoryService
```

### Requirement

Implement:

```java
createOrder()
```

which:

1. creates order
2. creates order items
3. writes an outbox event

All three database operations should belong to one local transaction.

Conceptually:

```java
@Transactional
public void createOrder(Order order) {

    orderRepository.save(order);

    orderItemRepository.saveItems(order.getItems());

    outboxRepository.save(
        new OrderCreatedEvent(order.getId())
    );
}
```

Then:

```text
Outbox Relay
     ↓
JMS
     ↓
Payment Consumer
```

Add:

* idempotency
* retry
* DLQ
* correlation ID

---

# 54. Advanced Exercise — Distributed Order Saga

Design:

```text
OrderCreated
    ↓
ReserveInventory
    ↓
InventoryReserved
    ↓
ChargePayment
    ↓
PaymentCompleted
    ↓
CreateShipment
```

Failure scenarios:

### Inventory failure

```text
InventoryFailed
      ↓
Cancel Order
```

### Payment failure

```text
PaymentFailed
      ↓
Release Inventory
      ↓
Cancel Order
```

### Shipping failure

```text
ShippingFailed
      ↓
Refund Payment
      ↓
Release Inventory
      ↓
Cancel Order
```

Implement the state machine:

```text
PENDING
  ↓
INVENTORY_RESERVED
  ↓
PAYMENT_COMPLETED
  ↓
SHIPPED
```

Failure states:

```text
INVENTORY_FAILED
PAYMENT_FAILED
SHIPPING_FAILED
CANCELLED
```

---

# 55. Day 23 Interview Questions

## Fundamentals

1. What is a transaction?
2. Explain ACID.
3. What is JTA?
4. What problem does JTA solve?
5. What is a transaction manager?
6. Local vs distributed transaction?
7. What is `UserTransaction`?
8. What is declarative transaction management?
9. CMT vs BMT?
10. What is transaction propagation?

## Propagation

11. Explain REQUIRED.
12. Explain REQUIRES_NEW.
13. Explain MANDATORY.
14. Explain SUPPORTS.
15. Explain NOT_SUPPORTED.
16. Explain NEVER.
17. When would you use REQUIRES_NEW?
18. What happens when an inner transaction fails?

## Isolation

19. What is transaction isolation?
20. Dirty read?
21. Non-repeatable read?
22. Phantom read?
23. Compare isolation levels.
24. Why does SERIALIZABLE have higher concurrency costs?

## Distributed transactions

25. What is XA?
26. What is 2PC?
27. Explain prepare phase.
28. Explain commit phase.
29. What happens if a participant fails during prepare?
30. What happens if the coordinator fails?
31. What is a heuristic outcome?
32. Why are distributed transactions expensive?
33. When should XA be used?

## Architecture

34. Why should external API calls generally not be inside DB transactions?
35. What is the dual-write problem?
36. Explain the Outbox Pattern.
37. Explain the Inbox Pattern.
38. How does idempotency help?
39. What is a Saga?
40. Saga vs 2PC?
41. Orchestration vs choreography?
42. How would you design distributed order processing?
43. How would you handle payment failure?
44. How would you guarantee business consistency without XA?
45. How do you prevent long-running transactions?

---

# 56. Architect-Level Question

Suppose your system has:

```text
Order DB
Payment DB
Inventory DB
Shipping DB
```

You need:

```text
Place Order
```

and the business says:

> "Either everything must succeed or everything must be rolled back."

Don't immediately jump to XA.

First analyze:

```text
1. Are these resources actually under one transaction boundary?
2. Are they independent services?
3. Can compensation work?
4. What consistency does the business really require?
5. How long can the workflow take?
6. What happens during network failures?
7. What is the availability requirement?
8. Can an operation be idempotent?
9. Can an Outbox be used?
10. Can a Saga model the business process?
```

Then compare architectural options:

```text
              Transaction Requirement
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
   Same transactional          Independent
      boundary?                  services?
          │                         │
          ▼                         ▼
    Local transaction          Saga + Events
          │                    + Outbox
          │                    + Idempotency
          │
          └──── Multiple XA resources?
                         │
                         ▼
                   Consider XA/2PC
```

The important architect skill is **choosing the transaction boundary based on business requirements**, rather than automatically applying a distributed transaction.

---

# 57. Day 23 Mastery Checklist

Before moving to Day 24, you should be able to explain:

### Transactions

* [ ] Transaction
* [ ] ACID
* [ ] Commit
* [ ] Rollback
* [ ] Transaction boundary
* [ ] Timeout

### JTA

* [ ] JTA
* [ ] Transaction Manager
* [ ] `UserTransaction`
* [ ] Declarative transactions
* [ ] Programmatic transactions
* [ ] CMT
* [ ] BMT

### Propagation

* [ ] REQUIRED
* [ ] REQUIRES_NEW
* [ ] MANDATORY
* [ ] SUPPORTS
* [ ] NOT_SUPPORTED
* [ ] NEVER

### Isolation

* [ ] Dirty read
* [ ] Non-repeatable read
* [ ] Phantom read
* [ ] Isolation levels

### Distributed Transactions

* [ ] XA
* [ ] 2PC
* [ ] Prepare
* [ ] Commit
* [ ] Coordinator
* [ ] Participants
* [ ] Heuristic outcomes

### Modern distributed architecture

* [ ] Dual-write problem
* [ ] Outbox
* [ ] Inbox
* [ ] Idempotency
* [ ] Saga
* [ ] Compensation
* [ ] Orchestration
* [ ] Choreography
* [ ] Event-driven consistency

---

# 58. Day 23 Golden Mental Model

Memorize this:

```text
LOCAL TRANSACTION

Service
   ↓
Transaction
   ↓
Database
   ↓
COMMIT / ROLLBACK
```

For multiple XA resources:

```text
             JTA
              │
       Transaction Manager
          /           \
         ▼             ▼
       DB A           DB B
         │             │
      PREPARE        PREPARE
          \           /
           ▼         ▼
             COMMIT
```

For microservices:

```text
Service A
   ↓
Local Transaction
   ↓
Outbox
   ↓
Event
   ↓
Service B
   ↓
Local Transaction
   ↓
Event
   ↓
Service C
```

With failure handling:

```text
Event
 ↓
Consumer
 ↓
Idempotency
 ↓
Local Transaction
 ↓
Success → ACK
 ↓
Failure
 ↓
Retry
 ↓
DLQ

Distributed business failure
 ↓
Saga
 ↓
Compensating Action
```

### The Day 23 golden rule

> **Keep transactions as local, short, and business-meaningful as possible. Use JTA/XA when coordinated atomicity across transactional resources is genuinely required; for independent distributed services, evaluate Outbox + idempotency + Saga/compensation instead.**

**Next — Day 24: Validation & Exception Handling**, covering Jakarta Validation, Bean Validation constraints, custom validators, validation groups, exception hierarchies, REST error handling, global exception mapping, error response contracts, logging, security-safe errors, and production-grade failure handling.
