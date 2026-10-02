# Day 29 — Jakarta EE Architecture & System Design

**Date:** October 31, 2026
**Goal:** Move from *“I can build Jakarta EE applications”* to *“I can design enterprise systems using Jakarta EE and explain why I chose each architectural component.”*

You have already covered:

* Servlets
* JSP/JSTL
* MVC
* JDBC
* JPA
* Transactions/JTA
* REST
* Security
* CDI
* EJB
* Messaging
* EIP
* Validation
* Application servers
* JNDI
* Testing
* Deployment/DevOps

Today is about **connecting everything into architecture**.

---

# 1. What You Should Be Able to Do After Day 29

Given a business requirement, you should be able to identify:

```text
Business Requirement
        ↓
Functional Requirements
        ↓
Non-Functional Requirements
        ↓
Architecture
        ↓
Components
        ↓
Data Model
        ↓
APIs
        ↓
Transactions
        ↓
Messaging
        ↓
Security
        ↓
Scaling
        ↓
Deployment
        ↓
Observability
```

You should be able to answer:

> Why did you choose this architecture?

rather than merely:

> What technology did you use?

---

# 2. Architecture Thinking

Suppose someone says:

> "Build an online shopping system."

Don't immediately start writing:

```java
class ProductService {
}
```

First ask:

### Business questions

```text
Who are the users?
What can they do?
What is the business workflow?
What data exists?
What are the critical operations?
```

### Scale questions

```text
How many users?
Requests/sec?
Peak traffic?
Data size?
Growth rate?
```

### Reliability questions

```text
What happens if DB fails?
What happens if payment fails?
What happens if messaging fails?
```

### Security questions

```text
Who can access what?
How is authentication performed?
How are secrets managed?
```

### Operational questions

```text
How will we deploy?
How will we monitor?
How will we rollback?
```

That is architecture.

---

# 3. Functional vs Non-Functional Requirements

This distinction is extremely important.

## Functional requirement

What the system does.

Example:

```text
Customer can place an order.
Customer can cancel an order.
Customer can view order history.
Admin can update product inventory.
```

## Non-functional requirement

How the system should behave.

Example:

```text
Availability: 99.9%
Latency: < 300 ms for common APIs
Capacity: 5,000 requests/sec
Security: encrypted communication
Scalability: horizontal scaling
Recovery: RTO 30 minutes
```

---

# 4. Architecture Starts With Requirements

Example:

```text
Requirement:
10 million users
5,000 requests/sec
99.9% availability
```

This requirement influences:

```text
Load Balancer
Horizontal Scaling
Database Architecture
Caching
Async Processing
Observability
Disaster Recovery
```

Architecture is therefore a consequence of requirements.

---

# 5. The Jakarta EE Architecture Stack

A mature enterprise application can look like:

```text
                 Clients
                    |
                    v
              Load Balancer
                    |
                    v
             Jakarta REST
                    |
                    v
              CDI Services
                    |
        +-----------+-----------+
        |                       |
        v                       v
      JPA/JDBC                JMS
        |                       |
        v                       v
    Database                Consumers
                                |
                                v
                         Other Systems
```

Cross-cutting concerns:

```text
Security
Validation
Transactions
Logging
Metrics
Tracing
Error Handling
Caching
```

---

# 6. Layered Architecture

A common architecture:

```text
Presentation
     ↓
Application
     ↓
Domain
     ↓
Persistence
     ↓
Infrastructure
```

For Jakarta EE:

```text
REST / Servlet
      ↓
Service / CDI / EJB
      ↓
Domain
      ↓
JPA / JDBC
      ↓
Database
```

---

# 7. Responsibility of Each Layer

## REST/Servlet layer

Responsible for:

```text
HTTP
DTO conversion
Input validation
Response construction
Authentication context
```

Should NOT contain:

```text
complex business logic
SQL
transaction orchestration
```

---

## Service layer

Responsible for:

```text
business operations
workflow
transaction boundary
business rules
coordination
```

---

## Persistence layer

Responsible for:

```text
database access
queries
persistence
mapping
```

---

## Domain

Represents:

```text
business concepts
business rules
entities/value objects
```

---

# 8. Dependency Direction

A clean architecture might look like:

```text
REST
 ↓
Application Service
 ↓
Domain
 ↑
Persistence Adapter
```

The key question:

> Which layer knows about which other layer?

Avoid:

```text
Controller → Database
```

Prefer:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

---

# 9. Don't Create a God Service

Bad:

```java
class OrderService {

    public void createOrder() {}

    public void processPayment() {}

    public void sendEmail() {}

    public void generateInvoice() {}

    public void updateInventory() {}

    public void calculateShipping() {}

    public void generateReport() {}
}
```

This becomes difficult to:

* test
* maintain
* deploy
* scale
* understand

Instead:

```text
OrderService
PaymentService
InventoryService
NotificationService
InvoiceService
ShippingService
```

---

# 10. Modular Monolith

Before jumping to microservices, understand modular monolith architecture.

```text
              Application
                  |
      +-----------+-----------+
      |           |           |
      v           v           v
    Order      Payment     Inventory
    Module      Module       Module
      |           |           |
      +-----------+-----------+
                  |
              Database
```

One deployment unit, but strong module boundaries.

This can be an effective intermediate architecture when the system does not require independently deployed services.

---

# 11. Microservices

Eventually:

```text
Order Service
Payment Service
Inventory Service
Notification Service
```

Each service may have:

```text
API
Business Logic
Database
Messaging
Deployment
Scaling
```

Example:

```text
                 API Gateway
                     |
        +------------+------------+
        |            |            |
        v            v            v
     Orders       Payments    Inventory
        |            |            |
       DB           DB           DB
```

---

# 12. Monolith vs Modular Monolith vs Microservices

| Architecture     | Deployment | Data                  | Operational complexity |
| ---------------- | ---------- | --------------------- | ---------------------- |
| Monolith         | One        | Usually shared        | Lower                  |
| Modular monolith | One        | Can be modular        | Moderate               |
| Microservices    | Many       | Usually service-owned | Higher                 |

Do not choose microservices simply because they are popular.

Choose architecture based on:

```text
Team structure
Scale
Domain boundaries
Deployment needs
Failure isolation
Data ownership
Operational maturity
```

---

# 13. Domain-Driven Thinking

Identify business domains.

For an e-commerce system:

```text
Customer
Catalog
Order
Payment
Inventory
Shipping
Notification
```

Potential bounded contexts:

```text
Catalog Context
Order Context
Payment Context
Inventory Context
Shipping Context
```

The goal is to prevent unrelated business concepts from becoming tightly coupled.

---

# 14. Synchronous vs Asynchronous Architecture

Suppose:

```text
POST /orders
```

The order process requires:

```text
Validate order
Reserve inventory
Charge payment
Send notification
Generate invoice
```

Should everything happen synchronously?

Possibly:

```text
Request
 ↓
Order
 ↓
Inventory
 ↓
Payment
 ↓
Invoice
 ↓
Notification
 ↓
Response
```

This can create long latency and many failure dependencies.

---

# 15. Event-Driven Architecture

Alternative:

```text
POST /orders
      |
      v
Create Order
      |
      v
OrderCreated
      |
      +----> Inventory
      |
      +----> Notification
      |
      +----> Analytics
```

Now unrelated work can happen asynchronously.

---

# 16. What Should Be Synchronous?

Usually operations that directly determine the immediate response.

Example:

```text
Create order
Validate request
Check basic business constraints
Create order
```

Potentially asynchronous:

```text
Email
Analytics
Audit processing
Search indexing
Notifications
Non-critical downstream workflows
```

The exact boundary depends on business requirements.

---

# 17. API Gateway

For multiple services:

```text
Clients
   |
   v
API Gateway
   |
   +---- Orders
   |
   +---- Payments
   |
   +---- Inventory
```

Gateway responsibilities can include:

```text
Routing
Authentication
Rate limiting
TLS termination
Request logging
Correlation IDs
API versioning
```

Don't put business logic into the gateway.

---

# 18. REST API Design

Good API design:

```text
GET    /orders
GET    /orders/{id}
POST   /orders
PUT    /orders/{id}
DELETE /orders/{id}
```

Avoid RPC-style designs such as:

```text
POST /createNewOrder
POST /getOrderDetails
POST /deleteOrder
```

unless there is a specific reason.

---

# 19. API Versioning

Suppose:

```text
/api/v1/orders
```

and later:

```text
/api/v2/orders
```

Versioning protects clients from breaking changes.

Alternative approaches include:

```text
URI versioning
Header versioning
Content negotiation
Backward-compatible evolution
```

Prefer compatibility where possible rather than creating unnecessary versions.

---

# 20. DTO Architecture

Don't expose JPA entities directly.

Instead:

```text
Database
   ↓
Entity
   ↓
Service
   ↓
DTO
   ↓
REST API
```

Example:

```java
public record EmployeeResponse(
        Long id,
        String name,
        String department
) {}
```

Benefits:

```text
API stability
security
decoupling
smaller payloads
control over exposed fields
```

---

# 21. Database Architecture

A basic architecture:

```text
Application
    |
    v
Connection Pool
    |
    v
Primary DB
```

At larger scale:

```text
                 Application
                      |
                 Connection Pool
                      |
              +-------+-------+
              |               |
              v               v
           Primary         Replica
           (writes)        (reads)
```

Read/write splitting requires careful consistency considerations.

---

# 22. Database Indexing

Suppose:

```sql
SELECT *
FROM orders
WHERE customer_id = ?;
```

If this query is frequent:

```sql
CREATE INDEX idx_orders_customer
ON orders(customer_id);
```

But indexes have costs:

```text
Storage
Write overhead
Maintenance
```

Indexes should be driven by actual query patterns.

---

# 23. Caching

Architecture:

```text
Application
    |
    v
Cache
    |
    +-- HIT --> Return
    |
    +-- MISS
          |
          v
       Database
```

Caching is useful for frequently accessed, relatively stable data.

Examples:

```text
Product catalog
Configuration
Reference data
Frequently accessed profiles
```

---

# 24. Cache Invalidation

One of the hardest cache problems:

```text
Database
    |
    v
Cache
```

Data changes:

```text
DB = new value
Cache = old value
```

Strategies include:

```text
Cache-aside
Write-through
Write-behind
TTL
Explicit invalidation
Event-based invalidation
```

---

# 25. Messaging Architecture

From Day 21:

```text
Producer
   |
   v
Queue/Topic
   |
   +---- Consumer A
   |
   +---- Consumer B
   |
   +---- Consumer C
```

Use messaging when you need:

```text
asynchronous processing
decoupling
buffering
event distribution
work queues
integration
```

---

# 26. Idempotency

Suppose:

```text
POST /payments
```

The client retries because of a timeout.

You don't want:

```text
Payment #1
Payment #2
```

for the same business request.

Use an idempotency key:

```http
Idempotency-Key: 8f8d7a...
```

Store the processing result:

```text
Idempotency Key
      ↓
Existing result?
   /       \
 yes       no
 |          |
return     process
```

---

# 27. Transaction Boundaries

A transaction should usually represent a meaningful business operation.

Example:

```text
Create Order
    |
    +-- Create Order Record
    +-- Create Order Items
    +-- Reserve Inventory
    |
   COMMIT
```

Avoid unnecessarily long transactions.

---

# 28. Distributed Transactions

Suppose:

```text
Order DB
Payment DB
Inventory DB
```

Trying to make all three one ACID transaction can introduce significant complexity.

Alternatives include:

```text
Saga
Outbox
Idempotency
Compensation
Event-driven architecture
```

---

# 29. Transactional Outbox

Classic problem:

```text
Database transaction
        |
        +--> Update DB
        |
        +--> Publish event
```

What if DB commit succeeds but message publication fails?

You get:

```text
Database = updated
Message = missing
```

Outbox solution:

```text
Transaction
   |
   +--> Business data
   |
   +--> Outbox event
   |
  COMMIT
```

Then:

```text
Outbox
   ↓
Relay
   ↓
Message Broker
```

---

# 30. Saga

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

Payment succeeds but inventory fails.

You may need:

```text
Refund Payment
Cancel Order
```

These are compensating actions.

A Saga coordinates these business steps without requiring one distributed ACID transaction across all services.

---

# 31. Resilience Patterns

Distributed systems fail.

Important patterns:

```text
Timeout
Retry
Circuit Breaker
Bulkhead
Rate Limiting
Backpressure
Fallback
Idempotency
Dead Letter Queue
```

---

# 32. Retry

Bad:

```text
retry immediately forever
```

Better:

```text
Attempt 1
 ↓
wait
 ↓
Attempt 2
 ↓
longer wait
 ↓
Attempt 3
 ↓
DLQ / failure
```

Use bounded retries and appropriate backoff.

---

# 33. Circuit Breaker

Imagine:

```text
Application → Payment Service
```

Payment service is down.

Without protection:

```text
1000 requests
     ↓
1000 failing calls
```

Circuit breaker:

```text
CLOSED
  ↓ failures
OPEN
  ↓
stop calls
  ↓
HALF-OPEN
  ↓
test request
  ↓
CLOSED
```

This protects the rest of the system.

---

# 34. Bulkhead

Separate resources for different workloads.

Example:

```text
Critical API
    ↓
Thread Pool A

Reporting API
    ↓
Thread Pool B
```

If reporting becomes overloaded:

```text
Reporting Pool = exhausted
```

Critical APIs can remain functional.

---

# 35. Rate Limiting

Suppose an API supports:

```text
100 requests/sec
```

A client sends:

```text
10,000 requests/sec
```

Rate limiting prevents uncontrolled load.

Common approaches:

```text
Token Bucket
Leaky Bucket
Fixed Window
Sliding Window
```

---

# 36. Security Architecture

A production enterprise system might look like:

```text
Client
  |
 HTTPS
  |
API Gateway
  |
Authentication
  |
Authorization
  |
REST API
  |
Service
  |
Database
```

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

---

# 37. RBAC

Role-Based Access Control:

```text
USER
ADMIN
MANAGER
AUDITOR
```

Permissions:

```text
USER:
  READ_ORDER

MANAGER:
  READ_ORDER
  UPDATE_ORDER

ADMIN:
  READ_ORDER
  UPDATE_ORDER
  DELETE_ORDER
```

Avoid scattering authorization logic randomly throughout controllers.

---

# 38. Object-Level Authorization

Important security problem:

```text
GET /orders/100
```

User owns order 50.

If the application only checks:

```text
"is authenticated?"
```

then user might access order 100.

Authorization must often check:

```text
Does this user have permission to access THIS resource?
```

This is commonly called object-level authorization.

---

# 39. Observability Architecture

```text
                Application
               /     |      \
              /      |       \
             v       v        v
          Logs    Metrics    Traces
             \       |       /
              \      |      /
               Monitoring
                   |
                 Alerts
```

Important metrics:

```text
Request rate
Error rate
Latency
CPU
Memory
DB pool
JMS queue depth
Cache hit ratio
```

---

# 40. Availability Architecture

Suppose you need high availability.

Don't deploy:

```text
                Client
                  |
                  v
             One Server
                  |
                  v
               One DB
```

A failure can take down the system.

Instead:

```text
                 Load Balancer
                /             \
               v               v
          App Instance 1   App Instance 2
               |               |
               +-------+-------+
                       |
                       v
                  DB Cluster
```

Then think about:

```text
region failure
database failure
network failure
deployment failure
dependency failure
```

---

# 41. Capacity Planning

Suppose:

```text
10,000 requests/sec
```

One instance handles:

```text
500 requests/sec
```

Very roughly:

```text
10,000 / 500 = 20 instances
```

Then consider:

```text
headroom
peak traffic
failure scenarios
resource bottlenecks
autoscaling
```

Don't use simple arithmetic as the final capacity model; validate with load testing and real bottleneck measurements.

---

# 42. Architecture Trade-Offs

This is one of the most important skills for your solutions-architect goal.

There is rarely one universally correct architecture.

Example:

### Synchronous REST

Advantages:

```text
simple
easy to understand
immediate response
```

Tradeoffs:

```text
tight coupling
latency propagation
dependency failures
```

### Messaging

Advantages:

```text
decoupling
asynchronous processing
buffering
```

Tradeoffs:

```text
eventual consistency
more operational complexity
debugging complexity
duplicate messages
```

Architecture means understanding these tradeoffs.

---

# 43. Architecture Decision Record

For important decisions, document:

```text
Decision
Context
Options
Chosen approach
Consequences
```

Example:

```text
Decision:
Use asynchronous messaging for notification.

Context:
Email sending shouldn't delay order creation.

Options:
1. synchronous email
2. JMS
3. external notification service

Decision:
JMS.

Consequence:
Order response is faster, but notification becomes eventually consistent.
```

This is an **ADR**.

---

# 44. Complete E-Commerce Architecture

Let's combine everything.

```text
                         Internet
                            |
                            v
                     Load Balancer
                            |
                            v
                       API Gateway
                            |
       +--------------------+--------------------+
       |                    |                    |
       v                    v                    v
    Catalog              Orders              Users
       |                    |                    |
       v                    v                    v
    Catalog DB          Order DB            User DB
                            |
                            v
                         Outbox
                            |
                            v
                      Message Broker
                     /       |       \
                    /        |        \
                   v         v         v
             Inventory    Payment   Notification
                 |           |           |
                 v           v           v
             Inventory     Payment      Email
                DB          DB         Provider
```

Cross-cutting:

```text
Authentication
Authorization
Validation
Rate Limiting
Caching
Logging
Metrics
Tracing
```

---

# 45. Request Flow

Customer:

```text
POST /orders
```

Flow:

```text
Client
  ↓
Load Balancer
  ↓
API Gateway
  ↓
Authentication
  ↓
Order REST Resource
  ↓
Validation
  ↓
OrderService
  ↓
Transaction
  ↓
Order DB + Outbox
  ↓
Commit
  ↓
HTTP 201
```

Then asynchronously:

```text
Outbox
   ↓
Broker
   ↓
Inventory Consumer
   ↓
Payment Consumer
   ↓
Notification Consumer
```

---

# 46. Failure Flow

Suppose Payment fails.

```text
OrderCreated
     ↓
Payment Consumer
     ↓
Payment failure
     ↓
Retry
     ↓
Retry
     ↓
DLQ
```

Then operations team can inspect:

```text
DLQ
Logs
Metrics
Trace
```

Depending on the business workflow, a compensation process might:

```text
Cancel Order
Release Inventory
```

---

# 47. Complete Enterprise Architecture

Your mental model should now be:

```text
                         CLIENTS
                            |
                            v
                       DNS / CDN
                            |
                            v
                    Load Balancer
                            |
                            v
                      API Gateway
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          REST API       REST API       REST API
             |              |              |
             +--------------+--------------+
                            |
                      CDI / EJB
                            |
                  Application Services
                            |
         +------------------+------------------+
         |                  |                  |
         v                  v                  v
       JPA/JDBC           Cache              JMS
         |                                     |
         v                                     v
      Database                             Consumers
         |
         +---- Replica
         |
         +---- Backup

       Cross-Cutting
       ─────────────
       Security
       Validation
       Transactions
       Logging
       Metrics
       Tracing
       Error Handling
```

---

# 48. System Design Process You Should Follow

For every architecture problem, use this sequence.

## Step 1 — Requirements

```text
Functional
Non-functional
```

## Step 2 — Scale

```text
Users
Requests/sec
Data
Growth
```

## Step 3 — APIs

```text
Endpoints
Requests
Responses
Authentication
```

## Step 4 — Data

```text
Entities
Relationships
Indexes
Consistency
```

## Step 5 — Architecture

```text
Monolith
Modular monolith
Microservices
Event-driven
```

## Step 6 — Communication

```text
REST
Messaging
Events
```

## Step 7 — Reliability

```text
Timeout
Retry
Circuit breaker
Idempotency
DLQ
```

## Step 8 — Security

```text
Authentication
Authorization
Encryption
Secrets
Audit
```

## Step 9 — Scaling

```text
Cache
Load balancing
Horizontal scaling
Database scaling
```

## Step 10 — Operations

```text
Deployment
Monitoring
Alerting
Rollback
DR
```

---

# 49. 30 Architecture Questions for Today

Design these systems using Jakarta EE concepts:

### Easy

1. Employee Management System
2. Library Management System
3. Hospital Appointment System
4. Course Management System
5. Leave Management System
6. Payroll System
7. Inventory System
8. Customer Management System
9. Document Management System
10. Help Desk System

### Medium

11. E-commerce platform
12. Food delivery system
13. Hotel booking system
14. Ticket booking system
15. Payment processing system
16. Notification platform
17. File storage platform
18. Learning management system
19. Banking transaction platform
20. Logistics management system

### Hard

21. Ride-sharing platform
22. Video streaming platform
23. Social media platform
24. Real-time chat platform
25. Distributed payment system
26. Multi-tenant SaaS platform
27. Order processing platform
28. Event ticketing platform
29. Distributed job scheduler
30. Global e-commerce platform

For every problem, produce:

```text
Requirements
↓
Scale estimation
↓
API design
↓
Data model
↓
High-level architecture
↓
Component design
↓
Database
↓
Caching
↓
Messaging
↓
Security
↓
Failure handling
↓
Deployment
↓
Observability
↓
Trade-offs
```

---

# 50. Day 29 Hands-On Challenge

Take your **Employee Management Platform** and create an architecture document containing:

### 1. Requirements

```text
Functional requirements
Non-functional requirements
```

### 2. API design

```text
POST /employees
GET /employees
GET /employees/{id}
PUT /employees/{id}
DELETE /employees/{id}
```

### 3. Architecture

```text
Client
 ↓
Load Balancer
 ↓
Jakarta REST
 ↓
CDI Service
 ↓
JPA
 ↓
Database
```

### 4. Security

```text
Authentication
Authorization
RBAC
HTTPS
```

### 5. Reliability

```text
Timeouts
Transactions
Idempotency
Retries
```

### 6. Deployment

```text
Docker
CI/CD
Rolling deployment
Health checks
```

### 7. Observability

```text
Logs
Metrics
Tracing
Alerts
```

### 8. Failure scenarios

Document what happens when:

```text
Application crashes
Database crashes
Message broker crashes
External API times out
Network fails
New deployment fails
Cache becomes unavailable
```

This exercise is more important than memorizing architecture terminology.

---

# 51. Architect Interview Questions

Be able to answer these verbally:

1. Why would you choose a monolith?
2. When would you introduce modules?
3. When would microservices be justified?
4. How do you identify service boundaries?
5. How do you design for horizontal scalability?
6. How do you prevent a database from becoming a bottleneck?
7. When should you use caching?
8. When should you use messaging?
9. REST vs messaging?
10. How do you guarantee idempotency?
11. How do you handle duplicate events?
12. How do you handle distributed transactions?
13. How does Saga work?
14. What is the Outbox pattern?
15. How do you design authentication?
16. How do you design authorization?
17. How do you handle service failures?
18. How do you design observability?
19. How do you perform zero-downtime deployment?
20. How do you design disaster recovery?
21. How do you choose database technology?
22. How do you handle read-heavy workloads?
23. How do you handle write-heavy workloads?
24. How do you design for 10× traffic growth?
25. How do you perform capacity planning?
26. How do you handle eventual consistency?
27. How do you design backward-compatible APIs?
28. How do you handle schema evolution?
29. How do you prevent cascading failures?
30. How do you explain architectural tradeoffs to a business stakeholder?

---

# 52. Your Architecture Vocabulary

By now, these terms should be familiar:

```text
Layered Architecture
Clean Architecture
Hexagonal Architecture
Modular Monolith
Microservices
DDD
Bounded Context
REST
Event-Driven Architecture
CQRS
Saga
Outbox
Idempotency
Caching
Replication
Sharding
Load Balancing
API Gateway
Circuit Breaker
Bulkhead
Rate Limiting
Backpressure
Observability
Distributed Tracing
RTO
RPO
Horizontal Scaling
Vertical Scaling
Blue-Green Deployment
Canary Deployment
Rolling Deployment
```

You don't need to use every pattern in every system.

The skill is knowing **when not to use one**.

---

# 53. The Most Important Architect Skill

Don't say:

> "We need Kafka."

Instead ask:

```text
Do we need asynchronous processing?
Do we need durable event streams?
Do we need replay?
Do we need multiple consumers?
What ordering guarantees are required?
What delivery semantics are required?
```

Then select the technology.

Likewise:

Don't say:

> "Let's use microservices."

Ask:

```text
Do we need independent deployment?
Do domains have clear boundaries?
Do teams need independent ownership?
Do services need independent scaling?
Can we operate the additional complexity?
```

Technology should follow requirements.

---

# 54. Day 29 Mastery Checklist

### Architecture

* [ ] Functional requirements
* [ ] Non-functional requirements
* [ ] Layered architecture
* [ ] Modular monolith
* [ ] Microservices
* [ ] Domain boundaries
* [ ] Bounded contexts

### API

* [ ] REST
* [ ] DTOs
* [ ] Versioning
* [ ] Pagination
* [ ] Idempotency
* [ ] API Gateway

### Data

* [ ] JPA
* [ ] JDBC
* [ ] Transactions
* [ ] Indexes
* [ ] Replication
* [ ] Caching
* [ ] Consistency

### Distributed Systems

* [ ] Messaging
* [ ] Events
* [ ] Outbox
* [ ] Saga
* [ ] Retry
* [ ] Circuit breaker
* [ ] Bulkhead
* [ ] Rate limiting
* [ ] Backpressure

### Security

* [ ] Authentication
* [ ] Authorization
* [ ] RBAC
* [ ] TLS
* [ ] Secrets
* [ ] Audit

### Operations

* [ ] CI/CD
* [ ] Docker
* [ ] Load balancing
* [ ] Health checks
* [ ] Monitoring
* [ ] Logging
* [ ] Tracing
* [ ] Deployment strategies
* [ ] Rollback
* [ ] Disaster recovery

---

# 55. The Day 29 Mental Model

Memorize this:

```text
                BUSINESS REQUIREMENTS
                         |
                         v
               FUNCTIONAL + NFRs
                         |
                         v
                SYSTEM ARCHITECTURE
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
        APIs           DATA        COMMUNICATION
          |              |              |
        REST           DB/JPA         JMS
        DTO            Cache          Events
          |              |              |
          +--------------+--------------+
                         |
                         v
                    RELIABILITY
                         |
          +--------------+--------------+
          |              |              |
       Timeout         Retry        Idempotency
       Circuit         Bulkhead     Backpressure
                         |
                         v
                     SECURITY
                         |
             Auth + Authorization
                         |
                         v
                    OPERATIONS
                         |
          +--------------+--------------+
          |              |              |
        Deploy        Observe        Recover
        Docker        Metrics        Backup
        CI/CD         Logs           Rollback
```

## Golden Rule

> **An architect doesn't start with technology. An architect starts with requirements, constraints, trade-offs, and failure scenarios—and then chooses technology that satisfies them.**

---

# Day 29 Final Challenge

Take **one system**—preferably your Employee Management System—and explain it on a whiteboard in this exact order:

```text
1. Requirements
2. Scale
3. API design
4. Data model
5. Architecture
6. Component responsibilities
7. Transactions
8. Cache
9. Messaging
10. Security
11. Failure handling
12. Scaling
13. Deployment
14. Observability
15. Disaster recovery
16. Trade-offs
```

If you can do that clearly, you're no longer merely learning Jakarta EE APIs—you are beginning to use Jakarta EE as an **enterprise architecture platform**.

**Tomorrow, Day 30 (November 1, 2026), is the capstone:** build a production-style **E-commerce / Order Management Platform** that combines the major concepts from all 30 days into one coherent Jakarta EE system.
