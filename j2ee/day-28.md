# Day 28 — Production Deployment & DevOps for J2EE/Jakarta EE

**Date:** October 30, 2026
**Goal:** Learn how to take a Jakarta EE application from source code → build → package → container → deployment → monitoring → scaling → rollback.

At this point, you've learned Servlets, JSP, MVC, JDBC, JPA, REST, CDI, EJB, JMS, JTA, validation, application servers, JNDI, and testing.

Today you learn how to **run all of that reliably in production**.

---

# 1. What You Should Be Able to Do After Day 28

By the end of today, you should understand:

```text
Developer
   ↓
Git
   ↓
Build
   ↓
Test
   ↓
Package
   ↓
WAR / EAR
   ↓
Container / Application Server
   ↓
Reverse Proxy / Load Balancer
   ↓
Production
   ↓
Logs + Metrics + Traces
   ↓
Alerts
   ↓
Rollback / Recovery
```

You should be able to explain:

* How a Jakarta EE application is packaged
* WAR vs EAR vs JAR
* Maven lifecycle
* How to deploy a WAR
* How to containerize a Jakarta EE application
* How configuration changes between environments
* How secrets should be managed
* How HTTPS works in production
* How load balancing works
* How to scale application servers
* Rolling vs blue-green vs canary deployment
* Health checks
* Graceful shutdown
* CI/CD
* Logs, metrics and traces
* Production troubleshooting
* Backup and disaster recovery

---

# 2. Production Architecture

A typical production architecture could look like:

```text
                  Internet
                     |
                     v
              +-------------+
              | DNS         |
              +-------------+
                     |
                     v
              +-------------+
              | Load        |
              | Balancer    |
              +-------------+
                 /       \
                /         \
               v           v
        +-----------+ +-----------+
        | Jakarta   | | Jakarta   |
        | EE App #1 | | EE App #2 |
        +-----------+ +-----------+
             |             |
             +------+------+
                    |
          +---------+---------+
          |                   |
          v                   v
     +---------+         +---------+
     | Database|         | Redis / |
     |         |         | Cache   |
     +---------+         +---------+
          |
          v
     +-------------+
     | Backup      |
     +-------------+

        Observability
             |
      +------+------+ 
      |      |      |
    Logs  Metrics  Traces
```

The important idea is:

> Production deployment is not just "start the server."

It is about **availability, security, scalability, observability and recovery**.

---

# 3. Build vs Deployment

Understand the difference.

### Build

Converts source code into a deployable artifact.

```text
Java source
   ↓
Compile
   ↓
Test
   ↓
Package
   ↓
WAR
```

### Deployment

Takes the artifact and puts it into an environment.

```text
employee.war
     ↓
Application Server
     ↓
Deployment
     ↓
Running application
```

---

# 4. Maven Lifecycle

For enterprise Java, Maven is extremely important.

Important lifecycle phases:

```text
validate
compile
test
package
verify
install
deploy
```

Typical command:

```bash
mvn clean package
```

This roughly performs:

```text
clean
 ↓
compile
 ↓
test
 ↓
package
```

Result:

```text
target/
    employee-app.war
```

---

# 5. Important Maven Commands

### Clean

```bash
mvn clean
```

Deletes previous build output.

---

### Compile

```bash
mvn compile
```

Compiles source code.

---

### Test

```bash
mvn test
```

Runs tests.

---

### Package

```bash
mvn package
```

Creates WAR/JAR.

---

### Install

```bash
mvn install
```

Installs artifact into local Maven repository.

Usually:

```text
~/.m2/repository
```

---

### Skip tests

```bash
mvn package -DskipTests
```

Be careful with this in production pipelines.

Skipping tests should be an explicit decision, not the default.

---

# 6. WAR Packaging

Traditional Jakarta EE web applications are commonly packaged as:

```text
employee.war
```

Typical structure:

```text
employee.war

WEB-INF/
    classes/
    lib/
    web.xml

META-INF/

index.jsp
```

The application server deploys the WAR.

---

# 7. WAR vs JAR vs EAR

| Package | Typical purpose                                    |
| ------- | -------------------------------------------------- |
| JAR     | Java library/application                           |
| WAR     | Web application                                    |
| EAR     | Enterprise application containing multiple modules |

Example:

```text
company.ear
│
├── employee-web.war
├── employee-services.jar
└── employee-ejb.jar
```

EAR is especially associated with traditional enterprise application deployments.

---

# 8. Application Server Deployment

For example, with a server such as WildFly:

```text
employee.war
       ↓
WildFly
       ↓
Deployment
       ↓
Employee Application
```

Deployment can be performed through:

* Management console
* CLI
* Deployment directory
* Automated CI/CD pipeline

---

# 9. Application Configuration

Never hard-code production configuration.

Bad:

```java
String url =
    "jdbc:postgresql://prod-db:5432/employee";

String password = "MyProductionPassword";
```

Instead:

```text
Application
     |
     +-- database URL
     +-- username
     +-- credentials
     +-- external service URL
     +-- feature flags
```

should come from environment-specific configuration.

---

# 10. Configuration by Environment

You typically have:

```text
Development
Testing
Staging
Production
```

Example:

### Development

```text
DB_HOST=localhost
DB_PORT=5432
DB_NAME=employee_dev
```

### Production

```text
DB_HOST=prod-db
DB_PORT=5432
DB_NAME=employee
```

The application code should remain unchanged.

---

# 11. Twelve-Factor Configuration Principle

A useful production principle is:

> Configuration should be externalized from the application artifact.

For example:

```text
employee.war
```

should not contain:

```text
production-password
```

Instead:

```text
WAR
 +
Environment configuration
 +
JNDI resources
 +
Secrets management
```

---

# 12. Secrets Management

Sensitive information includes:

```text
Database passwords
API keys
Private keys
JWT signing secrets
OAuth client secrets
Cloud credentials
```

Don't store them in:

```text
Git
source code
Dockerfile
logs
```

Instead use appropriate secret-management facilities.

Conceptually:

```text
Application
     |
     v
Secret Manager
     |
     v
Database credentials
```

---

# 13. JNDI in Production

You learned JNDI on Day 26.

Production architecture often looks like:

```text
Application
     |
     | lookup
     v
java:/jdbc/EmployeeDS
     |
     v
Connection Pool
     |
     v
Database
```

The application doesn't need to know:

```text
host
port
username
password
pool configuration
```

directly.

---

# 14. Dockerizing Jakarta EE

One of the most important DevOps skills.

Conceptually:

```text
Jakarta EE Application
        +
Application Server
        ↓
      Docker
        ↓
    Container
```

Example Dockerfile concept:

```dockerfile
FROM <jakarta-ee-runtime>

COPY target/employee.war /deployments/

EXPOSE 8080
```

The exact image and deployment directory depend on the chosen application server.

---

# 15. Why Docker?

Without containers:

```text
Developer machine
      ↓
Different Java
      ↓
Different server
      ↓
Different libraries
      ↓
"It works on my machine"
```

With containers:

```text
Application
+
Runtime
+
Configuration contract
+
Dependencies
        ↓
Container
```

This makes environments more reproducible.

---

# 16. Container Architecture

```text
Docker Host
│
├── Jakarta EE Container #1
│
├── Jakarta EE Container #2
│
├── Monitoring Agent
│
└── Log Collector
```

Then:

```text
Load Balancer
       |
   +---+---+
   |       |
   v       v
App #1   App #2
```

---

# 17. Stateless Applications

For horizontal scaling, prefer stateless application instances.

Instead of:

```text
User
 ↓
Server #1
 ↓
Session stored only in memory
```

you want:

```text
             Load Balancer
              /         \
             v           v
          Server #1    Server #2
              \           /
               \         /
                v       v
              Shared State
```

Possible shared state:

```text
Database
Redis
Distributed session store
```

or use stateless authentication where appropriate.

---

# 18. Session Replication

Traditional session architecture:

```text
Client
  |
  v
Server A
  |
Session A
```

If Server A crashes:

```text
Client
  |
  v
Server B
  |
Session missing
```

Possible solutions:

### Sticky sessions

```text
User A → Server A
User B → Server B
```

Easy but introduces operational tradeoffs.

### Session replication

```text
Server A ←→ Server B
```

Session state is replicated.

### External session store

```text
Server A ─┐
           ├── Redis
Server B ─┘
```

For modern scalable architectures, minimizing server-side session state is often desirable.

---

# 19. Reverse Proxy

A reverse proxy sits in front of application servers.

```text
Internet
   |
   v
Reverse Proxy
   |
   v
Jakarta EE
```

Responsibilities can include:

* TLS termination
* routing
* load balancing
* compression
* security headers
* request limits
* static content
* access logging

---

# 20. HTTPS/TLS

Production APIs should use HTTPS.

```text
Client
   |
 HTTPS
   |
   v
Load Balancer
   |
 HTTP/internal HTTPS
   |
   v
Application
```

TLS provides:

```text
Encryption
+
Integrity
+
Server authentication
```

Never transmit credentials or bearer tokens over plain HTTP in production.

---

# 21. TLS Termination

A common architecture:

```text
Client
   |
 HTTPS
   |
   v
Load Balancer
   |
   | internal network
   v
Jakarta EE
```

The load balancer handles the public TLS certificate.

Another model:

```text
Client
  |
 HTTPS
  |
Load Balancer
  |
 HTTPS
  |
Application
```

The second model provides encryption across the internal hop as well.

---

# 22. Health Checks

Production systems need health endpoints.

Example:

```text
GET /health
```

Possible result:

```json
{
  "status": "UP"
}
```

But distinguish:

### Liveness

"Is the process alive?"

### Readiness

"Can this instance receive traffic?"

This distinction is extremely important.

---

# 23. Liveness vs Readiness

Suppose:

```text
Application running
Database unavailable
```

Liveness:

```text
UP
```

Readiness:

```text
NOT_READY
```

The load balancer should stop sending traffic to an instance that isn't ready.

---

# 24. Graceful Shutdown

Bad shutdown:

```text
SIGTERM
   ↓
Immediately kill process
```

Requests may be interrupted.

Better:

```text
Shutdown signal
      ↓
Stop accepting new traffic
      ↓
Finish existing requests
      ↓
Close resources
      ↓
Shutdown application
```

This becomes especially important during rolling deployments.

---

# 25. Deployment Strategies

Three important strategies:

```text
Rolling
Blue-Green
Canary
```

---

# 26. Rolling Deployment

Suppose:

```text
A A A A
```

Deploy version B gradually:

```text
B A A A
B B A A
B B B A
B B B B
```

Advantages:

* lower infrastructure overhead
* gradual replacement

Challenges:

* two versions temporarily coexist
* backward compatibility becomes important

---

# 27. Blue-Green Deployment

Two environments:

```text
BLUE = Current
GREEN = New
```

Initially:

```text
Load Balancer
      |
      v
    BLUE
```

Deploy new version:

```text
Load Balancer
      |
      v
   GREEN
```

After validation:

```text
Load Balancer
      |
      v
   GREEN
```

Rollback can switch traffic back to BLUE.

---

# 28. Canary Deployment

Traffic is gradually shifted.

```text
Version A → 95%
Version B → 5%
```

Then:

```text
A → 75%
B → 25%
```

Then:

```text
A → 50%
B → 50%
```

Eventually:

```text
B → 100%
```

The important engineering requirement is strong monitoring during the rollout.

---

# 29. Database Migration During Deployment

This is one of the most important production problems.

Suppose version 1 has:

```text
employee.name
```

Version 2 needs:

```text
employee.first_name
employee.last_name
```

Don't immediately remove the old column.

A safer migration strategy can be:

```text
Phase 1:
Add new columns

Phase 2:
Application writes both

Phase 3:
Backfill existing data

Phase 4:
Application reads new columns

Phase 5:
Remove old columns later
```

This is often called an **expand-and-contract** migration strategy.

---

# 30. CI/CD Pipeline

A typical pipeline:

```text
Git Push
   ↓
Build
   ↓
Unit Tests
   ↓
Static Analysis
   ↓
Integration Tests
   ↓
Package WAR
   ↓
Build Container
   ↓
Security Scan
   ↓
Deploy Staging
   ↓
Smoke Tests
   ↓
Production Deployment
   ↓
Health Checks
   ↓
Monitoring
```

---

# 31. CI vs CD

### Continuous Integration

Frequently integrate code and automatically:

```text
build
+
test
+
validate
```

### Continuous Delivery

Keep software continuously ready for deployment.

### Continuous Deployment

Automatically deploy qualifying changes to production.

---

# 32. Artifact Management

Don't rebuild different binaries for different environments.

Prefer:

```text
Build once
     ↓
employee.war
     ↓
Test
     ↓
Staging
     ↓
Production
```

The artifact should remain identical.

Only environment configuration changes.

---

# 33. Observability

Production applications need:

```text
Logs
Metrics
Traces
```

Think:

```text
                 Observability
                 /     |      \
                /      |       \
             Logs    Metrics   Traces
```

---

# 34. Logging

Useful information:

```text
timestamp
request ID
user/service identity where appropriate
HTTP method
endpoint
status
duration
error type
```

Example:

```text
2026-10-30T10:15:23
requestId=abc123
method=POST
path=/api/employees
status=201
duration=84ms
```

Don't log:

```text
password
credit-card number
access token
private key
sensitive personal data
```

unless there is a specific, controlled requirement and appropriate protection.

---

# 35. Correlation ID

Suppose:

```text
Client
  |
  | requestId=abc123
  v
API
  |
  v
Order Service
  |
  v
Payment Service
```

Every component logs:

```text
requestId=abc123
```

Then you can reconstruct the request path.

This becomes extremely valuable in distributed systems.

---

# 36. Metrics

Important application metrics:

### Traffic

```text
requests/sec
```

### Latency

```text
p50
p95
p99
```

### Errors

```text
4xx/sec
5xx/sec
```

### Resource utilization

```text
CPU
Memory
Threads
Connections
```

### Database

```text
connection pool usage
query latency
deadlocks
```

### Messaging

```text
queue depth
consumer lag
redeliveries
DLQ messages
```

---

# 37. Golden Signals

Remember:

```text
Latency
Traffic
Errors
Saturation
```

These provide a useful high-level view of system health.

---

# 38. Distributed Tracing

Example:

```text
Request
  |
  +-- API
       |
       +-- Employee Service
              |
              +-- Database
              |
              +-- Messaging
```

Tracing lets you see:

```text
API = 20ms
Service = 40ms
DB = 150ms
```

You can identify where latency is coming from.

---

# 39. Production Alerting

Don't alert on everything.

Examples:

```text
5xx rate > threshold
p99 latency > threshold
DB connection pool exhausted
Disk almost full
JVM memory pressure
Application unavailable
Queue backlog increasing
DLQ growing
```

Alerts should be actionable.

---

# 40. JVM Production Monitoring

A Java/Jakarta EE application runs inside a JVM.

Important JVM concepts:

```text
Heap
Young generation
Old generation
Metaspace
Threads
GC
CPU
```

Monitor:

```text
heap usage
GC pause time
thread count
CPU
allocation rate
class loading
```

---

# 41. Thread Pool

Application servers use thread pools.

Conceptually:

```text
Incoming Requests
       |
       v
Thread Pool
 ┌─────┬─────┬─────┐
 │ T1  │ T2  │ T3  │
 └─────┴─────┴─────┘
       |
       v
Application
```

If all threads are busy:

```text
Requests
   ↓
Queue
   ↓
Timeout
```

Therefore thread-pool sizing matters.

---

# 42. Connection Pool

Similarly:

```text
Application
     |
     v
Connection Pool
 ┌────┬────┬────┬────┐
 │ C1 │ C2 │ C3 │ C4 │
 └────┴────┴────┴────┘
     |
     v
 Database
```

Too small:

```text
Requests wait
```

Too large:

```text
Database overloaded
```

The correct size depends on workload, database capacity and application behavior.

---

# 43. Backpressure

Suppose:

```text
Incoming = 10,000 req/sec
Application = 2,000 req/sec
```

If you accept everything indefinitely:

```text
Queue grows
     ↓
Memory grows
     ↓
Latency grows
     ↓
Timeouts
     ↓
Failure
```

Instead use:

```text
rate limiting
bounded queues
timeouts
load shedding
backpressure
```

---

# 44. Timeout Everything That Can Block

External calls should generally have explicit timeouts.

Examples:

```text
HTTP timeout
Database timeout
JMS timeout
Connection timeout
Transaction timeout
```

Never assume an external dependency will always respond.

---

# 45. Production Error Handling

A production error should not expose:

```text
SQL query
stack trace
database host
filesystem path
credentials
internal class names
```

Client should receive something like:

```json
{
  "type": "https://api.example.com/errors/internal",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An unexpected error occurred",
  "traceId": "abc123"
}
```

The detailed exception belongs in controlled server-side logs.

---

# 46. Backup Strategy

Production database:

```text
Primary DB
    |
    +---- Backup
    |
    +---- Replica
```

Think about:

```text
Backup frequency
Retention
Encryption
Restore testing
Point-in-time recovery
Cross-region recovery
```

A backup that has never been restored/tested should not automatically be considered reliable.

---

# 47. Disaster Recovery

Important terms:

### RTO

Recovery Time Objective:

> How quickly must the service be restored?

### RPO

Recovery Point Objective:

> How much data loss is acceptable?

Example:

```text
RTO = 30 minutes
RPO = 5 minutes
```

means the system is designed around restoring within 30 minutes and potentially losing no more than roughly 5 minutes of data, depending on the actual recovery architecture.

---

# 48. Horizontal vs Vertical Scaling

### Vertical

Increase server capacity:

```text
4 CPU → 16 CPU
8 GB → 32 GB
```

### Horizontal

Add instances:

```text
1 instance
   ↓
3 instances
   ↓
10 instances
```

For web applications, horizontal scaling is often central to high availability.

---

# 49. Production Architecture

Put everything together:

```text
                       Internet
                           |
                           v
                         DNS
                           |
                           v
                    Load Balancer
                           |
             +-------------+-------------+
             |                           |
             v                           v
       Jakarta EE #1               Jakarta EE #2
             |                           |
             +-------------+-------------+
                           |
              +------------+------------+
              |                         |
              v                         v
          Database                 Message Broker
              |                         |
              v                         v
           Replica                  Consumers
              
              Observability
             /      |       \
          Logs    Metrics    Traces
```

---

# 50. Production Deployment Example

Suppose you have:

```text
employee.war
```

Deployment pipeline:

```text
Developer
   |
   v
Git
   |
   v
CI
   |
   +--> Compile
   |
   +--> Unit Tests
   |
   +--> Integration Tests
   |
   +--> Security Scan
   |
   v
employee.war
   |
   v
Container Image
   |
   v
Registry
   |
   v
Staging
   |
   v
Smoke Tests
   |
   v
Production
```

---

# 51. Smoke Testing

After deployment:

```text
GET /health
GET /api/employees
POST /api/employees
GET /api/employees/{id}
```

Verify:

```text
HTTP status
database connectivity
authentication
authorization
messaging
basic business flow
```

---

# 52. Rollback

Always have a rollback strategy.

Example:

```text
Version 1.4
     ↓
Version 1.5
     ↓
Errors increase
     ↓
Rollback
     ↓
Version 1.4
```

But rollback isn't always trivial if the database schema changed.

Therefore:

> Application rollback and database rollback are separate concerns.

This is why backward-compatible database migrations are important.

---

# 53. Production Troubleshooting Framework

When users report:

> "The application is slow."

Don't immediately restart it.

Follow:

```text
1. Is traffic higher?
2. Are errors increasing?
3. Is latency increasing?
4. CPU?
5. Memory?
6. GC?
7. Thread pool?
8. DB pool?
9. Database latency?
10. External API latency?
11. JMS backlog?
12. Recent deployment?
```

---

# 54. Example: High API Latency

Suppose:

```text
p99 = 5 seconds
```

Investigate:

```text
API
 ↓
Service
 ↓
DB
```

Metrics show:

```text
API = 5 sec
DB query = 4.7 sec
```

Then investigate:

```text
SQL
indexes
locks
connection pool
execution plan
database CPU
```

Don't blindly increase application threads.

---

# 55. Example: Database Pool Exhaustion

Symptoms:

```text
Requests timing out
```

Metrics:

```text
Active connections = max
Idle connections = 0
```

Possible causes:

```text
slow queries
connection leaks
long transactions
pool too small
database overloaded
```

Correct troubleshooting requires identifying the cause rather than simply increasing the pool.

---

# 56. Example: Memory Problem

Symptoms:

```text
High heap
Long GC pauses
OutOfMemoryError
```

Investigate:

```text
heap dump
allocation rate
object retention
cache size
thread count
request payloads
memory leaks
```

Don't blindly increase heap size.

---

# 57. Security Checklist

Before production:

```text
✓ HTTPS
✓ Secure cookies
✓ HttpOnly cookies
✓ CSRF protection where applicable
✓ CORS configured correctly
✓ Authentication
✓ Authorization
✓ Input validation
✓ Output encoding
✓ Rate limiting
✓ Secrets management
✓ Dependency scanning
✓ Security headers
✓ Audit logging
✓ No sensitive data in logs
```

---

# 58. Deployment Checklist

Before production:

### Application

```text
✓ Tests pass
✓ No debug mode
✓ Configuration externalized
✓ Error handling configured
✓ Logging configured
```

### Database

```text
✓ Migration tested
✓ Indexes checked
✓ Backup configured
✓ Connection pool configured
```

### Security

```text
✓ HTTPS
✓ Authentication
✓ Authorization
✓ Secrets protected
```

### Infrastructure

```text
✓ Health checks
✓ Load balancing
✓ Monitoring
✓ Alerting
✓ Capacity planning
```

### Recovery

```text
✓ Rollback plan
✓ Backup
✓ Restore procedure
✓ Disaster recovery plan
```

---

# 59. Hands-On Project

## Production Employee Management Platform

Take your previous Employee Management System and deploy it like a real enterprise application.

### Architecture

```text
Browser / REST Client
        |
        v
Load Balancer
        |
        +----------------+
        |                |
        v                v
    App Server 1     App Server 2
        |                |
        +-------+--------+
                |
                v
             JNDI DS
                |
                v
            PostgreSQL
                |
                v
           Backup/Replica
```

---

## Project Requirements

### Application

Implement:

```text
POST   /api/employees
GET    /api/employees
GET    /api/employees/{id}
PUT    /api/employees/{id}
DELETE /api/employees/{id}
```

### Production features

Add:

```text
Authentication
Authorization
Validation
Transactions
JPA
JNDI DataSource
Connection Pool
Logging
Correlation ID
Health Check
Global Error Handling
```

---

# 60. Docker Deployment

Create:

```text
Dockerfile
docker-compose.yml
```

Conceptually:

```text
docker-compose
│
├── jakarta-ee-app-1
├── jakarta-ee-app-2
└── database
```

Then simulate:

```text
Load Balancer
      |
 +----+----+
 |         |
App1      App2
 |         |
 +----+----+
      |
   Database
```

---

# 61. CI/CD Exercise

Create a pipeline:

```text
Push code
   ↓
Build
   ↓
Unit tests
   ↓
Integration tests
   ↓
Package WAR
   ↓
Build image
   ↓
Deploy staging
   ↓
Smoke test
```

Then simulate:

```text
Production deployment
```

---

# 62. Deployment Failure Exercise

Intentionally deploy an application with:

```text
invalid DB configuration
```

Observe:

```text
Application startup
       ↓
DataSource failure
       ↓
Health check
       ↓
Not Ready
```

Then fix configuration and redeploy.

This is much more valuable than simply reading about deployment.

---

# 63. Day 28 Coding/Practical Exercises

Complete these:

1. Create a Maven WAR project.
2. Build it using `mvn clean package`.
3. Deploy WAR to an application server.
4. Configure a JNDI DataSource.
5. Configure a connection pool.
6. Externalize database configuration.
7. Dockerize the application.
8. Run two application instances.
9. Put a load balancer/reverse proxy in front.
10. Add health endpoints.
11. Implement graceful shutdown.
12. Add correlation IDs.
13. Add structured logging.
14. Add application metrics.
15. Configure HTTPS in a test environment.
16. Create a CI pipeline.
17. Create a staging environment.
18. Implement a smoke-test script.
19. Perform a rolling deployment.
20. Simulate a failed deployment and rollback.
21. Perform a database migration.
22. Test backup and restore.
23. Simulate database unavailability.
24. Simulate application instance failure.
25. Perform a basic load test.

---

# 64. Interview Questions

### Beginner

1. What is a WAR file?
2. What is an EAR?
3. What is Maven?
4. What is a Maven lifecycle?
5. What happens during `mvn package`?
6. What is an application server?
7. What is Docker?
8. Why use containers?
9. What is a reverse proxy?
10. What is a load balancer?

### Intermediate

11. What is JNDI?
12. Why externalize configuration?
13. What is connection pooling?
14. What is horizontal scaling?
15. What is vertical scaling?
16. What is session replication?
17. What are health checks?
18. What is graceful shutdown?
19. What is rolling deployment?
20. What is blue-green deployment?
21. What is canary deployment?
22. What is CI/CD?
23. What is artifact immutability?
24. What is TLS termination?
25. What is a readiness check?

### Advanced

26. How would you deploy a Jakarta EE application across multiple instances?
27. How do you make a Jakarta EE application horizontally scalable?
28. How do you handle sessions across multiple instances?
29. How would you design zero-downtime deployment?
30. How would you handle database migrations during deployment?
31. How would you investigate high API latency?
32. How would you investigate connection pool exhaustion?
33. How would you investigate high JVM memory usage?
34. How would you design production observability?
35. How do you design rollback?
36. Why should deployments be backward compatible?
37. How do you prevent secrets from entering Git?
38. How would you design disaster recovery?
39. What are RTO and RPO?
40. How do you scale a JMS consumer?
41. How do you monitor DLQ growth?
42. How do you handle external service timeouts?
43. How do you design application health checks?
44. What happens when one application instance crashes?
45. How do you perform capacity planning?

---

# 65. Architect-Level Scenario

### Requirement

You have:

```text
10 million users
```

The Employee/Business API receives:

```text
5,000 requests/sec
```

You need:

```text
99.9% availability
horizontal scaling
zero/minimal downtime deployment
database protection
observability
automatic recovery
```

Design:

```text
                    Internet
                       |
                       v
                     DNS
                       |
                       v
                Load Balancer
                 /          \
                /            \
               v              v
          Jakarta EE       Jakarta EE
          Instance 1       Instance 2
               |              |
               +------+-------+
                      |
                Connection Pool
                      |
                      v
                  Database
                  /      \
             Primary    Replica
```

Add:

```text
Health Checks
Metrics
Logs
Tracing
Alerts
Backups
CI/CD
Rolling/Blue-Green deployment
Rate limiting
Caching
Message broker
```

Now ask yourself:

> Where does each failure occur, and what happens next?

That question is the beginning of architect-level thinking.

---

# 66. The Production Failure Mindset

For every component, ask:

### Application

```text
What if it crashes?
```

### Database

```text
What if it becomes unavailable?
```

### Network

```text
What if packets are delayed?
```

### Message broker

```text
What if messages cannot be consumed?
```

### Load balancer

```text
What if an instance becomes unhealthy?
```

### Deployment

```text
What if the new version is broken?
```

### Configuration

```text
What if the production configuration is invalid?
```

### External API

```text
What if it takes 30 seconds to respond?
```

This is the transition from **developer thinking → production engineering → architect thinking**.

---

# 67. Day 28 Mastery Checklist

You should be able to explain without notes:

### Build

* [ ] Maven
* [ ] Maven lifecycle
* [ ] WAR
* [ ] JAR
* [ ] EAR

### Deployment

* [ ] Application server deployment
* [ ] Docker
* [ ] Containerization
* [ ] Reverse proxy
* [ ] Load balancer

### Configuration

* [ ] Environment configuration
* [ ] JNDI
* [ ] DataSource
* [ ] Secrets
* [ ] Configuration separation

### Availability

* [ ] Horizontal scaling
* [ ] Session management
* [ ] Health checks
* [ ] Readiness
* [ ] Liveness
* [ ] Graceful shutdown

### Deployment strategies

* [ ] Rolling
* [ ] Blue-green
* [ ] Canary
* [ ] Rollback
* [ ] Database migration strategy

### Observability

* [ ] Logs
* [ ] Metrics
* [ ] Traces
* [ ] Correlation IDs
* [ ] Alerts

### Reliability

* [ ] Timeouts
* [ ] Backpressure
* [ ] Connection pools
* [ ] Backups
* [ ] RTO
* [ ] RPO
* [ ] Disaster recovery

---

# 68. Day 28 Mental Model

Memorize this:

```text
                 PRODUCTION
                     |
        +------------+------------+
        |            |            |
     Deploy       Operate      Recover
        |            |            |
      CI/CD      Observe       Backup
      Docker     Metrics       Restore
      WAR        Logs          Rollback
      Server     Traces        DR
        |
        v
   Load Balancer
        |
   +----+----+
   |         |
 App #1    App #2
   |         |
   +----+----+
        |
   JNDI DataSource
        |
   Connection Pool
        |
     Database
```

## Golden Rule

> **A production application is not just code that works. It must be deployable, observable, secure, scalable, recoverable, and maintainable.**

### Day 28 outcome

You should now be able to take the Jakarta EE application you've been building since Day 1 and answer the complete question:

**“How do I build it, package it, deploy it, scale it, monitor it, secure it, troubleshoot it, and recover it in production?”**

**Next: Day 29 — Jakarta EE Architecture & System Design**, where you'll combine the entire course into architect-level designs: layered architecture, modular monoliths, microservices, scalability, reliability, caching, messaging, databases, API design, security, and trade-off analysis.
