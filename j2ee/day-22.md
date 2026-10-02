# Day 22 — Enterprise Integration Patterns (EIP)

**J2EE/Jakarta EE 30-Day Mastery**
**Date:** October 24, 2026
**Goal:** Master how enterprise applications communicate, route, transform, aggregate, retry, correlate, and recover from failures.

Today is one of the most important architecture days because **enterprise applications rarely work in isolation**.

---

# 1. What is Enterprise Integration?

Enterprise integration means connecting multiple applications, services, databases, message brokers, and external systems so they can work together.

For example:

```text
                 ┌──→ Payment Service
                 │
Customer → Order Service → Inventory Service
                 │
                 ├──→ Notification Service
                 │
                 └──→ Analytics Service
```

These systems may:

* run on different servers
* use different technologies
* have different databases
* communicate synchronously or asynchronously
* fail independently
* process different message formats

Integration patterns provide **reusable solutions to these communication problems**.

---

# 2. Why Enterprise Integration Patterns?

Imagine an e-commerce system:

```text
Order
  ↓
Inventory
  ↓
Payment
  ↓
Shipping
  ↓
Notification
```

Without integration patterns, code can become:

```text
OrderService
 ├── callPayment()
 ├── callInventory()
 ├── callShipping()
 ├── sendEmail()
 ├── sendSMS()
 ├── updateAnalytics()
 └── callERP()
```

This creates tight coupling.

Instead:

```text
OrderService
      ↓
   OrderCreated
      ↓
 Message Broker
   ↓    ↓    ↓
Payment Inventory Notification
```

Now systems are more independent.

---

# 3. Enterprise Integration Mental Model

Remember this:

```text
Producer
   ↓
Message
   ↓
Channel
   ↓
Router
   ↓
Translator
   ↓
Processor
   ↓
Consumer
```

Additional mechanisms:

```text
Retry
DLQ
Idempotency
Correlation
Aggregation
Splitting
Filtering
Monitoring
```

---

# 4. Pattern Categories

You should organize EIPs into these groups.

### Messaging patterns

* Message Channel
* Point-to-Point Channel
* Publish-Subscribe Channel
* Dead Letter Channel
* Guaranteed Delivery
* Competing Consumers

### Routing patterns

* Message Router
* Content-Based Router
* Recipient List
* Routing Slip
* Dynamic Router
* Filter

### Transformation patterns

* Message Translator
* Normalizer
* Content Enricher
* Claim Check

### Message construction

* Command Message
* Event Message
* Document Message
* Envelope Wrapper

### Aggregation patterns

* Aggregator
* Splitter
* Resequencer

### Reliability

* Retry
* Idempotent Consumer
* Circuit Breaker
* Dead Letter Queue
* Poison Message Handling

### Correlation

* Correlation Identifier
* Message Expiration
* Request-Reply

---

# 5. Message Channel

A **Message Channel** is the communication path between a producer and consumer.

```text
Producer
   ↓
Message Channel
   ↓
Consumer
```

In Jakarta Messaging, the channel can be represented through a:

```text
Queue
```

or:

```text
Topic
```

---

# 6. Point-to-Point Channel

One message is consumed by one consumer.

```text
Producer
   ↓
 Queue
   ↓
 ┌───────┬────────┬────────┐
 C1      C2       C3
```

Suppose:

```text
Order Queue
```

has:

```text
Consumer 1
Consumer 2
Consumer 3
```

Each message is processed by only one consumer.

This is useful for:

* background jobs
* order processing
* invoice generation
* image processing
* email jobs

---

# 7. Publish-Subscribe Channel

One message can be delivered to multiple subscribers.

```text
                 ┌──→ Consumer A
                 │
Producer → Topic ├──→ Consumer B
                 │
                 └──→ Consumer C
```

Example:

```text
OrderCreated
```

may be consumed by:

```text
Inventory
Payment
Analytics
Notification
```

This is appropriate for events.

---

# 8. Message Router

A **Message Router** decides where a message should go.

```text
                 ┌──→ Payment
Message → Router ├──→ Inventory
                 └──→ Notification
```

Example:

```java
public String route(OrderEvent event) {

    if ("PAYMENT".equals(event.getType())) {
        return "paymentQueue";
    }

    if ("INVENTORY".equals(event.getType())) {
        return "inventoryQueue";
    }

    return "unknownQueue";
}
```

The router separates routing logic from message producers.

---

# 9. Content-Based Router

A Content-Based Router chooses the destination based on message content.

Example:

```json
{
  "orderId": 1001,
  "country": "IN",
  "amount": 50000
}
```

Routing:

```text
country = IN
     ↓
India fulfillment

country = US
     ↓
US fulfillment
```

Architecture:

```text
              ┌──→ India Service
Message → Router
              └──→ US Service
```

---

# 10. Message Filter

A filter determines whether a message should continue.

```text
Message
   ↓
Filter
   ↓
valid?
 ┌─┴─┐
Yes  No
 ↓    ↓
Queue Drop
```

Example:

```java
if (order.getAmount() <= 0) {
    return;
}
```

Production use cases:

* invalid events
* unwanted events
* unsupported versions
* business-rule filtering
* security filtering

---

# 11. Recipient List

A message needs to be sent to multiple destinations.

```text
              ┌──→ Payment
              │
Order → List ─┼──→ Inventory
              │
              └──→ Notification
```

For example:

```java
List<String> recipients =
        List.of(
            "paymentQueue",
            "inventoryQueue",
            "notificationQueue"
        );
```

Useful when destinations are determined dynamically.

---

# 12. Routing Slip

A **Routing Slip** carries information about the sequence of processing steps.

Example:

```text
Order
 ↓
Validate
 ↓
Reserve Inventory
 ↓
Charge Payment
 ↓
Create Shipment
 ↓
Notify Customer
```

The route may be represented as:

```text
[
  validate,
  inventory,
  payment,
  shipping,
  notification
]
```

Useful when workflow steps vary dynamically.

---

# 13. Dynamic Router

A Dynamic Router determines routing dynamically rather than using a fixed route.

```text
Message
   ↓
Dynamic Router
   ↓
Routing rules
   ↓
Destination
```

For example:

```text
customer tier = GOLD
      ↓
Premium processing

customer tier = STANDARD
      ↓
Normal processing
```

---

# 14. Message Translator

Different systems may use different formats.

System A:

```json
{
  "customerId": 100,
  "total": 5000
}
```

System B expects:

```json
{
  "client_id": 100,
  "amount": 5000
}
```

Translator:

```text
System A
   ↓
Translator
   ↓
System B
```

Java:

```java
public PaymentRequest translate(Order order) {

    PaymentRequest request = new PaymentRequest();

    request.setClientId(order.getCustomerId());
    request.setAmount(order.getTotal());

    return request;
}
```

---

# 15. Normalizer

Suppose multiple systems produce different representations.

```text
System A → Format A
System B → Format B
System C → Format C
```

Normalize them:

```text
Format A ─┐
Format B ─┼→ Normalizer → Canonical Format
Format C ─┘
```

This is useful in integration-heavy enterprise systems.

---

# 16. Content Enricher

A message may not contain enough information.

Example:

```json
{
  "orderId": 1001,
  "customerId": 50
}
```

We need:

```text
customer name
customer address
customer tier
```

The enricher retrieves additional information.

```text
Message
   ↓
Content Enricher
   ↓
Customer DB/API
   ↓
Enriched Message
```

---

# 17. Claim Check Pattern

Sometimes a message is too large.

Instead of sending the entire payload:

```text
Huge File
   ↓
Storage
   ↓
Reference ID
   ↓
Message Broker
```

Message:

```json
{
  "fileId": "FILE-1001"
}
```

Consumer:

```text
fileId
  ↓
Object Storage
  ↓
Actual File
```

This is useful for:

* large files
* documents
* images
* videos
* large payloads

---

# 18. Splitter

A single message may contain multiple logical items.

Example:

```json
{
  "orderId": 1001,
  "items": [
    {"sku": "A"},
    {"sku": "B"},
    {"sku": "C"}
  ]
}
```

Splitter:

```text
Order
 ↓
Splitter
 ├── Item A
 ├── Item B
 └── Item C
```

Useful when each item needs independent processing.

---

# 19. Aggregator

The opposite of Splitter.

```text
Item A ─┐
Item B ─┼→ Aggregator → Complete Order
Item C ─┘
```

Example:

```text
Order
 ├── Payment result
 ├── Inventory result
 └── Shipping result
```

Aggregator waits for related messages and creates a combined result.

---

# 20. Correlation Identifier

Suppose:

```text
Request
 ↓
Payment Service
```

The response must be associated with the correct request.

Use:

```text
correlationId
```

Example:

```json
{
  "messageId": "MSG-1001",
  "correlationId": "ORDER-5001"
}
```

Response:

```json
{
  "messageId": "MSG-2001",
  "correlationId": "ORDER-5001"
}
```

Now the system knows:

```text
MSG-2001
     ↓
ORDER-5001
```

---

# 21. Request-Reply

Some messaging systems need request-response semantics.

```text
Client
 ↓
Request Queue
 ↓
Service
 ↓
Reply Queue
 ↓
Client
```

This resembles synchronous communication but uses messaging infrastructure.

Example:

```text
Order Service
     ↓
Payment Request
     ↓
Payment Service
     ↓
Payment Response
```

---

# 22. Competing Consumers

Multiple consumers process messages from the same queue.

```text
                ┌──→ Consumer 1
                │
Queue ──────────┼──→ Consumer 2
                │
                └──→ Consumer 3
```

Advantages:

* parallel processing
* scalability
* workload distribution

Example:

```text
100,000 image-processing jobs
          ↓
       Queue
          ↓
  10 consumers
```

---

# 23. Idempotent Consumer

This is extremely important.

Suppose:

```text
Payment message
```

is delivered twice.

Without idempotency:

```text
₹1000
+
₹1000
=
₹2000 charged
```

With idempotency:

```text
Message ID
   ↓
Already processed?
   ↓
Yes → Ignore
No  → Process
```

Example:

```java
if (processedMessageRepository.exists(messageId)) {
    return;
}

process(message);

processedMessageRepository.save(messageId);
```

Important caveat:

The **business operation and recording the processed message need appropriate transactional coordination**. Otherwise crashes can create duplicates or lost processing.

---

# 24. Dead Letter Channel / Dead Letter Queue

Some messages cannot be processed.

Example:

```text
Queue
 ↓
Consumer
 ↓
Processing failure
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
DLQ
```

DLQ means:

**Dead Letter Queue**

It prevents permanently broken messages from blocking normal processing.

---

# 25. Poison Message

A poison message is a message that repeatedly fails processing.

Example:

```json
{
  "amount": "INVALID"
}
```

Consumer:

```text
Process
 ↓
Fail
 ↓
Retry
 ↓
Fail
 ↓
Retry
 ↓
Fail
```

Eventually:

```text
DLQ
```

Production systems should have a maximum retry policy.

---

# 26. Retry Pattern

Basic:

```text
Message
 ↓
Consumer
 ↓
Failure
 ↓
Retry
```

Better:

```text
Attempt 1
   ↓
100 ms

Attempt 2
   ↓
500 ms

Attempt 3
   ↓
2 sec

Attempt 4
   ↓
DLQ
```

Use **exponential backoff**.

Avoid:

```text
retry immediately forever
```

because that can create a retry storm.

---

# 27. Message Expiration

Messages can become useless after a certain time.

Example:

```text
OTP generation request
```

If processed after 30 minutes, it may be invalid.

Therefore:

```text
Message
 ↓
Expiration Time
 ↓
Expired?
 ├── Yes → discard/DLQ
 └── No → process
```

---

# 28. Resequencer

Messages may arrive out of order.

Expected:

```text
1
2
3
4
5
```

Received:

```text
1
3
2
5
4
```

A resequencer restores ordering:

```text
1
2
3
4
5
```

Useful when message ordering matters.

---

# 29. Guaranteed Delivery

Enterprise systems often need to avoid message loss.

Conceptually:

```text
Producer
   ↓
Persistent Broker
   ↓
Consumer
```

Possible mechanisms:

* durable queues
* acknowledgements
* transactions
* retries
* persistent messages
* replication
* dead-letter queues

Remember:

> Reliable delivery is not the same as successful business processing.

---

# 30. Delivery Semantics

You should know these extremely well.

### At-most-once

```text
0 or 1 delivery
```

Possible loss, no duplicate processing.

### At-least-once

```text
1 or more deliveries
```

Possible duplicates.

Therefore:

```text
At-least-once
+
Idempotent Consumer
```

is a common practical design.

### Exactly-once

The goal is:

```text
exactly one effect
```

But end-to-end exactly-once behavior across independent systems is difficult.

Don't casually claim:

```text
"Kafka/JMS guarantees exactly-once business processing."
```

Always distinguish broker delivery semantics from end-to-end business effects.

---

# 31. Message Ordering

Suppose:

```text
AccountCreated
AccountClosed
```

arrive as:

```text
AccountClosed
AccountCreated
```

This can produce incorrect behavior.

Ordering can be achieved through techniques such as:

* partitioning by business key
* ordered queues
* single consumer per key
* sequence numbers
* resequencing

Example:

```text
accountId = 100

100 → Partition 1
```

All events for account 100 can be processed in order within that processing model.

---

# 32. Transactional Messaging

Suppose:

```text
Update Database
       +
Send Message
```

You don't want:

```text
DB update succeeds
Message fails
```

because now the systems disagree.

One important solution is the **Transactional Outbox Pattern**.

---

# 33. Transactional Outbox

Instead of:

```text
DB transaction
     +
Message broker
```

do:

```text
Database Transaction
       ↓
┌───────────────────────┐
│ Order                 │
│ Outbox Event          │
└───────────────────────┘
       ↓
   Commit
       ↓
Outbox Relay
       ↓
Message Broker
```

Example:

```text
BEGIN TRANSACTION

INSERT order

INSERT outbox_event

COMMIT
```

Then:

```text
Outbox Relay
     ↓
Broker
```

This significantly reduces the dual-write problem.

---

# 34. Inbox Pattern

The receiving side can also maintain an inbox.

```text
Message
   ↓
Inbox Table
   ↓
Already processed?
   ├── Yes → Ignore
   └── No  → Process
```

Typical combination:

```text
Producer:
Transactional Outbox

Consumer:
Inbox / Idempotency
```

This is a powerful enterprise integration architecture.

---

# 35. Saga-Style Workflow

Suppose:

```text
Order
 ↓
Payment
 ↓
Inventory
 ↓
Shipping
```

There is no single database transaction across all services.

Instead, each service performs a local transaction and the workflow coordinates the steps.

```text
Order Created
     ↓
Payment
     ↓
Inventory
     ↓
Shipping
```

If inventory fails:

```text
Inventory Failure
      ↓
Compensating Action
      ↓
Refund Payment
```

This is the basic idea behind a **Saga**.

---

# 36. Orchestration vs Choreography

### Orchestration

One coordinator controls the workflow.

```text
           Orchestrator
          /     |      \
         ↓      ↓       ↓
    Payment Inventory Shipping
```

### Choreography

Services react to events.

```text
OrderCreated
     ↓
Payment Service
     ↓
PaymentCompleted
     ↓
Inventory Service
     ↓
InventoryReserved
     ↓
Shipping Service
```

Both have different operational and architectural tradeoffs.

---

# 37. Event vs Command

Very important distinction.

### Command

Tells someone to perform an action.

```text
ProcessPayment
```

It is generally imperative.

### Event

States something already happened.

```text
PaymentCompleted
```

It is generally a fact.

Think:

```text
Command:
"Do this."

Event:
"This happened."
```

---

# 38. Event Envelope

Production events often use an envelope.

```json
{
  "eventId": "EVT-1001",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-10-24T10:30:00Z",
  "correlationId": "ORDER-1001",
  "producer": "order-service",
  "payload": {
    "orderId": 1001,
    "customerId": 500
  }
}
```

This provides metadata needed for:

* tracing
* correlation
* versioning
* debugging
* idempotency

---

# 39. Event Versioning

Suppose version 1:

```json
{
  "orderId": 1001,
  "amount": 500
}
```

Version 2:

```json
{
  "orderId": 1001,
  "amount": 500,
  "currency": "INR"
}
```

Consumers may still be using V1.

Therefore:

```text
Event Versioning
+
Backward Compatibility
```

is critical.

Avoid casually breaking existing event consumers.

---

# 40. Jakarta Messaging Example

A producer:

```java
import jakarta.inject.Inject;
import jakarta.jms.JMSContext;
import jakarta.jms.Queue;

public class OrderProducer {

    @Inject
    JMSContext context;

    @Inject
    Queue orderQueue;

    public void send(OrderCreatedEvent event) {

        context
            .createProducer()
            .send(orderQueue, event);
    }
}
```

A consumer can be implemented using an MDB:

```java
import jakarta.ejb.ActivationConfigProperty;
import jakarta.ejb.MessageDriven;
import jakarta.jms.Message;
import jakarta.jms.MessageListener;

@MessageDriven(
    activationConfig = {
        @ActivationConfigProperty(
            propertyName = "destinationType",
            propertyValue = "jakarta.jms.Queue"
        )
    }
)
public class OrderConsumer implements MessageListener {

    @Override
    public void onMessage(Message message) {

        // validate
        // deserialize
        // process
        // persist
    }
}
```

Exact activation configuration is application-server/broker dependent, so treat this as a conceptual template.

---

# 41. Production Message Processing

A production consumer should conceptually do:

```text
Receive Message
      ↓
Validate Envelope
      ↓
Validate Payload
      ↓
Check Idempotency
      ↓
Start Transaction
      ↓
Execute Business Logic
      ↓
Persist Changes
      ↓
Commit
      ↓
ACK
```

Failure:

```text
Failure
   ↓
Rollback
   ↓
Retry
   ↓
Retry exhausted?
   ↓
DLQ
```

---

# 42. Observability for Messaging

Every enterprise messaging system should expose:

### Metrics

* messages received
* messages processed
* processing latency
* failures
* retry count
* DLQ count
* queue depth
* consumer lag

### Logs

Include:

```text
messageId
correlationId
eventType
consumer
attempt
timestamp
```

### Tracing

Example:

```text
HTTP Request
   ↓
Order Service
   ↓
Message
   ↓
Payment Service
   ↓
Inventory Service
```

Use a correlation/trace identifier to connect the complete workflow.

---

# 43. Security

Don't blindly trust messages.

Validate:

* message schema
* message type
* event version
* authorization context where applicable
* payload constraints
* producer identity
* signatures where required

Avoid putting sensitive information into events unnecessarily.

Instead of:

```json
{
  "cardNumber": "...",
  "cvv": "..."
}
```

prefer:

```json
{
  "paymentId": "PAY-1001"
}
```

and let authorized systems retrieve sensitive information through controlled mechanisms.

---

# 44. Complete Enterprise Example

Let's design an order platform.

```text
                    ┌───────────────┐
                    │ Order Service │
                    └───────┬───────┘
                            │
                       OrderCreated
                            │
                            ▼
                    ┌───────────────┐
                    │ Message Broker│
                    └───────┬───────┘
                            │
          ┌─────────────────┼──────────────────┐
          ↓                 ↓                  ↓
    Payment Service   Inventory Service   Notification
          │                 │                  │
          ↓                 ↓                  ↓
     PaymentDone       StockReserved       Email/SMS
          │                 │
          └────────────┬────┘
                       ↓
                  Order Workflow
```

Add:

```text
Outbox
Inbox
Idempotency
Retry
DLQ
Correlation ID
Tracing
Metrics
Event Versioning
```

Now you have a realistic enterprise integration architecture.

---

# 45. EIP + Jakarta EE Architecture

A typical architecture could be:

```text
                  Client
                    │
                    ▼
              Jakarta REST
                    │
                    ▼
                Controller
                    │
                    ▼
                 Service
                    │
             ┌──────┴──────┐
             ▼             ▼
          Database       Outbox
                            │
                            ▼
                       JMS/Broker
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
         Consumer A     Consumer B      Consumer C
             │              │              │
             ▼              ▼              ▼
           DB/API          DB/API          DB/API
```

This is the architecture you should start thinking about as a **solutions architect**, not merely as a Java developer.

---

# 46. Important EIP Comparison

| Pattern                | Main Purpose                                        |
| ---------------------- | --------------------------------------------------- |
| Message Router         | Decide destination                                  |
| Content-Based Router   | Route based on message content                      |
| Filter                 | Remove unwanted messages                            |
| Recipient List         | Send to multiple destinations                       |
| Routing Slip           | Define processing sequence                          |
| Translator             | Convert message format                              |
| Normalizer             | Convert different formats to common format          |
| Content Enricher       | Add missing information                             |
| Claim Check            | Replace large payload with reference                |
| Splitter               | One message → many                                  |
| Aggregator             | Many messages → one                                 |
| Resequencer            | Restore message order                               |
| Correlation Identifier | Match related messages                              |
| Request-Reply          | Request + asynchronous/synchronous response pattern |
| Idempotent Consumer    | Safely handle duplicates                            |
| Dead Letter Channel    | Handle permanently failed messages                  |
| Retry                  | Recover from transient failures                     |
| Competing Consumers    | Parallel processing                                 |
| Outbox                 | Solve DB + message dual-write problem               |
| Inbox                  | Track received/processed messages                   |

---

# 47. Day 22 Hands-On Project

## Build: Enterprise Order Processing Platform

### Requirements

Create:

```text
Order Service
Payment Consumer
Inventory Consumer
Notification Consumer
```

Flow:

```text
POST /orders
      ↓
Order Service
      ↓
DB + Outbox
      ↓
OrderCreated
      ↓
Message Broker
      ↓
Consumers
```

### Implement

**Phase 1**

Create:

```text
Order
OrderItem
Customer
```

**Phase 2**

Create:

```text
OrderCreatedEvent
PaymentRequested
InventoryReservationRequested
```

**Phase 3**

Create producer.

**Phase 4**

Create consumers.

**Phase 5**

Add:

```text
messageId
correlationId
eventType
eventVersion
```

**Phase 6**

Implement idempotency.

**Phase 7**

Implement retry.

**Phase 8**

Implement DLQ handling.

**Phase 9**

Implement Outbox.

**Phase 10**

Add logging and metrics.

---

# 48. Suggested Project Structure

```text
order-platform/
│
├── order-service/
│   └── src/main/java/
│       └── com/example/order/
│           ├── api/
│           ├── service/
│           ├── domain/
│           ├── repository/
│           ├── messaging/
│           ├── outbox/
│           └── config/
│
├── payment-consumer/
│
├── inventory-consumer/
│
├── notification-consumer/
│
└── shared-events/
    └── src/main/java/
        └── com/example/events/
```

For larger systems, be deliberate about whether sharing event classes between services creates unwanted coupling; a versioned schema/contract is often preferable.

---

# 49. Day 22 Coding Exercises

Implement these in Java:

### Beginner

1. Implement a message queue abstraction.
2. Implement a message router.
3. Implement a content-based router.
4. Implement a filter.
5. Implement a recipient list.
6. Implement a message translator.
7. Implement a splitter.
8. Implement an aggregator.

### Intermediate

9. Implement an idempotent consumer.
10. Implement retry with exponential backoff.
11. Implement a DLQ mechanism.
12. Implement correlation IDs.
13. Implement message expiration.
14. Implement a resequencer.
15. Implement competing consumers.
16. Implement an outbox table design.
17. Implement an inbox table design.

### Advanced

18. Design a payment processing pipeline.
19. Design a distributed order workflow.
20. Design a Saga-style order cancellation flow.
21. Design event versioning.
22. Design a poison-message recovery system.
23. Design message deduplication.
24. Design a high-throughput notification system.
25. Design a multi-region event-processing architecture.

---

# 50. Day 22 Interview Questions

You should be able to answer these without notes.

### Fundamentals

1. What are Enterprise Integration Patterns?
2. Why are EIPs necessary?
3. What is a Message Channel?
4. Point-to-point vs publish-subscribe?
5. What is a Message Router?
6. What is a Content-Based Router?
7. What is a Message Filter?
8. What is a Recipient List?
9. What is a Routing Slip?
10. What is a Dynamic Router?

### Transformation

11. What is Message Translator?
12. What is Normalizer?
13. What is Content Enricher?
14. What is Claim Check?
15. Why shouldn't services exchange internal database entities?

### Aggregation

16. What is Splitter?
17. What is Aggregator?
18. How does an Aggregator know when processing is complete?
19. What is a Resequencer?
20. How do you handle out-of-order messages?

### Reliability

21. What is an Idempotent Consumer?
22. Why are duplicate messages possible?
23. What is a Dead Letter Queue?
24. What is a poison message?
25. How should retry be implemented?
26. Why use exponential backoff?
27. What is a retry storm?
28. What is guaranteed delivery?
29. Explain at-most-once delivery.
30. Explain at-least-once delivery.
31. Why is exactly-once difficult?

### Distributed systems

32. What is the Outbox Pattern?
33. What problem does Outbox solve?
34. What is the Inbox Pattern?
35. What is a Correlation Identifier?
36. What is Request-Reply?
37. What is a Saga?
38. Orchestration vs choreography?
39. Event vs command?
40. How do you handle event versioning?

### Architecture

41. How would you design an order-processing system?
42. How would you handle duplicate payments?
43. How would you process 1 million messages?
44. How would you prevent one bad message from blocking the queue?
45. How would you monitor a messaging system?
46. How would you design message ordering?
47. How would you handle consumer failures?
48. How would you handle broker failure?
49. How would you design DLQ recovery?
50. How would you make an event-driven system observable?

---

# 51. Architect-Level Scenario

### Problem

An e-commerce company receives:

```text
100,000 orders/minute
```

Each order needs:

```text
Payment
Inventory
Fraud Check
Notification
Analytics
Shipping
```

Design the integration.

A reasonable architecture to explore is:

```text
                   API
                    │
                    ▼
              Order Service
                    │
              DB + Outbox
                    │
                    ▼
              Message Broker
                    │
       ┌────────────┼──────────────┐
       ▼            ▼              ▼
   Payment      Inventory       Fraud
       │            │              │
       └────────────┼──────────────┘
                    │
                    ▼
                Events
             ┌──────┴──────┐
             ▼             ▼
       Notification     Analytics
```

Then ask yourself:

```text
What happens if Payment fails?
What happens if Inventory fails?
What happens if a message is duplicated?
What happens if the consumer crashes?
What happens if the broker is unavailable?
What happens if events arrive out of order?
What happens if a consumer is slow?
What happens if an event schema changes?
What happens if the same payment request arrives twice?
```

Those questions are the real **enterprise architecture thinking**.

---

# 52. Day 22 Mastery Checklist

Before moving to Day 23, you should understand:

### Messaging

* [ ] Queue
* [ ] Topic
* [ ] Message Channel
* [ ] Point-to-point
* [ ] Publish-subscribe

### Routing

* [ ] Message Router
* [ ] Content-Based Router
* [ ] Filter
* [ ] Recipient List
* [ ] Routing Slip
* [ ] Dynamic Router

### Transformation

* [ ] Translator
* [ ] Normalizer
* [ ] Content Enricher
* [ ] Claim Check

### Aggregation

* [ ] Splitter
* [ ] Aggregator
* [ ] Resequencer

### Reliability

* [ ] Retry
* [ ] Exponential Backoff
* [ ] DLQ
* [ ] Poison Message
* [ ] Idempotent Consumer
* [ ] Guaranteed Delivery

### Distributed Transactions

* [ ] Outbox
* [ ] Inbox
* [ ] Saga
* [ ] Compensation

### Enterprise Architecture

* [ ] Correlation ID
* [ ] Event vs Command
* [ ] Event Versioning
* [ ] Ordering
* [ ] Backpressure
* [ ] Consumer Scaling
* [ ] Observability

---

# 53. Day 22 Golden Mental Model

Memorize this architecture:

```text
                 PRODUCER
                    │
                    ▼
                  EVENT
                    │
                    ▼
              MESSAGE CHANNEL
                    │
                    ▼
                 ROUTER
              ┌─────┼─────┐
              ▼     ▼     ▼
           SERVICE SERVICE SERVICE
              │     │     │
              ▼     ▼     ▼
             DB    DB    DB
```

With production reliability:

```text
Producer
   ↓
Outbox
   ↓
Broker
   ↓
Router
   ↓
Consumer
   ↓
Idempotency
   ↓
Transaction
   ↓
Database
   ↓
ACK

Failure
   ↓
Retry
   ↓
Exponential Backoff
   ↓
Retry Exhausted
   ↓
DLQ
```

And for distributed workflows:

```text
Command
   ↓
Service
   ↓
Event
   ↓
Service
   ↓
Event
   ↓
Service
```

**The key lesson for Day 22:** EIPs are not merely JMS features. They are reusable architectural solutions for **communication, routing, transformation, reliability, coordination, scalability, and failure handling across enterprise systems**.

**Next: Day 23 — JTA & Distributed Transactions**, where you'll connect today's messaging concepts with transaction boundaries, XA, two-phase commit, transaction managers, distributed transaction failure modes, and Saga-based alternatives.
