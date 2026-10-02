# Day 13 — Transactions in J2EE / Jakarta EE

**Date:** October 15, 2026
**Goal:** Master enterprise transactions from **JDBC → JPA → JTA → EJB → Jakarta EE**, and understand how transaction boundaries, ACID, isolation, propagation, rollback, locking, and distributed transactions work.

By the end of today, you should be able to answer:

> **“What happens when an enterprise operation updates multiple database records and something fails halfway?”**

---

# 1. What Is a Transaction?

A **transaction** is a logical unit of work that should be completed according to defined consistency rules.

Example: transferring ₹10,000 from Account A to Account B.

There are two database operations:

```text
Debit A: -₹10,000
Credit B: +₹10,000
```

We don't want:

```text
Debit A → SUCCESS
Credit B → FAILURE
```

because money would disappear.

Instead:

```text
BEGIN TRANSACTION

Debit A
Credit B

COMMIT
```

If anything fails:

```text
BEGIN TRANSACTION

Debit A
Credit B → FAILURE

ROLLBACK
```

Result:

```text
Account A = original balance
Account B = original balance
```

---

# 2. Why Transactions Are Necessary

Without transactions:

```text
Request
   ↓
Update DB 1
   ↓
Update DB 2
   ↓
Update DB 3
   ↓
Exception
```

The database may contain partially completed work.

With a transaction:

```text
Request
   ↓
BEGIN
   ↓
DB Operation 1
   ↓
DB Operation 2
   ↓
DB Operation 3
   ↓
SUCCESS?
 ┌───────┴───────┐
YES             NO
 ↓                ↓
COMMIT          ROLLBACK
```

---

# 3. ACID Properties

Transactions are commonly described using **ACID**.

## A — Atomicity

All operations succeed or all are rolled back.

```text
A + B + C

SUCCESS → commit everything

FAILURE → rollback everything
```

---

## C — Consistency

The transaction should move the database from one valid state to another valid state according to its constraints and business rules.

Example:

```text
balance >= 0
```

A valid transaction shouldn't leave the database violating such rules.

---

## I — Isolation

Concurrent transactions should not interfere in unacceptable ways.

Example:

```text
Transaction A
Transaction B
```

Both are modifying the same account.

Isolation determines what one transaction can observe from another.

---

## D — Durability

After commit, changes should survive failures according to the database's durability guarantees.

```text
COMMIT
  ↓
Database records committed state
  ↓
Application crashes
  ↓
Data remains
```

---

# 4. Transaction Lifecycle

The basic lifecycle:

```text
START
  ↓
ACTIVE
  ↓
SQL/JPA operations
  ↓
 ┌───────────────┐
 │               │
SUCCESS         ERROR
 │               │
COMMIT          ROLLBACK
 │               │
COMPLETED       ABORTED
```

---

# 5. Basic JDBC Transaction

JDBC allows manual transaction management.

By default, a JDBC connection usually operates with:

```java
connection.setAutoCommit(true);
```

That means each SQL statement can be committed individually.

For a transaction:

```java
connection.setAutoCommit(false);
```

Then:

```java
connection.commit();
```

or:

```java
connection.rollback();
```

---

# 6. JDBC Transaction Example

```java
public void transferMoney(
        long fromAccount,
        long toAccount,
        BigDecimal amount) throws SQLException {

    Connection connection = dataSource.getConnection();

    try {
        connection.setAutoCommit(false);

        debit(connection, fromAccount, amount);

        credit(connection, toAccount, amount);

        connection.commit();

    } catch (Exception e) {

        connection.rollback();

        throw e;

    } finally {

        connection.setAutoCommit(true);
        connection.close();
    }
}
```

The important part:

```java
setAutoCommit(false)
```

followed by:

```java
commit()
```

or:

```java
rollback()
```

---

# 7. Why `finally` Matters

Connection pooling introduces an important issue.

A connection returned to the pool may retain transaction state if you don't clean it correctly.

For example:

```java
connection.setAutoCommit(false);
```

Then:

```java
connection.close();
```

With a connection pool, `close()` generally returns the connection to the pool rather than physically destroying it.

Therefore, transaction state must be correctly cleaned up.

Production code should ensure the connection is returned to a clean state.

---

# 8. Savepoints

JDBC supports savepoints.

Example:

```text
BEGIN
 ↓
Operation A
 ↓
SAVEPOINT S1
 ↓
Operation B
 ↓
Operation C
 ↓
ROLLBACK TO S1
```

Example:

```java
Savepoint savepoint = connection.setSavepoint();

try {
    performOperationB(connection);
} catch (Exception e) {
    connection.rollback(savepoint);
}
```

This rolls back to a particular point rather than necessarily rolling back the entire transaction.

---

# 9. Transaction Boundaries

One of the most important enterprise concepts:

> **Where does a transaction begin and end?**

Consider:

```text
Controller
    ↓
Service
    ↓
DAO
    ↓
Database
```

Usually, the **business operation** is the transaction boundary.

For example:

```text
transferMoney()
```

rather than:

```text
debit()
```

and:

```text
credit()
```

independently.

Correct conceptual boundary:

```text
transferMoney()
   |
   +--- debit()
   |
   +--- credit()
   |
   +--- audit()
```

All required operations belong to one transaction when atomicity demands it.

---

# 10. Why DAO Should Usually Not Own the Business Transaction

Bad design:

```text
DAO.debit()
   → BEGIN
   → UPDATE
   → COMMIT

DAO.credit()
   → BEGIN
   → UPDATE
   → COMMIT
```

Problem:

```text
debit → committed
credit → failed
```

The transfer isn't atomic.

Better:

```text
Service
  ↓
BEGIN
  ↓
DAO.debit()
  ↓
DAO.credit()
  ↓
COMMIT
```

The service/application layer understands the business operation.

---

# 11. JPA Transactions

JPA operations generally execute within a transaction when they modify persistent state.

With application-managed JPA:

```java
EntityTransaction transaction =
        entityManager.getTransaction();

try {
    transaction.begin();

    // JPA operations

    transaction.commit();

} catch (Exception e) {

    if (transaction.isActive()) {
        transaction.rollback();
    }

    throw e;
}
```

---

# 12. `EntityTransaction`

For resource-local transactions:

```java
EntityTransaction tx =
        entityManager.getTransaction();
```

Start:

```java
tx.begin();
```

Commit:

```java
tx.commit();
```

Rollback:

```java
tx.rollback();
```

Check:

```java
tx.isActive();
```

---

# 13. JPA Transaction Example

Suppose:

```java
@Entity
public class Account {

    @Id
    private Long id;

    private BigDecimal balance;

    // getters/setters
}
```

Transfer:

```java
public void transfer(
        Long fromId,
        Long toId,
        BigDecimal amount) {

    EntityTransaction tx =
            entityManager.getTransaction();

    try {
        tx.begin();

        Account from =
                entityManager.find(Account.class, fromId);

        Account to =
                entityManager.find(Account.class, toId);

        from.setBalance(
                from.getBalance().subtract(amount)
        );

        to.setBalance(
                to.getBalance().add(amount)
        );

        tx.commit();

    } catch (Exception e) {

        if (tx.isActive()) {
            tx.rollback();
        }

        throw e;
    }
}
```

Because the entities are managed, dirty checking can detect their changes.

---

# 14. Persistence Context + Transaction

Remember the Day 12 concept:

```text
EntityManager
      ↓
Persistence Context
      ↓
Managed Entities
```

During a transaction:

```text
BEGIN
 ↓
find(Account)
 ↓
Managed entity
 ↓
modify entity
 ↓
flush
 ↓
SQL UPDATE
 ↓
COMMIT
```

---

# 15. Flush vs Commit

These are **not the same thing**.

### Flush

Synchronizes the persistence context with the database.

```text
Java state
   ↓
flush()
   ↓
SQL sent to DB
```

### Commit

Completes the transaction.

```text
Transaction
   ↓
COMMIT
   ↓
Transaction successfully completed
```

Conceptually:

```text
flush != commit
```

A flushed SQL operation can still be rolled back before transaction commit.

---

# 16. JTA — Java Transaction API

In enterprise applications, you often need a transaction manager capable of coordinating multiple transactional resources.

That's where **JTA/Jakarta Transactions** comes in.

Conceptually:

```text
Application
     ↓
Transaction Manager
     ↓
 ┌───────────┬───────────┐
 ↓           ↓           ↓
Database    JMS        Other resource
```

JTA provides APIs and infrastructure for transaction management in enterprise environments.

---

# 17. Resource-Local vs JTA Transactions

### Resource-local

Typically one resource manages its own transaction.

```text
Application
    ↓
EntityManager
    ↓
Database
```

### JTA

A transaction manager coordinates participating resources.

```text
Application
      ↓
JTA Transaction Manager
      ↓
 ┌────┴────┐
 ↓         ↓
Database   JMS
```

---

# 18. Why JTA Exists

Imagine an order system:

```text
Create Order
      ↓
Update Inventory
      ↓
Save Payment Record
      ↓
Send Message
```

Potentially:

```text
Database
+
Messaging system
```

The enterprise application may need coordinated transactional behavior.

JTA provides the transaction-management abstraction needed for such scenarios.

---

# 19. Container-Managed Transactions

One of the major Jakarta EE concepts is that the container can manage transactions for you.

Instead of:

```java
tx.begin();

try {
    ...
    tx.commit();
} catch (...) {
    tx.rollback();
}
```

the container can manage the transaction around your business method.

Conceptually:

```text
Client
  ↓
Container
  ↓
BEGIN TRANSACTION
  ↓
Business Method
  ↓
COMMIT / ROLLBACK
```

This is particularly important with EJB and other Jakarta EE components.

---

# 20. Container-Managed Transaction Flow

Imagine:

```java
public void createOrder() {

    saveOrder();

    updateInventory();

    savePayment();
}
```

The container can conceptually perform:

```text
BEGIN
 ↓
createOrder()
 ↓
saveOrder()
 ↓
updateInventory()
 ↓
savePayment()
 ↓
COMMIT
```

If an appropriate failure causes rollback:

```text
BEGIN
 ↓
createOrder()
 ↓
saveOrder()
 ↓
updateInventory()
 ↓
ERROR
 ↓
ROLLBACK
```

---

# 21. `@Transactional`

Jakarta applications can use declarative transaction management with `@Transactional` where supported by the component/programming model.

Conceptually:

```java
@Transactional
public void transferMoney(
        Long from,
        Long to,
        BigDecimal amount) {

    debit(from, amount);

    credit(to, amount);
}
```

The infrastructure handles transaction demarcation.

This is conceptually similar to transaction management commonly encountered in Spring.

---

# 22. Declarative vs Programmatic Transactions

## Programmatic

You explicitly control the transaction.

```java
tx.begin();

try {
    ...
    tx.commit();
} catch (Exception e) {
    tx.rollback();
}
```

## Declarative

You declare the transaction boundary.

```java
@Transactional
public void processOrder() {
    ...
}
```

The container/runtime handles the mechanics.

### Mental model

```text
Programmatic:
YOU → manage transaction

Declarative:
CONTAINER → manage transaction
```

---

# 23. Transaction Propagation

Propagation answers:

> **What should happen when a transactional method calls another transactional method?**

Example:

```text
Service A
   ↓
Service B
```

Suppose A already has a transaction.

What happens when B executes?

Possible strategies include:

```text
Join existing transaction
Create a new transaction
Execute without transaction
```

Propagation semantics are implementation/framework-specific, but the concept is universal.

---

# 24. Common Propagation Concepts

You should understand these concepts thoroughly, especially because you'll encounter them in enterprise frameworks:

### REQUIRED

```text
Existing transaction?
    ↓
YES → join it
NO  → create one
```

### REQUIRES_NEW

```text
Existing transaction?
    ↓
YES → suspend it
     ↓
     create new transaction
```

### SUPPORTS

```text
Existing transaction?
    ↓
YES → participate
NO  → execute without transaction
```

### NOT_SUPPORTED

```text
Existing transaction?
    ↓
YES → suspend it
     ↓
execute without transaction
```

### MANDATORY

```text
Existing transaction?
 ↓
YES → use it
NO  → error
```

### NEVER

```text
Existing transaction?
 ↓
YES → error
NO  → execute
```

### NOT_SUPPORTED vs REQUIRES_NEW

Important interview distinction:

```text
REQUIRES_NEW
    existing transaction suspended
    new transaction started

NOT_SUPPORTED
    existing transaction suspended
    method executes without transaction
```

---

# 25. Transaction Isolation

Isolation controls how concurrent transactions interact.

Typical levels:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

---

# 26. READ UNCOMMITTED

A transaction may see data that another transaction hasn't committed.

Potential problem:

### Dirty read

```text
Transaction A
UPDATE balance = 500
(no commit)

Transaction B
READ balance
→ 500

Transaction A
ROLLBACK
```

B observed data that never became committed.

---

# 27. READ COMMITTED

A transaction only sees committed data from other transactions.

It prevents dirty reads.

But another transaction may modify data between two reads.

Example:

```text
Transaction A:
READ balance → 1000

Transaction B:
UPDATE balance → 500
COMMIT

Transaction A:
READ balance → 500
```

The same query can observe different committed values.

This is a **non-repeatable read**.

---

# 28. REPEATABLE READ

Provides stronger guarantees for repeated reads of the same rows, subject to the database's implementation.

Conceptually:

```text
Transaction A
READ → 1000

Transaction B
UPDATE → 500
COMMIT

Transaction A
READ
→ expected stable view for the previously read row
```

Exact behavior around phantom rows depends on the database and isolation implementation.

---

# 29. SERIALIZABLE

Strongest standard isolation level.

Transactions behave as though conflicting transactions were serialized rather than freely interleaved.

Conceptually:

```text
T1
 ↓
T2 waits / conflicts
 ↓
T1 finishes
 ↓
T2 proceeds
```

Advantage:

```text
strong consistency guarantees
```

Cost:

```text
less concurrency
potential blocking
potential deadlocks
lower throughput
```

---

# 30. Isolation Anomalies

Memorize these:

| Anomaly             | Meaning                                       |
| ------------------- | --------------------------------------------- |
| Dirty read          | Read uncommitted data                         |
| Non-repeatable read | Same row gives different values               |
| Phantom read        | Re-running a range query sees different rows  |
| Lost update         | One update unintentionally overwrites another |

---

# 31. Isolation Mental Model

Think:

```text
READ_UNCOMMITTED
      ↓
READ_COMMITTED
      ↓
REPEATABLE_READ
      ↓
SERIALIZABLE
```

Generally, stronger isolation can mean greater concurrency cost, but exact behavior depends on the database engine and implementation.

---

# 32. Transaction Rollback

Rollback means:

> Undo changes made within the current transaction according to the transaction system's semantics.

Example:

```java
try {
    transaction.begin();

    createOrder();
    updateInventory();
    createPayment();

    transaction.commit();

} catch (Exception e) {

    transaction.rollback();

    throw e;
}
```

---

# 33. What Causes Rollback?

Possible causes:

```text
Business exception
Database exception
Constraint violation
Deadlock
Timeout
Explicit rollback
System/runtime failure
```

But the exact rollback behavior depends on the transaction technology and exception configuration.

---

# 34. Checked vs Runtime Exceptions

This is an important enterprise interview topic.

Don't memorize:

> "Every exception automatically rolls back."

That is too simplistic.

Transaction frameworks and containers can define rollback behavior based on:

```text
exception type
transaction configuration
component type
annotation/configuration
explicit rollback
```

For example, declarative transaction systems may distinguish between runtime exceptions and checked exceptions.

Always understand the actual transaction framework's rules.

---

# 35. Explicit Rollback

Sometimes the business logic determines that the transaction must be rolled back.

Conceptually:

```java
if (!businessRuleSatisfied()) {
    // mark transaction for rollback
}
```

Jakarta transaction APIs provide mechanisms for marking the current transaction rollback-only.

---

# 36. Transaction Timeout

Transactions shouldn't run forever.

Example:

```text
BEGIN
 ↓
Database operation
 ↓
External dependency hangs
 ↓
Transaction remains open
```

This can hold:

```text
DB connections
locks
memory
resources
```

Therefore production systems often configure transaction timeouts.

---

# 37. Long Transactions Are Dangerous

Bad:

```text
BEGIN TRANSACTION

Read DB
 ↓
Call external API
 ↓
Wait 10 seconds
 ↓
Process file
 ↓
Call another service
 ↓
UPDATE DB
 ↓
COMMIT
```

The transaction may hold resources for too long.

Better:

```text
Short DB transaction
        ↓
External operation
        ↓
Another short transaction
```

when the business process allows it.

---

# 38. Transaction + External API

Very important architecture question.

Suppose:

```text
Database Transaction
       +
Payment API
```

You cannot assume a normal database rollback will undo an external HTTP request.

Example:

```text
BEGIN DB TRANSACTION
 ↓
Create order
 ↓
Call payment API
 ↓
Payment succeeds
 ↓
Database update fails
 ↓
ROLLBACK DB
```

The payment service may still have the payment.

Therefore:

> A local DB transaction does not magically make external systems transactional.

This leads to concepts such as:

```text
Saga
Compensation
Outbox Pattern
Idempotency
Event-driven workflows
```

You'll study these more deeply later.

---

# 39. Distributed Transactions

Suppose:

```text
Application
   ↓
DB1
   +
DB2
```

One logical transaction needs to coordinate multiple resources.

This is significantly more complicated than a single database transaction.

Conceptually:

```text
             Transaction Manager
                 /        \
                /          \
              DB1          DB2
```

---

# 40. Two-Phase Commit — 2PC

A classic distributed transaction protocol is **Two-Phase Commit**.

Two major phases:

```text
Phase 1: PREPARE
Phase 2: COMMIT
```

---

## Phase 1 — Prepare

Coordinator asks:

```text
Can you commit?
```

Participants respond:

```text
YES
```

or:

```text
NO
```

---

## Phase 2 — Commit

If everyone says YES:

```text
Coordinator
    ↓
COMMIT
    ↓
DB1
DB2
```

If someone says NO:

```text
ROLLBACK
```

---

# 41. 2PC Diagram

```text
              Coordinator
             /            \
            ↓              ↓
          DB1             DB2

        PREPARE          PREPARE
           ↓                ↓
          YES              YES
           \                /
            \              /
             ↓            ↓
                 COMMIT
```

If:

```text
DB1 → YES
DB2 → NO
```

then:

```text
ROLLBACK
```

---

# 42. Advantages of 2PC

It can provide coordinated atomic commit across participating transactional resources.

Useful when strong transactional coordination is genuinely required.

---

# 43. Problems With 2PC

Distributed transactions have substantial costs:

```text
network communication
coordination overhead
locking/resource retention
failure handling complexity
availability impact
operational complexity
```

Therefore:

> Don't introduce distributed transactions merely because you can.

Architecture should first determine whether the business requirement actually needs atomic distributed commit.

---

# 44. Local Transaction vs Distributed Transaction

### Local

```text
Application
   ↓
Database
```

### Distributed

```text
Application
      ↓
Transaction Coordinator
   ↙       ↓       ↘
 DB1      DB2      JMS
```

---

# 45. Transaction + JMS

A classic enterprise scenario:

```text
Create Order
     ↓
Update Database
     ↓
Publish Message
```

You may want:

```text
DB update succeeds
AND
message is reliably committed
```

Coordinating database and messaging resources is one of the classic reasons enterprise transaction management exists.

However, modern architectures frequently use patterns such as the **Transactional Outbox** instead of trying to make every distributed operation one giant transaction.

---

# 46. Transactional Outbox

A powerful architecture pattern.

Instead of:

```text
DB transaction
   +
Message broker transaction
```

you can do:

```text
BEGIN DB TRANSACTION
      ↓
Save Order
      ↓
Save Outbox Event
      ↓
COMMIT
```

Then:

```text
Outbox Publisher
      ↓
Reads event
      ↓
Publishes to broker
```

Architecture:

```text
              DB
       ┌───────────────┐
       │ orders        │
       │ outbox_events │
       └───────┬───────┘
               │
               ↓
        Outbox Publisher
               │
               ↓
          Kafka/JMS/etc.
```

This is extremely useful for modern enterprise systems.

---

# 47. Optimistic Locking

From Day 12, remember:

```java
@Version
private Long version;
```

Suppose:

```text
T1 reads Employee version 5
T2 reads Employee version 5
```

T1 updates:

```text
version 5 → 6
```

T2 attempts update using version 5.

The persistence provider/database detects the conflict.

This helps prevent silent lost updates.

---

# 48. Pessimistic Locking

Pessimistic locking acquires a database lock when data is read/updated under the relevant transaction.

Example:

```java
entityManager.find(
    Account.class,
    id,
    LockModeType.PESSIMISTIC_WRITE
);
```

Conceptually:

```text
T1 locks account
     ↓
T2 attempts modification
     ↓
wait/conflict
```

Use only when the concurrency requirements justify the locking behavior.

---

# 49. Optimistic vs Pessimistic

| Feature           | Optimistic                        | Pessimistic                      |
| ----------------- | --------------------------------- | -------------------------------- |
| Assumption        | Conflicts are relatively uncommon | Conflicts need active prevention |
| Typical mechanism | Version checking                  | Database locks                   |
| Blocking          | Usually less                      | Potentially more                 |
| Throughput        | Often better under low conflict   | Can degrade under contention     |
| Failure           | Detect conflict                   | Prevent/block conflict           |
| JPA example       | `@Version`                        | `PESSIMISTIC_WRITE`              |

---

# 50. Deadlocks

A deadlock occurs when transactions wait for each other indefinitely until the database detects the situation.

Example:

```text
T1 locks A
T2 locks B

T1 waits for B
T2 waits for A
```

Diagram:

```text
T1 → waiting for B
 ↑             ↓
 A           T2
 ↑             ↓
 └──── waiting for A
```

Database systems can detect deadlocks and abort one transaction.

---

# 51. How to Reduce Deadlocks

Common techniques:

### 1. Consistent lock ordering

Always lock:

```text
Account A
then Account B
```

instead of sometimes:

```text
A → B
```

and sometimes:

```text
B → A
```

### 2. Keep transactions short

### 3. Avoid unnecessary locks

### 4. Use appropriate indexes

### 5. Retry safely where appropriate

---

# 52. Transaction Retry

Some failures are transient.

Example:

```text
Transaction
    ↓
Deadlock
    ↓
Rollback
    ↓
Retry
```

But retrying is safe only when the operation is designed appropriately.

This is where **idempotency** becomes extremely important.

---

# 53. Idempotency + Transactions

Consider:

```text
POST /payments
```

Client sends request.

Server processes payment.

Network timeout occurs before response reaches client.

Client retries.

Without idempotency:

```text
Payment 1
Payment 2
```

Potential duplicate.

With an idempotency key:

```text
Idempotency-Key: abc123
```

the server can recognize the repeated request.

Transactions and idempotency solve different problems:

```text
Transaction
→ atomicity within transactional resources

Idempotency
→ safe handling of repeated requests
```

---

# 54. Transaction Boundary in an Enterprise Application

Recommended conceptual architecture:

```text
HTTP Request
     ↓
Controller
     ↓
Service
     ↓
@Transactional boundary
     ↓
 ┌───────────────┐
 ↓               ↓
DAO 1           DAO 2
 ↓               ↓
DB              DB
 └───────┬───────┘
         ↓
       COMMIT
```

The transaction should normally represent a meaningful business operation rather than a random DAO call.

---

# 55. Example: Order Creation

Business operation:

```text
createOrder()
```

Operations:

```text
1. Validate customer
2. Create order
3. Create order items
4. Reserve inventory
5. Calculate total
6. Save payment record
```

Potential transaction:

```text
BEGIN

createOrder()
createOrderItems()
reserveInventory()
savePaymentRecord()

COMMIT
```

If inventory reservation fails:

```text
ROLLBACK
```

---

# 56. But Don't Put Everything Into One Transaction

Suppose:

```text
Create order
 ↓
Send email
 ↓
Generate PDF
 ↓
Call shipping API
 ↓
Call analytics API
```

Putting all of this into one transaction can be problematic.

Better architecture may be:

```text
Transaction 1
    ↓
Create Order
    ↓
Commit
    ↓
Publish OrderCreated
    ↓
 ┌───────┬─────────┬─────────┐
 ↓       ↓         ↓
Email  Shipping  Analytics
```

This leads toward event-driven architecture.

---

# 57. Transaction Synchronization

Enterprise transaction systems can provide lifecycle callbacks around transaction completion.

Conceptually:

```text
Before completion
       ↓
Transaction completes
       ↓
After completion
```

This can be useful for infrastructure-level behavior such as synchronization, resource cleanup, or triggering appropriate post-transaction processing.

Be careful not to hide critical business workflows inside transaction callbacks.

---

# 58. Transaction and Caching

Potential problem:

```text
BEGIN
 ↓
DB UPDATE
 ↓
Cache UPDATE
 ↓
DB ROLLBACK
```

Now cache may contain incorrect data.

A safer conceptual flow is often:

```text
BEGIN
 ↓
DB UPDATE
 ↓
COMMIT
 ↓
Invalidate/update cache
```

or use appropriate transactional/event-driven cache strategies.

The important point:

> Cache consistency is a separate concern from database transaction atomicity.

---

# 59. Transaction and Microservices

Suppose:

```text
Order Service
Payment Service
Inventory Service
Shipping Service
```

Each may have its own database:

```text
Order DB
Payment DB
Inventory DB
Shipping DB
```

Trying to create one giant distributed transaction across all services can introduce significant coupling and operational complexity.

Modern designs often use:

```text
Saga
Outbox
Events
Compensation
Idempotency
Retries
```

instead.

---

# 60. Saga Concept

A Saga breaks a business transaction into multiple local transactions.

Example:

```text
Create Order
    ↓
Reserve Inventory
    ↓
Process Payment
    ↓
Create Shipment
```

If payment fails:

```text
Cancel Order
Release Inventory
```

These are **compensating actions**, not database rollbacks across all services.

---

# 61. Database Transaction vs Saga

### Database transaction

```text
BEGIN
 ↓
A
 ↓
B
 ↓
C
 ↓
ROLLBACK
```

### Saga

```text
Local Tx A
   ↓
Local Tx B
   ↓
Local Tx C
   ↓
failure
   ↓
Compensating action
   ↓
Compensating action
```

This distinction is essential for a future solutions architect.

---

# 62. Transactional Thinking for Architects

When designing a system, ask:

### Question 1

What must be atomic?

### Question 2

Which operations use the same transactional resource?

### Question 3

Where should the transaction start?

### Question 4

Where should it end?

### Question 5

What happens on failure?

### Question 6

What isolation level is required?

### Question 7

Could locking cause contention?

### Question 8

Can the operation be retried?

### Question 9

Is idempotency required?

### Question 10

Are multiple resources/services involved?

---

# 63. Complete Enterprise Transaction Flow

Understand this architecture deeply:

```text
                 HTTP Request
                      ↓
               Controller
                      ↓
                 Service
                      ↓
             Transaction Boundary
                      ↓
        ┌─────────────┴─────────────┐
        ↓                           ↓
      DAO 1                       DAO 2
        ↓                           ↓
      JPA                         JDBC
        ↓                           ↓
        └────────── Database ───────┘
                      ↓
                   flush
                      ↓
                 Transaction
                 Manager
                      ↓
               COMMIT / ROLLBACK
                      ↓
                 HTTP Response
```

---

# 64. Production Transaction Rules

Memorize these.

### Rule 1

Keep transactions short.

### Rule 2

Don't call slow external services inside DB transactions unless there is a compelling reason and the design accounts for the consequences.

### Rule 3

Define transaction boundaries around business operations.

### Rule 4

Don't assume rollback can undo external side effects.

### Rule 5

Choose isolation deliberately.

### Rule 6

Watch for deadlocks.

### Rule 7

Use optimistic locking where appropriate.

### Rule 8

Use pessimistic locking only when justified.

### Rule 9

Make retries safe.

### Rule 10

Use idempotency for retryable business operations where needed.

### Rule 11

Don't assume distributed transactions are automatically the best architecture.

### Rule 12

Monitor:

```text
transaction duration
rollback rate
deadlocks
lock waits
connection pool usage
database latency
timeout rate
```

---

# 65. Common Transaction Mistakes

## Mistake 1 — Transaction per DAO operation

```text
DAO A → commit
DAO B → commit
```

Can break business atomicity.

---

## Mistake 2 — Huge transaction

```text
BEGIN

20 seconds of work

COMMIT
```

Potentially holds resources too long.

---

## Mistake 3 — External API inside DB transaction

```text
BEGIN
 ↓
HTTP API
 ↓
wait
 ↓
DB UPDATE
 ↓
COMMIT
```

Can create long-running transactions and still doesn't make the HTTP call rollbackable.

---

## Mistake 4 — Ignoring isolation

Concurrent operations may produce incorrect business behavior.

---

## Mistake 5 — Ignoring deadlocks

Locks must be designed carefully.

---

## Mistake 6 — Assuming rollback means everything everywhere is undone

It doesn't.

A local database rollback cannot automatically reverse:

```text
email
HTTP request
external payment
Kafka message already consumed
file upload
```

---

# 66. Interview Questions

## Beginner

### 1. What is a transaction?

A logical unit of work that should satisfy defined atomicity and consistency requirements.

### 2. What is ACID?

```text
Atomicity
Consistency
Isolation
Durability
```

### 3. What is commit?

Makes the transaction's changes final according to the resource's transaction semantics.

### 4. What is rollback?

Aborts the transaction and reverses transactional changes that can be rolled back.

### 5. What is auto-commit?

Each individual database operation is committed automatically rather than grouped into an explicit transaction.

---

# 67. Intermediate Interview Questions

### 6. What is transaction isolation?

It defines how concurrent transactions interact and what changes they can observe.

### 7. What is dirty read?

Reading data written by another transaction before it commits.

### 8. What is a non-repeatable read?

Reading the same row twice and seeing different committed values because another transaction modified it between reads.

### 9. What is phantom read?

A repeated range query observes a different set of rows because another transaction inserted/deleted matching rows.

### 10. What is deadlock?

Two or more transactions wait on resources held by one another.

---

# 68. Advanced Interview Questions

### 11. JTA vs JDBC transaction?

JDBC transactions manage transactional behavior at the JDBC/resource level, while JTA provides a broader transaction-management abstraction suitable for coordinating transactional resources in enterprise environments.

### 12. Local vs distributed transaction?

Local transactions typically involve one transactional resource; distributed transactions coordinate multiple resources.

### 13. What is 2PC?

A distributed commit protocol with prepare and commit phases coordinated by a transaction coordinator.

### 14. Why are distributed transactions expensive?

Because of coordination, network communication, failure handling, resource retention, and reduced flexibility/availability.

### 15. What is transaction propagation?

Rules defining how a method participates in an existing transaction or creates/suspends transactions when called from another transactional context.

---

# 69. Architect-Level Interview Questions

### 16. Should every service method be transactional?

Not necessarily.

The transaction boundary should correspond to the business operation and actual consistency requirements.

---

### 17. Should every DAO method start a transaction?

Usually not when transaction management is intended at the application/service boundary.

---

### 18. Should an HTTP call happen inside a database transaction?

Sometimes technically possible, but generally undesirable when avoidable because it can lengthen the transaction and hold DB resources. The correct architecture depends on the business consistency requirement.

---

### 19. Can DB rollback undo a Kafka message?

Not automatically.

You need appropriate transaction coordination or an architecture such as transactional outbox depending on the system.

---

### 20. How do microservices maintain consistency?

Common approaches include:

```text
Local transactions
+
Events
+
Saga
+
Outbox
+
Idempotency
+
Compensation
```

rather than assuming one global database transaction.

---

# 70. Hands-On Exercises

Implement these in Java.

### Exercise 1

Create JDBC transaction:

```text
Employee salary update
+
Audit record
```

Both should commit or rollback together.

---

### Exercise 2

Implement:

```java
transferMoney()
```

with:

```text
debit
credit
commit
rollback
```

---

### Exercise 3

Add a savepoint.

---

### Exercise 4

Create two concurrent transactions and observe isolation behavior.

---

### Exercise 5

Create a deadlock scenario with two database sessions.

---

### Exercise 6

Implement JPA optimistic locking using:

```java
@Version
```

---

### Exercise 7

Implement pessimistic locking:

```java
PESSIMISTIC_WRITE
```

---

### Exercise 8

Implement an idempotent payment operation.

---

### Exercise 9

Implement a transactional outbox.

---

### Exercise 10

Design an order Saga:

```text
Order
Inventory
Payment
Shipping
```

and define compensating actions.

---

# 71. Day 13 Mini Project

## Banking Transaction System

Build:

```text
BankingWebApp
```

### Features

```text
Create Account
Deposit
Withdraw
Transfer
Transaction History
```

Architecture:

```text
Browser
   ↓
Servlet
   ↓
BankingService
   ↓
Transaction Boundary
   ↓
AccountDAO
   ↓
JPA/JDBC
   ↓
Database
```

### Transfer operation

```text
BEGIN
   ↓
Load sender
   ↓
Load receiver
   ↓
Validate balance
   ↓
Debit sender
   ↓
Credit receiver
   ↓
Create transaction record
   ↓
COMMIT
```

Failure:

```text
ANY ERROR
   ↓
ROLLBACK
```

---

# 72. Recommended Project Structure

```text
src/main/java/
└── com.example.banking
    ├── controller
    │   └── TransferServlet.java
    │
    ├── service
    │   └── BankingService.java
    │
    ├── dao
    │   ├── AccountDAO.java
    │   └── TransactionDAO.java
    │
    ├── entity
    │   ├── Account.java
    │   └── BankTransaction.java
    │
    ├── dto
    │   └── TransferRequest.java
    │
    ├── exception
    │   ├── InsufficientBalanceException.java
    │   └── AccountNotFoundException.java
    │
    └── filter
        └── AuthenticationFilter.java

src/main/webapp/
└── WEB-INF/
    └── views/
        ├── transfer.jsp
        └── history.jsp
```

---

# 73. Banking Database

```sql
CREATE TABLE account (
    id BIGINT PRIMARY KEY,
    account_number VARCHAR(50) UNIQUE NOT NULL,
    balance DECIMAL(19,2) NOT NULL,
    version BIGINT NOT NULL
);
```

Transaction table:

```sql
CREATE TABLE bank_transaction (
    id BIGINT PRIMARY KEY,
    from_account BIGINT,
    to_account BIGINT,
    amount DECIMAL(19,2) NOT NULL,
    status VARCHAR(30) NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

---

# 74. Complete Transaction Mental Model

You should now understand:

```text
                  TRANSACTION
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
     Atomicity      Isolation      Durability
        │              │
        │              ├── Dirty Read
        │              ├── Non-repeatable
        │              ├── Phantom
        │              └── Lost Update
        │
        ↓
    Commit/Rollback
        │
        ↓
 Transaction Boundary
        │
        ├── JDBC
        ├── JPA
        ├── JTA
        └── Container
                │
                ↓
        Distributed Transactions
                │
           ┌────┴────┐
           ↓         ↓
          2PC       Saga
                     │
              Compensation
                     │
              Outbox/Events
```

---

# 75. Day 13 Mastery Checklist

Before moving to Day 14, you should be able to explain without notes:

* [ ] What is a transaction?
* [ ] ACID
* [ ] Commit vs rollback
* [ ] JDBC transactions
* [ ] Auto-commit
* [ ] Savepoints
* [ ] JPA transactions
* [ ] Persistence context + transactions
* [ ] Flush vs commit
* [ ] JTA
* [ ] Resource-local vs JTA
* [ ] Container-managed transactions
* [ ] Declarative vs programmatic transactions
* [ ] Transaction boundaries
* [ ] Propagation concepts
* [ ] Isolation levels
* [ ] Dirty reads
* [ ] Non-repeatable reads
* [ ] Phantom reads
* [ ] Lost updates
* [ ] Transaction timeout
* [ ] Deadlocks
* [ ] Optimistic locking
* [ ] Pessimistic locking
* [ ] 2PC
* [ ] Distributed transactions
* [ ] Transaction + JMS
* [ ] Transaction + external API
* [ ] Transactional Outbox
* [ ] Saga
* [ ] Compensation
* [ ] Idempotency + retry
* [ ] Transaction + caching
* [ ] Microservice transaction design

## The most important takeaway

As a future solutions architect, don't think of transactions as merely:

```java
begin();
commit();
rollback();
```

Think of them as an **architecture boundary**:

```text
Business Operation
       ↓
What must be atomic?
       ↓
Which resources participate?
       ↓
Where does transaction begin?
       ↓
What isolation is required?
       ↓
What happens when something fails?
       ↓
Can we retry?
       ↓
Is idempotency required?
       ↓
Local transaction or distributed workflow?
       ↓
Commit / Rollback / Compensation
```

**Day 14** in your plan is the **midpoint Employee Management System**, where you'll combine Servlets + JSP + EL/JSTL + MVC + JDBC/JPA + transactions + filters + validation into a complete enterprise application.
