# Day 21 — Jakarta Messaging (JMS) & Enterprise Asynchronous Processing

**J2EE/Jakarta EE 30-Day Mastery**
**Date: October 23, 2026**

Day 20 taught you advanced EJB. Today we introduce one of the most important concepts in enterprise architecture:

> **How do Java applications communicate asynchronously and reliably without requiring the sender and receiver to execute at the same time?**

You will learn **Jakarta Messaging (JMS)**, historically called **Java Message Service**, and how it works with EJB, transactions, retries, dead-letter queues, idempotency, ordering, and event-driven architecture.

---

# 1. Why Do We Need Messaging?

Consider an e-commerce application.

A customer places an order:

```text
POST /orders
```

The application may need to:

```text
Create order
Reserve inventory
Process payment
Send email
Generate invoice
Update analytics
Notify warehouse
Update recommendation system
```

If everything is synchronous:

```text
Client
  |
  v
Order Service
  |
  +--> Inventory
  |
  +--> Payment
  |
  +--> Email
  |
  +--> Invoice
  |
  +--> Analytics
  |
  v
Response
```

The request can become slow and tightly coupled.

Instead:

```text
Client
  |
  v
Order Service
  |
  v
Message Broker
  |
  +--> Email Consumer
  +--> Invoice Consumer
  +--> Analytics Consumer
  +--> Warehouse Consumer
```

Now the order service doesn't necessarily need to wait for every downstream operation.

---

# 2. What Is Jakarta Messaging?

**Jakarta Messaging** provides a standard API for applications to send and receive messages through messaging providers.

Historically:

```text
JMS
```

Modern Jakarta EE terminology:

```text
Jakarta Messaging
```

The API allows applications to work with messaging infrastructure without tightly coupling business code to a particular provider implementation.

Conceptually:

```text
Producer
   |
   v
Messaging Provider
   |
   v
Consumer
```

---

# 3. Messaging Mental Model

Memorize:

```text
Producer
    |
    | Message
    v
Destination
    |
    v
Consumer
```

For example:

```text
OrderService
    |
    | OrderCreated
    v
OrderQueue
    |
    v
OrderProcessor
```

---

# 4. Synchronous vs Asynchronous Communication

### Synchronous

```text
Client
  |
  v
Service A
  |
  v
Service B
  |
  v
Response
```

A waits for B.

### Asynchronous

```text
Client
  |
  v
Service A
  |
  v
Queue
  |
  v
Service B
```

A can continue after producing the message, depending on the required delivery semantics.

---

# 5. Benefits of Messaging

Messaging can provide:

* Loose coupling
* Asynchronous processing
* Load leveling
* Buffering
* Retry capability
* Independent processing
* Event-driven architecture
* Integration between systems
* Better isolation of failures

Example:

```text
100 requests/sec
      |
      v
    Queue
      |
      v
Consumer processes at
50 requests/sec
```

The queue can temporarily absorb the difference.

But remember:

> A queue doesn't magically solve overload. If producers consistently outpace consumers, backlog will grow.

---

# 6. Jakarta Messaging Architecture

```text
             Jakarta EE Application
                      |
        +-------------+-------------+
        |                           |
        v                           v
     Producer                    Consumer
        |                           ^
        v                           |
   Messaging API               Messaging API
        |                           |
        +-------------+-------------+
                      |
                      v
              Messaging Provider
                      |
                      v
                Destination
```

The provider manages the actual messaging infrastructure.

---

# 7. Core JMS Concepts

You should know:

```text
ConnectionFactory
Destination
Queue
Topic
JMSContext
JMSProducer
JMSConsumer
Message
```

Modern Jakarta Messaging APIs commonly use `JMSContext` to simplify resource handling.

---

# 8. Queue

A queue represents point-to-point messaging.

Example:

```text
Producer
   |
   v
+----------------+
| Order Queue    |
+----------------+
   |
   +----> Consumer
```

Typically one consumer processes a particular message.

If you have:

```text
Consumer A
Consumer B
Consumer C
```

messages can be distributed among consumers according to provider behavior.

Conceptually:

```text
Message 1 → Consumer A
Message 2 → Consumer B
Message 3 → Consumer C
```

---

# 9. Topic

A topic represents publish/subscribe messaging.

```text
             Producer
                 |
                 v
             OrderTopic
          /       |       \
         /        |        \
        v         v         v
   Consumer A Consumer B Consumer C
```

A published event can be delivered to multiple subscribers according to subscription configuration and delivery semantics.

---

# 10. Queue vs Topic

| Feature          | Queue                             | Topic                            |
| ---------------- | --------------------------------- | -------------------------------- |
| Model            | Point-to-point                    | Publish/subscribe                |
| Typical purpose  | Work distribution                 | Event broadcasting               |
| Consumers        | Competing consumers               | Multiple subscribers             |
| Example          | Process payment                   | Order-created event              |
| Message delivery | One processing consumer typically | Multiple subscribers can receive |

Memorize:

```text
Queue → Work
Topic → Broadcast/Event
```

This is a useful mental shortcut, though actual provider semantics can be more nuanced.

---

# 11. Real-World Example

Suppose:

```text
OrderCreated
```

You want:

```text
Inventory
Notification
Analytics
Fraud Detection
```

A topic can conceptually be:

```text
                 OrderCreated
                      |
                      v
                 OrderTopic
               /     |      \
              /      |       \
             v       v        v
       Inventory  Analytics  Notification
```

Each subscriber can process the event independently.

---

# 12. Queue Example

Suppose invoice generation is expensive.

```text
OrderService
     |
     v
InvoiceQueue
     |
  +--+--+
  |     |
  v     v
Worker1 Worker2
```

Messages are distributed among workers.

This allows horizontal processing.

---

# 13. `ConnectionFactory`

A messaging application needs a way to obtain messaging contexts/connections.

Modern Jakarta Messaging code commonly uses:

```java
@Resource(lookup = "jms/OrderConnectionFactory")
private ConnectionFactory connectionFactory;
```

Then:

```java
try (JMSContext context =
         connectionFactory.createContext()) {

    ...
}
```

In a Jakarta EE application, the server can manage the configured resource.

---

# 14. Destination Injection

A queue can be injected:

```java
@Resource(lookup = "jms/OrderQueue")
private Queue orderQueue;
```

A topic:

```java
@Resource(lookup = "jms/OrderTopic")
private Topic orderTopic;
```

The actual JNDI names depend on your application server/provider configuration.

---

# 15. JMS Producer

Modern API:

```java
JMSProducer producer =
    context.createProducer();
```

Send:

```java
producer.send(orderQueue, "Order 123");
```

Conceptually:

```text
Java Object
    |
    v
JMSProducer
    |
    v
Queue
```

---

# 16. JMS Consumer

A basic consumer:

```java
JMSConsumer consumer =
    context.createConsumer(orderQueue);

String message =
    consumer.receiveBody(String.class);
```

This is a synchronous receive from the consumer's perspective.

For enterprise asynchronous processing, a message-driven component is often more appropriate.

---

# 17. Message-Driven Bean

You encountered MDB on Day 20.

A Message-Driven Bean is an EJB component designed to consume messages asynchronously.

Conceptually:

```text
Queue
  |
  v
MDB
  |
  v
Business Service
```

Example:

```java
@MessageDriven
public class OrderConsumer
        implements MessageListener {

    @Override
    public void onMessage(Message message) {

        // process message
    }
}
```

The container manages the MDB lifecycle and message delivery.

---

# 18. MDB Architecture

```text
Producer
   |
   v
Queue
   |
   v
Jakarta Messaging Provider
   |
   v
MDB Container
   |
   v
OrderConsumer
   |
   v
Business Service
   |
   v
Database
```

This is one of the classic Jakarta EE asynchronous processing patterns.

---

# 19. MDB Doesn't Need a Caller

A REST endpoint is invoked by:

```text
HTTP client
```

An MDB is triggered by:

```text
Message arrival
```

So:

```text
REST
 ↓
HTTP request
```

while:

```text
MDB
 ↓
Message delivery
```

---

# 20. Message Types

Jakarta Messaging supports message representations such as:

```text
TextMessage
BytesMessage
MapMessage
ObjectMessage
StreamMessage
```

In modern systems, a common pattern is:

```text
TextMessage
    ↓
JSON
```

For example:

```json
{
  "eventId": "evt-1001",
  "orderId": 123,
  "type": "ORDER_CREATED"
}
```

---

# 21. Don't Blindly Serialize Domain Entities

Avoid sending an entire JPA entity graph:

```java
ObjectMessage(entity)
```

without considering:

* Version compatibility
* Serialization
* Coupling
* Security
* Payload size
* Lazy relationships
* Schema evolution

Prefer an explicit event DTO/schema.

For example:

```java
public record OrderCreatedEvent(
    String eventId,
    Long orderId,
    Long customerId,
    BigDecimal total
) {}
```

---

# 22. Event vs Command

This distinction is important.

### Command

```text
"Process this payment."
```

It asks another component to perform an action.

### Event

```text
"Order 123 was created."
```

It communicates something that already happened.

Conceptually:

```text
Command
    ↓
Please do X

Event
    ↓
X happened
```

---

# 23. Messaging Delivery Semantics

Three commonly discussed models are:

```text
At-most-once
At-least-once
Exactly-once
```

### At-most-once

Message may be lost, but isn't normally redelivered.

```text
0 or 1 delivery
```

### At-least-once

Message should be delivered, but duplicates can occur.

```text
1 or more deliveries
```

### Exactly-once

The desired business effect happens exactly once.

This is much harder to guarantee end-to-end in distributed systems.

---

# 24. Why At-Least-Once Matters

Imagine:

```text
PaymentProcessed
```

arrives twice:

```text
Message
   |
   +--> Process
   |
   +--> Process again
```

If processing isn't idempotent, you could create duplicate effects.

Therefore:

> **Reliable messaging often requires idempotent consumers.**

---

# 25. Idempotency

An operation is idempotent if repeating it produces the same intended business result.

Example:

```text
PUT /employees/100
```

setting:

```text
salary = 50000
```

multiple times results in:

```text
salary = 50000
```

A payment operation is more dangerous:

```text
charge(₹1000)
```

Running twice could charge twice.

So introduce an idempotency key:

```text
paymentId = PAY-10001
```

Consumer:

```text
Already processed PAY-10001?
    |
    +-- YES → Don't repeat
    |
    +-- NO  → Process
```

---

# 26. Idempotency Table

A database can maintain:

```text
processed_messages
------------------
message_id
processed_at
status
```

Processing:

```text
BEGIN
   |
   v
Check message_id
   |
   +--> Already exists → skip
   |
   +--> Doesn't exist
           |
           v
      Insert message_id
           |
           v
      Business operation
           |
           v
         COMMIT
```

When transactionally coordinated with the business update, this can make duplicate delivery safe.

---

# 27. Message Acknowledgement

Acknowledgement communicates that a message has been successfully handled according to the messaging model.

Conceptually:

```text
Broker
  |
  | Message
  v
Consumer
  |
  | Process
  v
ACK
```

If processing fails before the message is considered successfully handled, the provider can potentially redeliver depending on configuration and transaction/acknowledgement mode.

---

# 28. Transactional Messaging

One of the most important enterprise concepts.

Suppose:

```text
OrderConsumer
```

receives:

```text
OrderCreated
```

and performs:

```text
Update database
```

You want:

```text
Message processing
+
Database update
```

to have well-defined atomicity.

With Jakarta EE transaction management, MDB processing can participate in container-managed transactions.

Conceptually:

```text
BEGIN TRANSACTION
      |
      +--> Consume message
      |
      +--> Update database
      |
      v
COMMIT
```

If the transaction fails:

```text
ROLLBACK
```

and the message may become eligible for redelivery according to provider/container configuration.

---

# 29. Transaction Failure

Suppose:

```text
Message: OrderCreated
```

Consumer:

```text
Save order
Update inventory
```

Then:

```text
Database failure
```

Conceptually:

```text
BEGIN
 |
 +--> Save order
 |
 +--> Update inventory
 |
 +--> ERROR
 |
 ROLLBACK
```

The transaction prevents a partial database update when both operations are within the same transaction.

---

# 30. Redelivery

Suppose processing fails:

```text
Message 100
   |
   v
Consumer
   |
   v
ERROR
```

The provider may redeliver:

```text
Message 100
   |
   v
Consumer
   |
   v
Retry
```

Possible sequence:

```text
Attempt 1 → failure
Attempt 2 → failure
Attempt 3 → success
```

You need to configure and reason about retry behavior carefully.

---

# 31. Dead-Letter Queue

What if:

```text
Attempt 1 → fail
Attempt 2 → fail
Attempt 3 → fail
...
```

Forever retrying is usually undesirable.

A dead-letter destination can hold messages that cannot be processed successfully after configured retry handling.

Conceptually:

```text
Main Queue
    |
    v
Consumer
    |
    +---- Success
    |
    +---- Failure
             |
             v
        Retry policy
             |
             v
       Dead Letter Queue
```

---

# 32. Dead-Letter Queue Operations

A production DLQ needs operational handling.

Typical workflow:

```text
DLQ
 |
 +--> Inspect
 |
 +--> Diagnose
 |
 +--> Fix underlying issue
 |
 +--> Replay
```

Don't blindly replay everything.

A message may be failing because of:

* Invalid data
* Schema mismatch
* Business rule
* Missing dependency
* Database outage
* Programming bug

---

# 33. Poison Message

A poison message repeatedly fails processing.

Example:

```json
{
  "orderId": "INVALID"
}
```

Every retry produces:

```text
ERROR
ERROR
ERROR
ERROR
```

Eventually:

```text
DLQ
```

Your architecture should prevent one bad message from consuming all processing capacity.

---

# 34. Message Ordering

Suppose:

```text
OrderCreated
OrderCancelled
```

arrive.

If processed in reverse order:

```text
OrderCancelled
OrderCreated
```

you could get an invalid business state.

Therefore some systems require ordering.

But ordering can reduce parallelism.

Architectural trade-off:

```text
More ordering
    ↓
Potentially less parallelism
```

---

# 35. Ordering by Business Key

A common design is to preserve ordering for a specific entity.

Example:

```text
customerId = 100
```

Events:

```text
E1
E2
E3
```

Need:

```text
E1 → E2 → E3
```

But different customers may process concurrently:

```text
Customer 100 → Worker A
Customer 200 → Worker B
```

This is a powerful distributed-systems pattern.

Exact ordering guarantees depend on the messaging provider/configuration.

---

# 36. Queue Backpressure

Suppose:

```text
Producer = 10,000 msg/sec
Consumer = 2,000 msg/sec
```

Backlog:

```text
Queue
████████████████████████
```

continues growing.

You need:

* More consumers
* Faster processing
* Batching
* Rate limiting
* Backpressure
* Load shedding where appropriate
* Capacity planning

Messaging provides buffering, not infinite capacity.

---

# 37. Scaling Consumers

Instead of:

```text
Queue
 |
 Consumer
```

you can have:

```text
              Queue
                |
       +--------+--------+
       |        |        |
       v        v        v
   Consumer  Consumer  Consumer
      A         B         C
```

This can increase throughput.

But check:

```text
Database capacity
External API limits
Ordering requirements
Concurrency limits
Consumer CPU
Memory
```

Adding consumers isn't always a free performance improvement.

---

# 38. Queue vs Thread Pool

Don't confuse:

```text
JMS Queue
```

with:

```text
ExecutorService
```

A thread pool is an in-process concurrency mechanism.

A messaging system provides application/system-level communication and can provide persistence, delivery semantics, routing, and integration features depending on the provider.

Conceptually:

```text
ExecutorService
    ↓
Same application process

Messaging
    ↓
Potentially separate components/processes/systems
```

---

# 39. JMS vs `@Asynchronous`

You learned `@Asynchronous` on Day 20.

### EJB asynchronous method

```text
Caller
  |
  v
EJB
  |
  +---- async execution
```

### Messaging

```text
Producer
  |
  v
Broker
  |
  v
Consumer
```

Messaging gives you a much stronger communication abstraction for decoupled components.

Use the mechanism based on the reliability, durability, integration, and scaling requirements.

---

# 40. JMS vs REST

REST:

```text
Request
  ↓
Response
```

Messaging:

```text
Message
  ↓
Eventually processed
```

REST is often appropriate when the caller needs an immediate response.

Messaging is useful when:

* Processing can happen later
* Consumers are independent
* Work needs buffering
* Events need distribution
* Systems need decoupling

---

# 41. JMS vs Kafka

At a high level:

### Jakarta Messaging

API/specification for messaging.

A Jakarta Messaging provider implements it.

### Kafka

A distributed event-streaming platform with its own programming model.

They can both support asynchronous/event-driven architectures, but they are not identical technologies.

Think:

```text
Jakarta Messaging
      ↓
Standard Java/Jakarta API

Kafka
      ↓
Distributed event streaming platform
```

---

# 42. JMS Producer Example

Example service:

```java
@Stateless
public class OrderService {

    @Resource(lookup = "jms/OrderConnectionFactory")
    private ConnectionFactory connectionFactory;

    @Resource(lookup = "jms/OrderQueue")
    private Queue orderQueue;

    public void publishOrder(Long orderId) {

        try (JMSContext context =
                 connectionFactory.createContext()) {

            context.createProducer()
                   .send(orderQueue,
                        "ORDER_CREATED:" + orderId);
        }
    }
}
```

For production, prefer an explicit event schema instead of concatenated strings.

---

# 43. Better Event Design

Instead of:

```text
ORDER_CREATED:123
```

use structured JSON:

```json
{
  "eventId": "evt-123",
  "eventType": "ORDER_CREATED",
  "version": 1,
  "occurredAt": "2026-10-23T10:30:00Z",
  "orderId": 123,
  "customerId": 456
}
```

Important fields:

```text
eventId
eventType
version
timestamp
aggregate/entity ID
payload
```

---

# 44. Event Versioning

Today:

```json
{
  "version": 1,
  "orderId": 123
}
```

Later:

```json
{
  "version": 2,
  "orderId": 123,
  "currency": "INR"
}
```

Consumers need to handle schema evolution.

Never assume:

```text
Producer version == Consumer version
```

in a distributed system.

---

# 45. Message Headers and Metadata

Messages can contain metadata such as:

```text
Correlation ID
Message ID
Type
Timestamp
Priority
Delivery information
```

Correlation ID:

```text
requestId = abc-123
```

allows tracing:

```text
HTTP Request
     |
     v
Order Service
     |
     v
JMS Message
     |
     v
Consumer
```

All logs can include:

```text
correlationId=abc-123
```

This is extremely useful in production troubleshooting.

---

# 46. Correlation ID

Example:

```text
HTTP
X-Correlation-ID: abc123
```

Then:

```text
OrderCreated
correlationId = abc123
```

Consumer:

```text
OrderConsumer
correlationId = abc123
```

Now you can trace one business operation across asynchronous boundaries.

---

# 47. Retry Strategy

Don't simply retry forever.

A common conceptual strategy:

```text
Attempt 1
   ↓
wait
   ↓
Attempt 2
   ↓
wait longer
   ↓
Attempt 3
   ↓
DLQ
```

This is related to:

```text
Exponential backoff
Maximum retry count
Dead-letter handling
```

---

# 48. Avoid Retry Storms

Suppose a downstream database is unavailable.

1,000 consumers all retry immediately:

```text
Retry
Retry
Retry
Retry
Retry
...
```

The database gets hammered again.

Better:

```text
Failure
  ↓
Backoff
  ↓
Retry
  ↓
Backoff
  ↓
Retry
```

Potentially combined with circuit-breaker/rate-limiting strategies.

---

# 49. Transactional Outbox

This is an architect-level concept you should understand.

Problem:

```text
Database
+
Message Broker
```

Suppose:

```text
BEGIN
 |
 +--> Save Order
 |
 +--> Publish Message
 |
 +--> Broker failure
 |
 ROLLBACK?
```

Coordinating both systems reliably can become difficult.

A common solution is the **Transactional Outbox**:

```text
BEGIN
 |
 +--> Save Order
 |
 +--> Save Outbox Event
 |
 COMMIT
```

Then:

```text
Outbox
  |
  v
Publisher
  |
  v
Message Broker
```

The order and outbox event are committed atomically in the same database transaction.

---

# 50. Outbox Architecture

```text
              Order Service
                   |
          +--------+--------+
          |                 |
          v                 v
      Order Table       Outbox Table
          |                 |
          +--------+--------+
                   |
                COMMIT
                   |
                   v
              Outbox Relay
                   |
                   v
              Message Broker
                   |
          +--------+--------+
          |        |        |
          v        v        v
       Consumer Consumer Consumer
```

This is extremely useful for reliable event publication.

---

# 51. Inbox Pattern

Another useful pattern is an inbox/idempotency table.

```text
Message
   |
   v
Inbox Table
   |
   +--> Already processed?
   |
   +--> No
          |
          v
      Business logic
```

Combined:

```text
Producer
   |
   v
Outbox
   |
   v
Broker
   |
   v
Consumer
   |
   v
Inbox
   |
   v
Database
```

These patterns are very important in distributed architecture.

---

# 52. JMS Transaction Flow

A typical enterprise consumer might look conceptually like:

```text
Message arrives
      |
      v
Transaction begins
      |
      +--> Read message
      |
      +--> Validate
      |
      +--> Check idempotency
      |
      +--> Update database
      |
      v
Transaction commits
```

Failure:

```text
Message
   |
   v
Transaction
   |
   +--> DB update
   |
   +--> FAILURE
   |
   v
ROLLBACK
   |
   v
Redelivery / retry according to configuration
```

---

# 53. Security Considerations

Messages are data crossing boundaries.

Validate:

```text
Message schema
Message size
Required fields
Allowed values
Authorization context
```

Never blindly trust:

```text
customerId
price
role
accountId
```

from an external message.

A message saying:

```json
{
  "customerId": 100
}
```

doesn't automatically mean the producer is authorized to perform every operation on customer 100.

---

# 54. Sensitive Data

Avoid putting unnecessary sensitive information into messages.

For example, don't casually publish:

```text
Passwords
Authentication tokens
Full payment credentials
Secrets
```

Prefer references:

```json
{
  "paymentId": "PAY-1001"
}
```

rather than sensitive payment data.

---

# 55. Message Size

Huge messages create:

```text
Network overhead
Memory pressure
Serialization cost
Broker storage cost
Consumer latency
```

Prefer compact event payloads.

Instead of:

```text
Entire Customer
Entire Order
Entire Product Catalog
Entire Payment
```

consider:

```json
{
  "orderId": 123,
  "customerId": 456
}
```

and let consumers retrieve additional information when appropriate.

---

# 56. Event-Driven Architecture

Now combine what you've learned.

```text
                    Order Service
                         |
                         v
                  OrderCreated Event
                         |
                         v
                    Event Broker
                         |
       +-----------------+----------------+
       |                 |                |
       v                 v                v
 Inventory           Analytics        Notification
 Service              Service           Service
```

This reduces direct coupling.

Instead of:

```text
OrderService → InventoryService
OrderService → AnalyticsService
OrderService → NotificationService
```

you can have:

```text
OrderService
     |
     v
OrderCreated
     |
     +--> Inventory
     +--> Analytics
     +--> Notification
```

---

# 57. But Don't Turn Everything Into Events

Event-driven architecture introduces complexity.

You now need to handle:

```text
Eventual consistency
Duplicate messages
Ordering
Retries
DLQs
Observability
Schema evolution
Replay
Failure recovery
```

Therefore:

> Use asynchronous messaging where its architectural benefits justify its operational complexity.

---

# 58. Real-World Order Architecture

```text
                     Client
                        |
                        v
                  REST API
                        |
                        v
                  OrderService
                     EJB
                        |
                +-------+-------+
                |               |
                v               v
             Order DB       Outbox
                                |
                              Relay
                                |
                                v
                         Message Broker
                                |
          +---------------------+---------------------+
          |                     |                     |
          v                     v                     v
    Inventory Consumer   Notification Consumer   Analytics Consumer
```

This is a strong architecture to understand for interviews.

---

# 59. Hands-On Project

Build:

## Enterprise Order Event System

### Producer

```text
OrderService
```

Responsibilities:

```text
Create order
Save order
Create event
Publish event
```

### Consumer 1

```text
InventoryConsumer
```

Consumes:

```text
ORDER_CREATED
```

### Consumer 2

```text
NotificationConsumer
```

Consumes:

```text
ORDER_CREATED
```

### Consumer 3

```text
AnalyticsConsumer
```

Consumes:

```text
ORDER_CREATED
```

---

# 60. Event Schema

Use:

```json
{
  "eventId": "evt-001",
  "eventType": "ORDER_CREATED",
  "version": 1,
  "occurredAt": "2026-10-23T10:30:00Z",
  "orderId": 1001,
  "customerId": 501,
  "totalAmount": 2499.00
}
```

Implement:

```text
Producer
Queue/Topic
MDB Consumer
Transaction
Idempotency
Retry
DLQ
Logging
Correlation ID
```

---

# 61. Coding Exercise — Producer

Create:

```java
@Stateless
public class OrderEventPublisher {
}
```

Implement:

```text
publishOrderCreated()
publishOrderCancelled()
publishOrderPaid()
```

---

# 62. Coding Exercise — Consumer

Create:

```java
@MessageDriven
public class OrderCreatedConsumer
        implements MessageListener {

    @Override
    public void onMessage(Message message) {
        // validate
        // process
    }
}
```

Add:

```text
Logging
Validation
Idempotency
Transaction
Error handling
```

---

# 63. Coding Exercise — Idempotency

Create:

```text
processed_messages
```

with:

```text
message_id
processed_at
status
```

Consumer:

```text
Receive message
      |
      v
Check message ID
      |
   +--+--+
   |     |
 exists  new
   |     |
   v     v
 ignore  process
         |
         v
       commit
```

---

# 64. Coding Exercise — Retry

Implement a conceptual retry policy:

```text
Attempt 1
   ↓
Failure
   ↓
Backoff
   ↓
Attempt 2
   ↓
Failure
   ↓
Backoff
   ↓
Attempt 3
   ↓
DLQ
```

Track:

```text
messageId
attemptCount
lastError
lastAttemptAt
```

---

# 65. Coding Exercise — Event Versioning

Create:

```text
OrderCreated v1
OrderCreated v2
```

Implement consumer compatibility.

Think about:

```text
Missing field
New field
Renamed field
Deprecated field
Unknown field
```

---

# 66. Coding Exercise — Outbox

Create:

```text
orders
outbox_events
```

Transaction:

```text
BEGIN
 |
 +--> INSERT order
 |
 +--> INSERT outbox event
 |
COMMIT
```

Then create an outbox publisher:

```text
Outbox
  |
  v
Publisher
  |
  v
JMS
```

This is one of the most valuable exercises in today's lesson.

---

# 67. Day 21 Interview Questions

## Fundamentals

1. What is Jakarta Messaging?
2. What problem does messaging solve?
3. What is a message producer?
4. What is a consumer?
5. What is a destination?
6. What is a queue?
7. What is a topic?
8. Queue vs topic?
9. What is `ConnectionFactory`?
10. What is `JMSContext`?

## MDB

11. What is a Message-Driven Bean?
12. How does an MDB work?
13. How is MDB different from a REST endpoint?
14. How is MDB different from `@Asynchronous`?
15. Can MDBs participate in transactions?
16. How does an MDB handle failure?
17. What happens when a message is redelivered?

## Reliability

18. What is at-most-once delivery?
19. What is at-least-once delivery?
20. What is exactly-once processing?
21. Why is idempotency important?
22. What is a poison message?
23. What is a dead-letter queue?
24. What is retry backoff?
25. What is message ordering?

## Architecture

26. REST vs messaging?
27. JMS vs Kafka?
28. Queue vs topic?
29. What is event-driven architecture?
30. What is eventual consistency?
31. What is the transactional outbox?
32. What is the inbox pattern?
33. How do you design idempotent consumers?
34. How do you scale consumers?
35. How do you prevent retry storms?

---

# 68. Architect-Level Questions

### Scenario 1

Your order API sends:

```text
OrderCreated
```

but the notification service is down.

Should order creation fail?

Think about:

```text
Business criticality
Consistency requirement
User experience
Retry
Durability
```

---

### Scenario 2

A consumer processes:

```text
PaymentCompleted
```

twice.

How do you prevent double business effects?

Think:

```text
Idempotency key
Processed-message store
Atomic transaction
```

---

### Scenario 3

Producer sends:

```text
OrderCreated
```

but the database transaction succeeds while message publishing fails.

How do you solve the reliability gap?

Think:

```text
Transactional Outbox
```

---

### Scenario 4

You have:

```text
100,000 messages
```

and consumers process only:

```text
1,000/sec
```

What happens?

Think:

```text
Queue backlog
Latency
Storage
Scaling
Backpressure
```

---

### Scenario 5

You need:

```text
OrderCreated
OrderPaid
OrderShipped
```

in strict order for each order.

How would you design processing?

Think about:

```text
Partitioning/business key
Ordering
Concurrency
Retries
```

---

# 69. Day 21 Production Checklist

Before using messaging in production, answer:

### Reliability

* Can messages be lost?
* Can messages be duplicated?
* What happens after consumer crash?
* What happens after broker failure?

### Processing

* Is the consumer idempotent?
* What is the retry policy?
* Is there a DLQ?
* How are poison messages handled?

### Performance

* How many consumers?
* What's the throughput?
* What's the maximum backlog?
* What's the consumer processing latency?

### Ordering

* Does order matter?
* For the whole system or only per entity?
* Does ordering limit scalability?

### Schema

* How are events versioned?
* Can old consumers process new messages?
* How are breaking changes handled?

### Observability

Track:

```text
Message ID
Correlation ID
Event type
Queue depth
Processing latency
Retry count
Failure count
DLQ count
Consumer throughput
```

---

# 70. Day 21 Mental Model

Memorize:

```text
             PRODUCER
                 |
                 v
          Message/Event
                 |
                 v
          Queue / Topic
                 |
          +------+------+
          |             |
          v             v
      Consumer A    Consumer B
          |             |
          v             v
       Business      Business
       Logic         Logic
          |             |
          v             v
       Database      Database
```

With reliability:

```text
Message
   |
   v
Consumer
   |
   +--> Idempotency
   |
   +--> Transaction
   |
   +--> Business Logic
   |
   +--> Success → ACK
   |
   +--> Failure → Retry
                     |
                     v
                    DLQ
```

And for reliable publication:

```text
Business Operation
       |
       v
Database Transaction
       |
       +--> Business Data
       |
       +--> Outbox Event
       |
       v
     COMMIT
       |
       v
Outbox Publisher
       |
       v
Message Broker
       |
       v
Consumers
```

# Day 21 Golden Rule

> **Messaging is not simply "send something to a queue." It is a distributed-systems mechanism with delivery semantics, transactions, retries, ordering, idempotency, backpressure, observability, and failure handling.**

For your solutions-architect goal, the key question is no longer:

> "How do I send a JMS message?"

It is:

```text
Should this interaction be synchronous or asynchronous?

What happens if the consumer is down?

Can the message be duplicated?

Can the message be lost?

Do we need ordering?

Where is the transaction boundary?

How do we retry?

What goes to the DLQ?

How do we scale consumers?

How do we evolve the event schema?

How do we trace the message end-to-end?
```

If you can answer those questions and implement the **Order → Outbox → Broker → MDB Consumer → Idempotency → Retry → DLQ** flow, you've understood the core of enterprise messaging.

**Day 22 → Enterprise Integration Patterns: Message Router, Filter, Translator, Aggregator, Splitter, Content-Based Router, Dead Letter Channel, Retry, Idempotent Consumer, Competing Consumers, Correlation Identifier, Request-Reply, and Saga-style workflows.**
