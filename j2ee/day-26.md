# Day 26 — JNDI, Datasources & Resource Management

**Date:** October 28, 2026
**J2EE/Jakarta EE 30-Day Mastery — Day 26**

Today we go deep into one of the most important enterprise-Java concepts:

> **How does a Jakarta EE application obtain infrastructure resources without hard-coding infrastructure details?**

The answer revolves around:

```text
JNDI
  ↓
Resource Naming
  ↓
Resource Injection
  ↓
DataSource
  ↓
Connection Pool
  ↓
Database
```

And the same idea applies to:

```text
JMS ConnectionFactory
JMS Queue
JMS Topic
Mail Session
Transaction resources
Other server-managed resources
```

---

# 1. What You Should Master Today

By the end of Day 26, you should understand:

* JNDI
* Naming contexts
* JNDI names
* `InitialContext`
* Resource lookup
* `@Resource`
* `DataSource`
* Connection pooling
* Pool sizing
* Connection lifecycle
* Connection leaks
* Datasource configuration
* JPA + datasource
* JDBC + datasource
* JTA + datasource
* JMS resources
* `ConnectionFactory`
* Queue/Topic lookup
* Resource injection
* Environment-specific configuration
* Externalized configuration
* Resource security
* Timeouts
* Validation
* Monitoring
* Production resource management

---

# 2. The Big Picture

Think of your application as a business application that needs infrastructure.

Instead of doing this:

```java
String url = "jdbc:postgresql://prod-db:5432/company";
String username = "admin";
String password = "secret";
```

you want:

```java
@Resource(lookup = "java:/jdbc/EmployeeDS")
private DataSource dataSource;
```

Then:

```text
Application
     |
     | "java:/jdbc/EmployeeDS"
     ↓
JNDI
     |
     ↓
DataSource
     |
     ↓
Connection Pool
     |
     ↓
Database
```

Your application doesn't need to know the physical database details.

---

# 3. What Is JNDI?

**JNDI = Java Naming and Directory Interface.**

It provides an API for locating objects/resources using names.

Think of it like a directory:

```text
JNDI
│
├── jdbc/
│   └── EmployeeDS
│
├── jms/
│   ├── OrderQueue
│   └── OrderTopic
│
└── mail/
    └── MailSession
```

The application asks:

```text
"Give me java:/jdbc/EmployeeDS"
```

The runtime returns:

```text
DataSource
```

---

# 4. Why JNDI Exists

Without JNDI:

```text
Application
     |
     +-- DB URL
     +-- DB username
     +-- DB password
     +-- JMS broker
     +-- queue name
     +-- mail server
```

This tightly couples the application to infrastructure.

With JNDI:

```text
Application
     |
     ↓
JNDI
     |
     ├── DataSource
     ├── JMS ConnectionFactory
     ├── Queue
     └── Mail Session
```

Infrastructure can change independently.

---

# 5. The Core Principle

Remember:

> **Application code should depend on logical resource names; infrastructure configuration should determine what those names point to.**

For example:

```text
Application:

java:/jdbc/EmployeeDS
```

Development:

```text
java:/jdbc/EmployeeDS
       ↓
localhost database
```

Production:

```text
java:/jdbc/EmployeeDS
       ↓
production database cluster
```

The Java code remains unchanged.

---

# 6. JNDI Naming Concepts

Important terms:

### Naming Context

A namespace containing bindings.

```text
Context
 ├── jdbc
 ├── jms
 └── mail
```

### Binding

Mapping a name to an object.

```text
"jdbc/EmployeeDS"
        ↓
DataSource
```

### Lookup

Retrieving an object by name.

```java
context.lookup("java:/jdbc/EmployeeDS");
```

---

# 7. Manual JNDI Lookup

Example:

```java
import javax.naming.InitialContext;
import javax.sql.DataSource;

InitialContext context = new InitialContext();

DataSource dataSource =
        (DataSource) context.lookup(
                "java:/jdbc/EmployeeDS"
        );
```

Modern Jakarta EE applications use the corresponding `jakarta.*` APIs where applicable.

---

# 8. Resource Injection

Usually, you don't need manual lookup.

You can use:

```java
@Resource(lookup = "java:/jdbc/EmployeeDS")
private DataSource dataSource;
```

The container performs the lookup and injection.

Architecture:

```text
Application Startup
       ↓
Container discovers @Resource
       ↓
JNDI lookup
       ↓
DataSource obtained
       ↓
Dependency injected
```

---

# 9. Injection vs Manual Lookup

### Manual lookup

```java
InitialContext context = new InitialContext();

DataSource ds =
    (DataSource) context.lookup(
        "java:/jdbc/EmployeeDS"
    );
```

### Injection

```java
@Resource(lookup = "java:/jdbc/EmployeeDS")
private DataSource ds;
```

In managed Jakarta EE applications, injection is generally cleaner.

---

# 10. What Is a DataSource?

`DataSource` is the standard abstraction for obtaining database connections.

```java
DataSource dataSource;
```

Then:

```java
Connection connection =
        dataSource.getConnection();
```

The important part:

```text
Application
    ↓
DataSource
    ↓
Connection
```

The application doesn't need to know how the connection is created.

---

# 11. DataSource vs DriverManager

Basic JDBC:

```java
Connection connection =
    DriverManager.getConnection(
        url,
        username,
        password
    );
```

Enterprise approach:

```java
Connection connection =
    dataSource.getConnection();
```

Why?

Because the application server can provide:

* pooling
* validation
* timeout configuration
* monitoring
* transaction integration
* centralized configuration

---

# 12. Connection Pool

A connection pool maintains reusable database connections.

Without pooling:

```text
Request
 ↓
Create connection
 ↓
Execute query
 ↓
Close connection
```

With pooling:

```text
Request
 ↓
Borrow connection
 ↓
Execute query
 ↓
Return connection
```

Pool:

```text
Connection Pool
│
├── Connection 1
├── Connection 2
├── Connection 3
├── Connection 4
└── Connection 5
```

---

# 13. Why Connection Creation Is Expensive

Creating a DB connection may involve:

```text
Application
   ↓
Network connection
   ↓
Database authentication
   ↓
Session initialization
   ↓
Database resources
```

Doing this for every request creates unnecessary overhead.

Pooling amortizes that cost.

---

# 14. Connection Lifecycle

The correct pattern:

```java
try (Connection connection = dataSource.getConnection();
     PreparedStatement statement =
         connection.prepareStatement(
             "SELECT id, name FROM employee")) {

    try (ResultSet resultSet = statement.executeQuery()) {

        while (resultSet.next()) {
            // process row
        }
    }
}
```

Important:

```text
getConnection()
      ↓
Use connection
      ↓
close()
      ↓
Return to pool
```

`close()` on a pooled connection generally returns it to the pool rather than physically closing the underlying database connection.

---

# 15. Connection Leak

Bad:

```java
Connection connection =
    dataSource.getConnection();

PreparedStatement ps =
    connection.prepareStatement(sql);

// forgot to close
```

Repeated requests can eventually exhaust the pool.

Example:

```text
Pool size = 20

Request 1 → connection 1 leaked
Request 2 → connection 2 leaked
...
Request 20 → connection 20 leaked

Request 21
    ↓
WAIT
    ↓
Timeout
```

---

# 16. Connection Leak Symptoms

Typical symptoms:

```text
Requests become slow
        ↓
Connection pool usage increases
        ↓
Available connections → 0
        ↓
Requests wait
        ↓
Timeouts
```

Possible errors mention:

```text
connection timeout
pool exhausted
unable to obtain connection
```

---

# 17. Pool Configuration

Common properties:

```text
minimum pool size
maximum pool size
connection timeout
idle timeout
max lifetime
validation
leak detection
```

Example conceptual configuration:

```text
minimum = 10
maximum = 50
connection timeout = 30 seconds
idle timeout = 10 minutes
```

These values are examples, not universal recommendations.

---

# 18. How Do You Choose Pool Size?

Don't simply say:

> More connections = more performance.

That is often wrong.

Suppose:

```text
Application instances = 10
Pool max per instance = 50
```

Potential maximum connections:

```text
10 × 50 = 500
```

If the database supports only 200 useful concurrent connections, you have a problem.

Therefore consider:

```text
Database connection limit
        ↓
Number of application instances
        ↓
Query duration
        ↓
Concurrency
        ↓
CPU
        ↓
Workload
```

---

# 19. Little's Law Connection

A useful performance concept:

```text
L = λ × W
```

Where:

```text
L = average number of concurrent requests
λ = throughput
W = average time
```

Example:

```text
Throughput = 100 requests/sec
Average DB time = 100 ms
```

Then approximately:

```text
100 × 0.1 = 10
```

About 10 concurrent database operations are implied by that workload segment.

This doesn't directly give the final pool size, but it helps reason about concurrency.

---

# 20. Pool Size Is a System-Level Decision

Consider:

```text
3 application instances
100 max connections each
```

Potential DB connections:

```text
3 × 100 = 300
```

Now suppose you scale to:

```text
20 instances
```

You could potentially reach:

```text
20 × 100 = 2000
```

Your application scaling decision has now become a database capacity problem.

This is a crucial architecture lesson.

---

# 21. DataSource + JDBC

Example repository:

```java
public class EmployeeRepository {

    private final DataSource dataSource;

    public EmployeeRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Employee findById(long id)
            throws SQLException {

        String sql = """
            SELECT id, name, email
            FROM employee
            WHERE id = ?
            """;

        try (Connection connection =
                 dataSource.getConnection();
             PreparedStatement ps =
                 connection.prepareStatement(sql)) {

            ps.setLong(1, id);

            try (ResultSet rs =
                     ps.executeQuery()) {

                if (rs.next()) {
                    return new Employee(
                        rs.getLong("id"),
                        rs.getString("name"),
                        rs.getString("email")
                    );
                }

                return null;
            }
        }
    }
}
```

---

# 22. DataSource + JPA

JPA can use a configured datasource.

Conceptually:

```text
JPA
 ↓
Persistence Provider
 ↓
JTA/DataSource
 ↓
Connection Pool
 ↓
Database
```

Your persistence configuration can refer to a datasource resource.

For example, conceptually:

```xml
<jta-data-source>
    java:/jdbc/EmployeeDS
</jta-data-source>
```

---

# 23. JPA + JTA + DataSource

This is an important enterprise stack:

```text
Business Operation
       ↓
JTA Transaction
       ↓
JPA
       ↓
DataSource
       ↓
Connection Pool
       ↓
Database
```

Example:

```text
Create Employee
       |
       +--> Insert employee
       |
       +--> Insert audit record
       |
       +--> Publish transactional work
       |
       ↓
     COMMIT
```

The transaction manager coordinates the transactional resources involved.

---

# 24. Resource-Local vs JTA

Two important persistence models:

### Resource-local

The persistence unit manages its own database transaction.

Conceptually:

```text
Application
   ↓
EntityManager
   ↓
Database transaction
```

### JTA

A transaction manager coordinates the transaction.

```text
Application
   ↓
JTA Transaction
   ↓
EntityManager
   ↓
DataSource
   ↓
Database
```

JTA becomes especially important when multiple transactional resources participate.

---

# 25. Resource Injection for Other Resources

`@Resource` isn't limited to datasources.

Conceptually:

```java
@Resource(lookup = "java:/jdbc/EmployeeDS")
DataSource dataSource;
```

and:

```java
@Resource(lookup = "java:/jms/OrderConnectionFactory")
ConnectionFactory connectionFactory;
```

and:

```java
@Resource(lookup = "java:/jms/OrderQueue")
Queue orderQueue;
```

The common idea:

```text
Logical Name
     ↓
JNDI
     ↓
Managed Resource
```

---

# 26. JMS Resource Architecture

```text
Application
    |
    | lookup
    v
JNDI
    |
    +---- ConnectionFactory
    |
    +---- Queue
    |
    +---- Topic
```

Then:

```text
Producer
   ↓
ConnectionFactory
   ↓
Queue
   ↓
Consumer
```

---

# 27. ConnectionFactory

A JMS `ConnectionFactory` provides the mechanism for creating messaging contexts/connections.

Modern Jakarta Messaging APIs commonly use `JMSContext`.

Conceptually:

```java
@Resource(lookup = "java:/jms/OrderConnectionFactory")
private ConnectionFactory connectionFactory;
```

Then the application can obtain a messaging context according to the API/runtime configuration.

---

# 28. Queue Resource

Example:

```java
@Resource(lookup = "java:/jms/OrderQueue")
private Queue orderQueue;
```

Then:

```text
OrderService
      |
      v
OrderQueue
      |
      v
Inventory Consumer
```

---

# 29. Topic Resource

Topic:

```text
Producer
   |
   v
Topic
 ┌─┼────┐
 ↓ ↓    ↓
A  B    C
```

Multiple consumers can receive the published message according to the subscription model.

---

# 30. JNDI + JMS Example

Conceptually:

```java
@Resource(lookup = "java:/jms/OrderConnectionFactory")
private ConnectionFactory connectionFactory;

@Resource(lookup = "java:/jms/OrderQueue")
private Queue orderQueue;
```

Then:

```java
public void publish(OrderEvent event) {

    try (JMSContext context =
             connectionFactory.createContext()) {

        context.createProducer()
               .send(
                   orderQueue,
                   event.toJson()
               );
    }
}
```

Exact configuration and serialization behavior depends on your Jakarta Messaging provider/runtime.

---

# 31. Resource Naming Strategy

Use meaningful names.

Good:

```text
java:/jdbc/EmployeeDS
java:/jdbc/OrderDS
java:/jms/OrderQueue
java:/jms/NotificationQueue
java:/jms/EventTopic
```

Avoid unclear names:

```text
java:/x
java:/foo
java:/db1
java:/queue1
```

Names should communicate purpose.

---

# 32. Environment Independence

Consider:

```text
Development:
java:/jdbc/EmployeeDS → localhost

QA:
java:/jdbc/EmployeeDS → qa-db

Production:
java:/jdbc/EmployeeDS → production-db
```

Application code:

```java
@Resource(lookup = "java:/jdbc/EmployeeDS")
DataSource dataSource;
```

No code change.

This is one of the most valuable benefits of resource indirection.

---

# 33. Resource Configuration vs Application Configuration

Separate these concepts.

### Application configuration

Things such as:

```text
feature flags
business limits
timeouts
application behavior
```

### Infrastructure resources

Things such as:

```text
database
JMS broker
connection pools
mail server
security providers
```

Application servers are particularly useful for managing infrastructure resources.

---

# 34. Database Credentials

Avoid:

```java
private static final String PASSWORD =
    "prod-password";
```

Instead:

```text
Application
    ↓
Managed DataSource
    ↓
Credentials/configuration
    ↓
Database
```

The application should not unnecessarily handle raw production credentials.

---

# 35. Connection Validation

Connections can become invalid because of:

* database restart
* network interruption
* firewall timeout
* database-side timeout
* infrastructure failures

Pool configuration can include connection validation.

Conceptually:

```text
Borrow connection
      ↓
Validate
      ↓
Healthy?
  /       \
Yes       No
 |         |
Use      Replace
```

---

# 36. Connection Timeout

Imagine all pool connections are occupied.

```text
Pool
│
├── C1 busy
├── C2 busy
├── C3 busy
└── C4 busy
```

New request:

```text
getConnection()
      ↓
WAIT
```

You need a timeout.

Otherwise requests can wait indefinitely.

---

# 37. Database Timeout vs Connection Timeout

Don't confuse them.

### Connection acquisition timeout

How long to wait for a pool connection.

```text
Application → Pool
```

### Query timeout

How long a SQL operation may execute.

```text
Application → Database
```

### Network/socket timeout

How long network communication can wait.

These are different failure modes.

---

# 38. Pool Exhaustion

Architecture:

```text
Traffic increases
      ↓
More concurrent requests
      ↓
More DB connections needed
      ↓
Pool exhausted
      ↓
Requests wait
      ↓
Timeout
      ↓
5xx errors
```

But the solution isn't always:

> Increase pool size.

It may be:

```text
Optimize SQL
Optimize indexes
Reduce transaction duration
Remove unnecessary DB calls
Cache reads
Fix connection leaks
Reduce concurrency
Scale database
```

---

# 39. Long Transactions

Bad:

```text
Begin transaction
      ↓
Call external API
      ↓
Wait 10 seconds
      ↓
More processing
      ↓
Commit
```

This holds transactional resources unnecessarily.

Prefer:

```text
Validate
 ↓
Short DB transaction
 ↓
Commit
 ↓
External interaction
```

where business requirements permit.

---

# 40. Resource Management and Transactions

A good enterprise flow:

```text
Request
  ↓
Service
  ↓
Begin transaction
  ↓
Obtain resource
  ↓
Perform DB work
  ↓
Commit
  ↓
Return resource
```

Avoid:

```text
Request
  ↓
Open DB connection
  ↓
Do lots of unrelated work
  ↓
Call remote API
  ↓
Wait
  ↓
Commit
```

---

# 41. Resource Lifecycle

Think in terms of:

```text
Acquire
   ↓
Use
   ↓
Release
```

For example:

```text
DataSource
   ↓
Connection
   ↓
PreparedStatement
   ↓
ResultSet
```

Close in reverse dependency order:

```text
ResultSet
   ↓
PreparedStatement
   ↓
Connection
```

Try-with-resources handles this safely.

---

# 42. Resource Ownership

A useful architecture question:

> Who owns the resource?

For server-managed datasource:

```text
Application Server
        |
        ↓
Connection Pool
        |
        ↓
Database connections
```

Application:

```text
borrows → uses → returns
```

Application should not attempt to manage the pool itself.

---

# 43. Don't Create Your Own Global Pool

Avoid:

```java
static List<Connection> connections;
```

or building an ad-hoc connection pool inside the application when the runtime already provides one.

This can cause:

* duplicated pooling
* resource exhaustion
* poor lifecycle management
* difficult monitoring
* configuration conflicts

---

# 44. Resource Security

Resources themselves must be protected.

For databases:

```text
Application
   ↓
DataSource
   ↓
Least-privilege DB user
   ↓
Database
```

Don't give an application unnecessary database privileges.

For example, a read-only service might only need:

```text
SELECT
```

rather than:

```text
SELECT
INSERT
UPDATE
DELETE
DROP
ALTER
```

---

# 45. Principle of Least Privilege

Give each application/resource only what it needs.

```text
Employee API
   ↓
employee_db_user
   ↓
Required permissions only
```

This limits the blast radius of security problems.

---

# 46. Monitoring Resources

Production monitoring should include:

### Connection pool

```text
Active connections
Idle connections
Maximum connections
Waiting requests
Connection acquisition time
Timeouts
```

### Database

```text
CPU
Memory
Connections
Query latency
Locks
Slow queries
```

### JMS

```text
Queue depth
Consumer count
Message age
Redelivery
DLQ size
```

---

# 47. Queue Depth as a Signal

Suppose:

```text
Queue depth:
100
500
1000
5000
10000
```

This may indicate:

```text
Producer rate > Consumer processing rate
```

Possible causes:

* consumer too slow
* insufficient consumers
* downstream database slow
* external dependency slow
* poison messages
* insufficient resources

This is why resource monitoring matters.

---

# 48. Backpressure

Suppose:

```text
Producer = 10,000 messages/sec
Consumer = 5,000 messages/sec
```

Queue grows.

```text
10k in
  ↓
Queue
  ↓
5k out
```

Backpressure strategies can include:

* bounded queues
* rate limiting
* consumer scaling
* load shedding
* batching
* prioritization

---

# 49. Resource Failure

Think about:

```text
Database unavailable
```

Then:

```text
DataSource
   ↓
Connection acquisition
   ↓
Failure
```

Your application should:

* fail fast where appropriate
* avoid endless retries
* use sensible timeouts
* expose health state
* log useful information
* avoid cascading failures
* recover when the dependency returns

---

# 50. Retry and Database Resources

Be careful with retry.

Bad:

```text
Database failure
 ↓
retry
 ↓
retry
 ↓
retry
 ↓
retry
```

Across hundreds of application instances this can create a retry storm.

Better:

```text
Failure
 ↓
bounded retry
 ↓
backoff
 ↓
timeout
 ↓
fallback / failure response
```

---

# 51. JNDI Resource Architecture

Memorize this:

```text
                   Application
                       |
              ┌────────┼────────┐
              ↓        ↓        ↓
           DataSource Queue   Topic
              |        |        |
              ↓        ↓        ↓
           JNDI Resource Registry
                       |
                       ↓
                Application Server
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       DB Pool      JMS Broker    Other
```

---

# 52. Example Employee Repository

```java
public class EmployeeRepository {

    private final DataSource dataSource;

    public EmployeeRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void save(Employee employee)
            throws SQLException {

        String sql = """
            INSERT INTO employee(name, email)
            VALUES (?, ?)
            """;

        try (Connection connection =
                 dataSource.getConnection();
             PreparedStatement ps =
                 connection.prepareStatement(sql)) {

            ps.setString(1, employee.getName());
            ps.setString(2, employee.getEmail());

            ps.executeUpdate();
        }
    }
}
```

Notice:

```text
No database URL
No username
No password
No pool implementation
```

The infrastructure supplies those.

---

# 53. Better Layered Architecture

```text
REST Resource
      ↓
Service
      ↓
Repository
      ↓
DataSource
      ↓
Connection Pool
      ↓
Database
```

Resource management should remain below the business layer.

---

# 54. Don't Leak Infrastructure Details

Avoid returning:

```java
Connection
```

from your service API.

Bad:

```java
Connection getConnection();
```

Better:

```java
Employee findById(long id);
```

The repository owns the persistence details.

---

# 55. DataSource Abstraction

Your application can depend on:

```java
javax.sql.DataSource
```

or the relevant standard API available in your Jakarta/Java runtime.

That allows the infrastructure implementation to change without changing business logic.

---

# 56. Resource Injection in CDI

You can combine CDI with resource injection.

Conceptually:

```java
@ApplicationScoped
public class EmployeeRepository {

    @Resource(lookup = "java:/jdbc/EmployeeDS")
    private DataSource dataSource;

}
```

Then:

```text
CDI
 ↓
Creates bean
 ↓
Container injects DataSource
 ↓
Repository ready
```

---

# 57. JNDI vs Dependency Injection

These concepts are related but different.

### JNDI

Provides naming/lookup.

```text
Name → Resource
```

### Dependency Injection

Provides dependencies to objects.

```text
Bean → Dependency
```

They can work together:

```text
@Resource
    ↓
JNDI lookup
    ↓
Resource
    ↓
Injection
```

---

# 58. JNDI vs Environment Variables

They solve somewhat different problems.

Environment variables can provide:

```text
PORT=8080
FEATURE_X=true
```

JNDI can expose managed resources:

```text
java:/jdbc/EmployeeDS
java:/jms/OrderQueue
```

In modern cloud-native systems, environment/configuration services are common; in traditional Jakarta EE deployments, JNDI remains important for container-managed resources.

---

# 59. Production Resource Design

For a production application:

```text
                     Application
                          |
         ┌────────────────┼────────────────┐
         ↓                ↓                ↓
      DataSource      JMS Factory       Security
         |                |                |
      JNDI             JNDI             JNDI/
         |                |             Runtime
         ↓                ↓
    Connection Pool    Broker
         |
         ↓
      Database
```

---

# 60. Day 26 Mini Project

Build:

# Enterprise Resource Management Platform

Requirements:

### Database

```text
Employee
Department
AuditLog
```

### Datasource

```text
java:/jdbc/EmployeeDS
```

### JMS

```text
java:/jms/EmployeeEventQueue
```

### Application

```text
REST API
   ↓
Service
   ↓
Repository
   ↓
DataSource
   ↓
Database
```

Events:

```text
Employee Created
Employee Updated
Employee Deleted
```

Architecture:

```text
Employee API
     |
     v
Employee Service
     |
     +------------+
     |            |
     v            v
Database       JMS Queue
                  |
                  v
           Audit Consumer
```

---

# 61. Project Package Structure

```text
employee-platform/
│
├── controller/
│   └── EmployeeResource.java
│
├── service/
│   └── EmployeeService.java
│
├── repository/
│   └── EmployeeRepository.java
│
├── messaging/
│   ├── EmployeeEventProducer.java
│   └── EmployeeEventConsumer.java
│
├── entity/
│   └── Employee.java
│
├── dto/
│   └── EmployeeDTO.java
│
├── exception/
│   └── EmployeeNotFoundException.java
│
└── config/
    └── ResourceConfiguration.java
```

---

# 62. Project Flow

Create employee:

```text
POST /employees
       ↓
REST Resource
       ↓
EmployeeService
       ↓
Transaction
       ↓
EmployeeRepository
       ↓
DataSource
       ↓
Connection Pool
       ↓
Database
       ↓
EmployeeCreated Event
       ↓
JMS Queue
       ↓
Audit Consumer
```

---

# 63. Day 26 Coding Exercises

### Beginner

1. Explain JNDI.
2. Perform manual JNDI lookup.
3. Inject a datasource.
4. Obtain a connection.
5. Execute a prepared statement.
6. Close resources using try-with-resources.
7. Explain connection pooling.
8. Explain connection leaks.
9. Configure a logical datasource name.
10. Explain JNDI vs DI.

### Intermediate

11. Configure a datasource.
12. Configure pool limits.
13. Build a JDBC repository using `DataSource`.
14. Integrate JPA with a datasource.
15. Configure a JMS queue.
16. Inject a JMS queue.
17. Inject a JMS connection factory.
18. Monitor pool usage.
19. Simulate connection exhaustion.
20. Diagnose a connection leak.

### Advanced

21. Calculate possible DB connections across 10 application instances.
22. Design pool sizing for a high-throughput system.
23. Design database failure handling.
24. Design JMS backpressure.
25. Design resource isolation between applications.
26. Design least-privilege datasource access.
27. Design multi-environment resource configuration.
28. Design a clustered datasource architecture.
29. Design resource monitoring.
30. Design a production resource-management strategy.

---

# 64. Interview Questions

### Q1. What is JNDI?

A Java naming mechanism used to locate resources and objects by logical names.

### Q2. Why is JNDI useful in enterprise applications?

It decouples application code from infrastructure-specific resource configuration.

### Q3. What is a DataSource?

An abstraction for obtaining database connections.

### Q4. Why use DataSource instead of DriverManager?

A managed datasource can provide connection pooling, configuration, monitoring, and transaction integration.

### Q5. What is connection pooling?

Maintaining reusable database connections that applications borrow and return.

### Q6. What happens when you close a pooled connection?

Typically the connection is returned to the pool rather than physically terminating the underlying DB connection.

### Q7. What is connection leakage?

Failure to properly release borrowed connections, eventually exhausting the pool.

### Q8. What causes pool exhaustion?

Possible causes include connection leaks, long queries, long transactions, insufficient pool size, database slowness, or excessive concurrency.

### Q9. Should you always increase the pool size when the pool is exhausted?

No. First determine why the pool is exhausted.

### Q10. What is JNDI lookup?

Retrieving a resource from the naming system using its logical name.

### Q11. What is resource injection?

Allowing the container to inject a configured resource into an application component.

### Q12. What is `@Resource`?

A Jakarta EE annotation commonly used for injecting or declaring container-managed resources.

### Q13. What is connection timeout?

The maximum time an operation may wait to acquire a connection or communicate with the database, depending on which timeout is being configured.

### Q14. Why validate connections?

To detect stale or invalid connections before handing them to application code.

### Q15. What is least privilege?

Giving a resource or identity only the permissions required to perform its function.

---

# 65. Architect-Level Interview Questions

### Scenario 1

You have:

```text
20 application instances
maximum pool size = 50
```

What is the theoretical maximum number of application-held database connections?

```text
20 × 50 = 1000
```

Now ask:

> Can the database safely support 1000 connections?

That's the real architecture question.

---

### Scenario 2

Your pool shows:

```text
Active = 100
Idle = 0
Waiting = 300
```

Don't immediately increase the pool.

Investigate:

```text
Connection leaks
       ↓
Slow SQL
       ↓
Long transactions
       ↓
Database locks
       ↓
Database capacity
       ↓
Unexpected traffic
```

---

### Scenario 3

Database latency increases from:

```text
50 ms → 5 seconds
```

Then:

```text
Connections remain occupied longer
        ↓
Pool utilization rises
        ↓
Requests wait
        ↓
Application latency rises
        ↓
Timeouts
```

A database performance problem can therefore become an application-server resource problem.

---

# 66. Production Monitoring Dashboard

Your dashboard should ideally expose:

```text
APPLICATION
├── Request rate
├── Error rate
├── Latency
└── Active requests

DATASOURCE
├── Active connections
├── Idle connections
├── Maximum pool
├── Waiting threads
├── Acquisition latency
└── Connection timeouts

DATABASE
├── CPU
├── Connections
├── Query latency
├── Locks
└── Slow queries

JMS
├── Queue depth
├── Consumer count
├── Message age
├── Redelivery
└── DLQ messages
```

---

# 67. Common Mistakes

### Mistake 1

Hard-coding database credentials.

### Mistake 2

Creating a new DB connection for every request.

### Mistake 3

Not closing connections.

### Mistake 4

Creating a custom connection pool unnecessarily.

### Mistake 5

Making the pool extremely large.

### Mistake 6

Ignoring database connection limits.

### Mistake 7

Holding connections during external API calls.

### Mistake 8

Ignoring connection acquisition timeouts.

### Mistake 9

Ignoring stale connections.

### Mistake 10

Treating JNDI as merely an old legacy feature.

---

# 68. The Most Important Architecture Lesson

Consider this chain:

```text
HTTP Traffic
     ↓
Application Instances
     ↓
Concurrent Requests
     ↓
Connection Pool
     ↓
Database
```

Every layer constrains the next layer.

For example:

```text
1000 requests/sec
       ↓
500 concurrent requests
       ↓
100 DB connections
       ↓
Database supports 80 efficiently
```

You can't solve every scaling problem by simply adding more application servers.

More application servers can actually increase database pressure.

This is a key **solutions-architect-level insight**.

---

# 69. Day 26 Mastery Checklist

You should now be able to explain:

* [ ] JNDI
* [ ] Naming context
* [ ] Binding
* [ ] Lookup
* [ ] JNDI names
* [ ] `InitialContext`
* [ ] `@Resource`
* [ ] DataSource
* [ ] DriverManager vs DataSource
* [ ] Connection pooling
* [ ] Pool sizing
* [ ] Connection leaks
* [ ] Connection timeout
* [ ] Query timeout
* [ ] Connection validation
* [ ] JPA + DataSource
* [ ] JTA + DataSource
* [ ] JMS resources
* [ ] ConnectionFactory
* [ ] Queue
* [ ] Topic
* [ ] Resource injection
* [ ] Environment-specific resources
* [ ] Resource security
* [ ] Least privilege
* [ ] Resource monitoring
* [ ] Pool exhaustion
* [ ] Backpressure
* [ ] Database capacity
* [ ] Graceful resource shutdown

---

# 70. Final Mental Model

Memorize this:

```text
                         APPLICATION
                              |
             ┌────────────────┼────────────────┐
             ↓                ↓                ↓
         DataSource       JMS Resources      Security
             |                |                |
             └───────┬────────┴────────────────┘
                     ↓
                    JNDI
                     ↓
             APPLICATION SERVER
                     |
        ┌────────────┼────────────┐
        ↓            ↓            ↓
 Connection Pool   JMS Broker   Security
        |
        ↓
    Database
```

And the most important resource lifecycle:

```text
      ACQUIRE
         ↓
        USE
         ↓
      RELEASE
```

For database connections:

```text
DataSource
    ↓
getConnection()
    ↓
Connection
    ↓
PreparedStatement
    ↓
ResultSet
    ↓
close()
    ↓
Connection returned to pool
```

### Golden Rule

> **JNDI provides the naming indirection, the application server manages resources, the DataSource provides database connections, the connection pool manages reuse, and your application should acquire, use, and release resources without owning the underlying infrastructure.**

**Day 27 → Testing Enterprise Applications**

You will bring together:

```text
Unit Testing
Integration Testing
JPA Testing
JDBC Testing
REST Testing
CDI Testing
EJB Testing
JMS Testing
Transaction Testing
Mocking
Test Containers
Test Databases
End-to-End Testing
```

and learn how to test a real Jakarta EE application rather than testing individual Java classes in isolation.
