# Day 25 — Application Servers & Enterprise Deployment

**Date:** October 27, 2026
**J2EE/Jakarta EE 30-Day Mastery — Day 25**

Today you move from **writing Jakarta EE applications** to understanding **where and how enterprise applications actually run in production**.

The core mental model for today:

```text
                    Application Server
 ┌──────────────────────────────────────────────────────┐
 │                                                      │
 │  HTTP ──> Web Container ──> Servlet/JSP             │
 │                         │                            │
 │                         ├──> Jakarta REST            │
 │                         ├──> CDI                     │
 │                         ├──> EJB                     │
 │                         ├──> JPA                     │
 │                         ├──> JTA                     │
 │                         ├──> JMS                     │
 │                         ├──> Security                │
 │                         └──> JNDI                    │
 │                                                      │
 │             Datasource ──> Database                  │
 │             JMS ─────────> Message Broker             │
 │                                                      │
 └──────────────────────────────────────────────────────┘
```

---

# 1. What Is an Application Server?

An **application server** provides the runtime environment and enterprise services required to execute enterprise applications.

Instead of your application implementing everything itself, the server provides infrastructure such as:

* HTTP request processing
* Servlet execution
* JSP processing
* Dependency injection
* Transactions
* Security
* Connection pooling
* JNDI
* Messaging
* Persistence integration
* EJB services
* Scheduling
* Clustering
* Resource management
* Monitoring
* Deployment management

Conceptually:

```text
Your Application
       ↓
Jakarta EE APIs
       ↓
Application Server
       ↓
Operating System
       ↓
Hardware / VM / Container
```

---

# 2. Why Do We Need an Application Server?

Imagine an enterprise application containing:

```text
REST API
Servlets
JPA
Database
Transactions
JMS
Authentication
Authorization
Scheduled jobs
Connection pools
```

Without a container, you would need to implement/manage much of this infrastructure yourself.

An application server provides these services.

For example:

```text
Application
    |
    +---- REST
    |
    +---- CDI
    |
    +---- JPA
    |
    +---- JTA
    |
    +---- JMS
    |
    +---- Security
    |
    +---- Transactions
    |
    v
Application Server
```

---

# 3. Web Server vs Servlet Container vs Application Server

This distinction is extremely important.

## Web Server

Primarily handles HTTP and static resources.

Examples:

* Apache HTTP Server
* Nginx

```text
Browser
   |
   v
Nginx
   |
   +--> HTML
   +--> CSS
   +--> JS
```

---

# 4. Servlet Container

A servlet container executes:

* Servlets
* Filters
* Listeners
* JSP

Examples:

* Apache Tomcat
* Jetty

Architecture:

```text
HTTP Request
     |
     v
Tomcat
     |
     v
Servlet
     |
     v
Service
```

Tomcat is primarily a **web/servlet container**, not a full Jakarta EE application server.

---

# 5. Full Jakarta EE Application Server

A full application server provides a broader Jakarta EE runtime.

Examples include:

* WildFly
* Payara
* GlassFish

Conceptually:

```text
                Application Server
                       |
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
 Web Container       CDI              EJB
       ↓               ↓                ↓
 Servlets           DI              Transactions
       │               │                │
       └───────────────┼────────────────┘
                       ↓
                     JPA
                       ↓
                    Database
```

---

# 6. Tomcat vs WildFly vs Payara vs GlassFish

| Feature         | Tomcat                    | WildFly           | Payara            | GlassFish                                   |
| --------------- | ------------------------- | ----------------- | ----------------- | ------------------------------------------- |
| Servlet         | Yes                       | Yes               | Yes               | Yes                                         |
| JSP             | Yes                       | Yes               | Yes               | Yes                                         |
| Full Jakarta EE | No                        | Yes               | Yes               | Yes                                         |
| CDI             | Limited/not full platform | Yes               | Yes               | Yes                                         |
| EJB             | No full EJB platform      | Yes               | Yes               | Yes                                         |
| JTA             | Not full Jakarta EE       | Yes               | Yes               | Yes                                         |
| JMS             | Not full platform         | Yes               | Yes               | Yes                                         |
| JPA integration | Can be added              | Yes               | Yes               | Yes                                         |
| Typical role    | Web container             | Enterprise server | Enterprise server | Jakarta EE reference implementation lineage |

The important distinction:

```text
Tomcat
   =
Servlet/JSP runtime

WildFly/Payara/GlassFish
   =
Jakarta EE application server
```

---

# 7. Jakarta EE Profiles

Jakarta EE provides different platform profiles.

The important idea is that not every application needs every enterprise capability.

Conceptually:

```text
Jakarta EE
   |
   +-- Web capabilities
   |
   +-- Core capabilities
   |
   +-- Enterprise capabilities
```

A lightweight application might need:

```text
REST
CDI
JSON
Validation
```

A larger enterprise application might additionally use:

```text
EJB
JMS
JTA
JPA
Security
JNDI
```

---

# 8. What Happens When You Deploy an Application?

Suppose you have:

```text
employee.war
```

You deploy it to WildFly.

Conceptually:

```text
employee.war
     |
     v
Application Server
     |
     +--> Read deployment descriptor
     |
     +--> Scan annotations
     |
     +--> Create application context
     |
     +--> Initialize CDI
     |
     +--> Initialize Servlets
     |
     +--> Configure resources
     |
     +--> Initialize persistence
     |
     +--> Register REST resources
     |
     +--> Start application
     |
     v
Application Ready
```

---

# 9. WAR File

WAR = **Web Application Archive**

Typical structure:

```text
employee.war

WEB-INF/
    web.xml
    classes/
    lib/

index.jsp
css/
js/
images/
```

Example:

```text
employee.war
│
├── index.jsp
│
└── WEB-INF
    ├── web.xml
    ├── classes
    │   └── com/example/...
    └── lib
        ├── library1.jar
        └── library2.jar
```

---

# 10. JAR

JAR = Java Archive.

Used for:

* libraries
* Java applications
* CDI modules
* utility modules

Example:

```text
employee-service.jar
```

---

# 11. EAR

EAR = Enterprise Archive.

Historically used to package multiple enterprise modules.

Example:

```text
company.ear
│
├── employee-web.war
├── employee-service.jar
├── billing.jar
└── lib/
```

Architecture:

```text
company.ear
     |
     +-- Web Module
     |      |
     |      +-- Servlets
     |      +-- REST
     |      +-- JSP
     |
     +-- Business Module
     |      |
     |      +-- EJB
     |      +-- CDI
     |
     +-- Libraries
```

---

# 12. WAR vs JAR vs EAR

| Package | Typical Purpose                                    |
| ------- | -------------------------------------------------- |
| JAR     | Java library/application/module                    |
| WAR     | Web application                                    |
| EAR     | Enterprise application containing multiple modules |

For modern deployments, you should understand all three even though many applications use simpler deployment models.

---

# 13. Application Server Architecture

Think of the server in layers:

```text
                  Client
                    |
                    v
              Load Balancer
                    |
                    v
          ┌───────────────────┐
          │ Application Server│
          ├───────────────────┤
          │ HTTP/Web           │
          │ Servlet            │
          │ REST               │
          │ CDI                │
          │ EJB                │
          │ JPA                │
          │ JTA                │
          │ JMS                │
          │ Security           │
          │ JNDI               │
          └───────────────────┘
                    |
          ┌─────────┴─────────┐
          ↓                   ↓
      Database            Message Broker
```

---

# 14. Deployment Lifecycle

Understand this sequence:

```text
Server Startup
      ↓
Read Configuration
      ↓
Initialize Services
      ↓
Initialize Datasources
      ↓
Initialize JMS
      ↓
Initialize Security
      ↓
Deploy Applications
      ↓
Scan Components
      ↓
Initialize Components
      ↓
Application Available
```

When undeploying:

```text
Stop Application
      ↓
Destroy Components
      ↓
Release Resources
      ↓
Close Connections
      ↓
Unload Application
```

---

# 15. Context Path

Suppose:

```text
employee.war
```

is deployed.

The context path may be:

```text
/employee
```

Then:

```text
http://localhost:8080/employee
```

If you have:

```java
@WebServlet("/employees")
```

the complete URL becomes:

```text
http://localhost:8080/employee/employees
```

---

# 16. Server Port

Typical development configuration:

```text
HTTP
8080

HTTPS
8443
```

For example:

```text
http://localhost:8080/employee/api/employees
```

The exact ports are configurable.

---

# 17. JNDI

JNDI = **Java Naming and Directory Interface**

It allows applications to look up resources by names.

Think:

```text
Application
     |
     | lookup
     v
JNDI
     |
     +---- DataSource
     +---- JMS ConnectionFactory
     +---- Queue
     +---- Topic
```

Example conceptual name:

```text
java:/jdbc/EmployeeDS
```

Your application doesn't need to hard-code:

```text
database host
database username
database password
```

Instead:

```text
Application
     |
     v
JNDI DataSource
     |
     v
Connection Pool
     |
     v
Database
```

This is extremely important in enterprise deployment.

---

# 18. Datasources

Instead of creating database connections manually:

```java
DriverManager.getConnection(...);
```

enterprise applications commonly use:

```text
DataSource
```

managed by the application server.

Architecture:

```text
Application
     |
     v
DataSource
     |
     v
Connection Pool
     |
     +-- Connection 1
     +-- Connection 2
     +-- Connection 3
     +-- Connection 4
     |
     v
Database
```

---

# 19. Why Connection Pooling?

Creating a database connection repeatedly is expensive.

Without pooling:

```text
Request
  ↓
Create connection
  ↓
Execute SQL
  ↓
Close connection
```

With pooling:

```text
Request
  ↓
Borrow connection
  ↓
Execute SQL
  ↓
Return connection
```

The connections remain available for reuse.

---

# 20. Connection Pool Configuration

Important concepts:

```text
minimum pool size
maximum pool size
connection timeout
idle timeout
validation
connection leak detection
```

For example:

```text
min = 10
max = 100
```

does **not** mean you should blindly configure 100 connections.

The correct pool size depends on:

* database capacity
* workload
* query duration
* number of application instances
* concurrency
* CPU
* connection limits

---

# 21. JMS Resources

Application servers can expose messaging resources through JNDI.

Example:

```text
java:/jms/OrderQueue
```

Application:

```text
JNDI
 |
 +-- ConnectionFactory
 |
 +-- Queue
 |
 +-- Topic
```

Then:

```text
OrderService
     |
     v
OrderQueue
     |
     v
InventoryConsumer
```

---

# 22. Security Configuration

Application servers can manage security resources such as:

```text
Users
Groups
Roles
Realms
Identity stores
Authentication mechanisms
Authorization
```

Conceptually:

```text
HTTP Request
     |
     v
Authentication
     |
     v
Identity
     |
     v
Roles
     |
     v
Authorization
     |
     v
Application
```

---

# 23. Authentication vs Authorization

Authentication:

> Who are you?

Authorization:

> What are you allowed to do?

Example:

```text
User: pradeep

Roles:
    EMPLOYEE
    MANAGER
```

Endpoint:

```text
DELETE /employees/10
```

could require:

```text
MANAGER
```

---

# 24. Class Loading

One of the most important advanced application-server concepts.

Imagine:

```text
Application A
    |
    +-- library-x v1

Application B
    |
    +-- library-x v2
```

How can both applications work?

Application servers use classloader/module isolation mechanisms.

Conceptually:

```text
Application Server
       |
       +---- App A ClassLoader
       |
       +---- App B ClassLoader
       |
       +---- Server Modules
```

This prevents applications from automatically sharing every class.

---

# 25. Why Classloader Problems Happen

A classic problem:

```text
ClassCastException
```

even though classes appear to have the same name.

For example:

```text
com.example.User
```

loaded by:

```text
ClassLoader A
```

is not necessarily the same runtime type as:

```text
com.example.User
```

loaded by:

```text
ClassLoader B
```

The classloader identity matters.

---

# 26. Dependency Conflicts

Suppose:

```text
Application
 ├── jackson 2.x
 └── server provides another Jackson version
```

Potential issues:

```text
NoSuchMethodError
ClassNotFoundException
NoClassDefFoundError
LinkageError
ClassCastException
```

This is why understanding server classloading is important.

---

# 27. Server Configuration

Application servers have server-level configuration.

Conceptually:

```text
Server Configuration
│
├── HTTP listeners
├── HTTPS
├── Datasources
├── JMS
├── Security
├── Logging
├── Thread pools
├── Transactions
├── Clustering
└── Deployment settings
```

---

# 28. Standalone vs Managed/Domain Configuration

Some application servers support different management models.

A standalone server:

```text
Server
 ├── Application
 ├── Datasource
 ├── JMS
 └── Security
```

A managed/domain-style setup can coordinate multiple servers:

```text
Management
      |
 ┌────┼────┐
 ↓    ↓    ↓
Node1 Node2 Node3
```

Useful for larger deployments.

---

# 29. Management Console

Application servers usually provide administrative interfaces.

You may configure:

* deployments
* datasources
* JMS
* logging
* security
* server runtime
* metrics
* thread pools

---

# 30. CLI Administration

Production engineers often automate administration instead of manually clicking through a console.

Conceptually:

```text
CLI / Automation
      |
      v
Application Server
      |
      +--> Deploy
      +--> Configure
      +--> Restart
      +--> Read metrics
      +--> Manage resources
```

This is important for DevOps.

---

# 31. Logging

A production application needs structured logging.

Avoid:

```java
System.out.println("Employee created");
```

Prefer a logging framework and appropriate levels:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

Example:

```text
2026-10-27 10:30:12 INFO
employee-service
employee-created
employeeId=101
traceId=abc123
```

---

# 32. Correlation IDs

Suppose:

```text
Client
  ↓
Load Balancer
  ↓
Employee API
  ↓
Order Service
  ↓
Notification Service
```

You need to trace one request across services.

Example:

```text
X-Correlation-ID: 8fa23b
```

Then logs can contain:

```text
traceId=8fa23b
```

throughout the request.

---

# 33. Health Checks

Production applications need health information.

Two common concepts:

### Liveness

Is the process alive?

```text
Application running?
```

### Readiness

Can the application accept traffic?

```text
Database available?
Required dependencies available?
Application initialized?
```

Architecture:

```text
Load Balancer
      |
      | readiness
      v
Application Server
```

---

# 34. Graceful Shutdown

Never think of production shutdown as simply:

```text
kill process
```

A graceful shutdown should allow:

```text
Stop accepting new traffic
        ↓
Finish existing requests
        ↓
Complete/rollback transactions
        ↓
Stop consumers
        ↓
Close resources
        ↓
Shutdown
```

This becomes especially important with:

* JMS consumers
* long-running requests
* transactions
* background jobs

---

# 35. Clustering

Suppose:

```text
           Load Balancer
            /         \
           /           \
      Server A       Server B
```

Both run:

```text
employee.war
```

Now the application can handle more traffic and potentially tolerate individual server failures.

---

# 36. Session Replication

Suppose a user logs in:

```text
Request 1 → Server A
```

Then:

```text
Request 2 → Server B
```

If the session exists only on Server A:

```text
Server B
   |
   └── Session not found
```

Solutions include:

* sticky sessions
* session replication
* external session storage

Modern distributed systems often prefer minimizing server-local session state.

---

# 37. Sticky Sessions

Load balancer consistently sends a user's requests to the same server.

```text
User A
  |
  v
Load Balancer
  |
  +----> Server A
          ↑
       User A
```

Advantage:

```text
Simple
```

Disadvantages:

```text
Uneven distribution
Failure can lose session
Scaling becomes more complicated
```

---

# 38. External Session Storage

Another architecture:

```text
Server A ──┐
           ├──> Redis
Server B ──┤
           │
Server C ──┘
```

Now session state isn't tied to one application-server instance.

---

# 39. Deployment Architecture

A production Jakarta EE deployment might look like:

```text
                    Internet
                       |
                       v
                Load Balancer
                  /         \
                 /           \
                v             v
          App Server A   App Server B
                |             |
                +------┬------+
                       |
                Connection Pool
                       |
                       v
                  PostgreSQL
```

Messaging:

```text
App Server
    |
    v
Message Broker
    |
    +--> Consumer A
    +--> Consumer B
```

---

# 40. Environment Configuration

Never hard-code production configuration:

```java
String password = "mypassword";
```

Instead:

```text
Development
    ↓
dev datasource

Testing
    ↓
test datasource

Production
    ↓
production datasource
```

Typical environment-specific resources:

```text
Database URL
Credentials
JNDI names
Messaging destinations
API endpoints
Timeouts
Feature configuration
Security configuration
```

---

# 41. Secrets

Do not put production secrets into:

```text
Git
Source code
WAR
EAR
JSP
Logs
```

Use appropriate secret-management mechanisms.

Conceptually:

```text
Application
     |
     v
Secret Manager
     |
     v
Credentials
```

---

# 42. Deployment Strategies

Important deployment strategies:

### In-place deployment

```text
Stop old
Deploy new
Start new
```

Possible downtime.

---

### Rolling deployment

```text
Server A → version 2
Server B → version 1

Server A healthy
        ↓
Server B → version 2
```

Useful for clusters.

---

### Blue-Green

```text
Blue
Version 1

Green
Version 2
```

Traffic can be switched between environments.

---

### Canary

A small percentage of traffic goes to the new version.

```text
95% → V1
5%  → V2
```

Then traffic can be increased based on observed behavior.

---

# 43. Application Server Failure

Imagine:

```text
Server A ❌
```

Load balancer should stop sending traffic there.

```text
              Load Balancer
                    |
              ┌─────┴─────┐
              ↓           ↓
          Server A ❌   Server B ✅
```

This requires:

* health checks
* proper timeout configuration
* connection management
* graceful recovery
* stateless application design where possible

---

# 44. Application Server vs Spring Boot

You should understand this distinction because you're targeting Java backend architecture.

Traditional Jakarta EE:

```text
Application
     ↓
Application Server
     ↓
Jakarta EE APIs
```

Spring Boot:

```text
Application
     ↓
Embedded Runtime
     ↓
Spring Framework
```

Spring Boot commonly embeds a web server such as Tomcat, Jetty, or Undertow.

The architectural question isn't simply:

> Which one is better?

Instead ask:

> What runtime model, operational model, ecosystem, deployment constraints, and enterprise capabilities does the system require?

---

# 45. Traditional Enterprise Java Architecture

A classic application:

```text
                 Client
                   |
                   v
             Load Balancer
                   |
                   v
            Web/Application
                Server
                   |
        ┌──────────┼──────────┐
        ↓          ↓          ↓
     Servlet      REST       EJB
        |          |          |
        └──────┬───┴──────────┘
               ↓
             CDI
               ↓
            Service
               ↓
              JPA
               ↓
              JTA
               ↓
          DataSource
               ↓
           Database
```

---

# 46. Enterprise Deployment Example

Suppose your Employee application has:

```text
employee.war
```

and requires:

```text
PostgreSQL
JMS
Security
JNDI
```

Production:

```text
                    Load Balancer
                         |
               ┌─────────┴─────────┐
               ↓                   ↓
         WildFly Node 1      WildFly Node 2
               |                   |
               ├───────┬───────────┤
               ↓       ↓           ↓
             JNDI    JMS        Security
               |
               ↓
          DataSource
               |
               ↓
          PostgreSQL
```

---

# 47. Example JNDI DataSource Usage

Your application should depend on the configured resource rather than knowing infrastructure details.

Conceptually:

```java
@Resource(lookup = "java:/jdbc/EmployeeDS")
private DataSource dataSource;
```

Then:

```java
try (Connection connection = dataSource.getConnection()) {

    // execute SQL

}
```

Notice the application does not need to construct the database connection manually.

---

# 48. Why JNDI Is Useful

It separates:

```text
Application code
```

from:

```text
Infrastructure configuration
```

For example:

```text
Development
java:/jdbc/EmployeeDS
        ↓
dev-db

Production
java:/jdbc/EmployeeDS
        ↓
production-db
```

The application code remains unchanged.

---

# 49. Application Server Responsibilities

Memorize this:

```text
Application Server
│
├── Deployment
├── HTTP
├── Servlet
├── REST
├── CDI
├── EJB
├── JPA integration
├── Transactions
├── JMS
├── JNDI
├── Security
├── Connection pools
├── Thread management
├── Logging
├── Monitoring
├── Clustering
└── Resource management
```

---

# 50. What Should Your Application NOT Do?

Avoid application code that manually manages infrastructure unnecessarily.

Bad:

```java
new Thread(...);
```

for every request.

Bad:

```java
DriverManager.getConnection(...)
```

every request in a production enterprise application.

Bad:

```java
System.out.println(...)
```

for production observability.

Bad:

```java
String password = "...";
```

Bad:

```java
static Map<String, Object> sessions;
```

for distributed session management.

The runtime should provide appropriate infrastructure services.

---

# 51. Day 25 Practical Project

Build:

# Employee Enterprise Deployment Platform

Architecture:

```text
                    Browser
                       |
                       v
                 Load Balancer
                       |
              ┌────────┴────────┐
              ↓                 ↓
        Application A      Application B
              |                 |
              └────────┬────────┘
                       ↓
                    JNDI
                       |
                ┌──────┴──────┐
                ↓             ↓
           DataSource      JMS Queue
                ↓             ↓
           PostgreSQL    Notification
```

Application capabilities:

```text
Employee CRUD
Authentication
Authorization
REST API
JSP UI
JPA
Transactions
JMS
Logging
Validation
Exception handling
```

---

# 52. Suggested Project Structure

```text
employee-enterprise/
│
├── src/main/java/
│   └── com/example/employee/
│       │
│       ├── controller/
│       │   └── EmployeeServlet.java
│       │
│       ├── resource/
│       │   └── EmployeeResource.java
│       │
│       ├── service/
│       │   └── EmployeeService.java
│       │
│       ├── repository/
│       │   └── EmployeeRepository.java
│       │
│       ├── entity/
│       │   └── Employee.java
│       │
│       ├── dto/
│       │   └── EmployeeDTO.java
│       │
│       ├── messaging/
│       │   └── EmployeeEventProducer.java
│       │
│       ├── security/
│       │   └── SecurityFilter.java
│       │
│       └── exception/
│           └── EmployeeNotFoundException.java
│
├── src/main/resources/
│
├── src/main/webapp/
│   └── WEB-INF/
│       └── views/
│
└── pom.xml
```

---

# 53. Deployment Exercise

Deploy your application to an application server.

Practice:

### Step 1

Build:

```bash
mvn clean package
```

### Step 2

Find:

```text
target/employee.war
```

### Step 3

Deploy it to your application server.

### Step 4

Verify:

```text
http://localhost:8080/employee
```

### Step 5

Verify REST:

```text
GET /employee/api/employees
```

### Step 6

Verify database connectivity.

### Step 7

Verify logs.

### Step 8

Verify application shutdown/restart.

---

# 54. Troubleshooting Checklist

When deployment fails:

```text
1. Is the server running?
2. Is the WAR valid?
3. Is the application deployed?
4. Is the context path correct?
5. Are dependencies present?
6. Are there classloading conflicts?
7. Is JNDI configured?
8. Is the datasource available?
9. Is the database reachable?
10. Is the database credential correct?
11. Is the port available?
12. Are annotations being discovered?
13. Is CDI initialized?
14. Is JPA initialized?
15. Is the transaction manager available?
16. Are there server logs?
```

---

# 55. Common Production Errors

### `404 Not Found`

Possible causes:

```text
Wrong context path
Wrong endpoint
Application not deployed
Wrong URL mapping
```

---

### `500 Internal Server Error`

Possible causes:

```text
Application exception
Configuration error
Database failure
Dependency problem
```

---

### `ClassNotFoundException`

Usually investigate:

```text
Missing dependency
Packaging
Classloader
Server module
```

---

### `NoSuchMethodError`

Often investigate:

```text
Dependency version conflict
Classloader conflict
Incompatible library versions
```

---

### Database connection failure

Check:

```text
Datasource
JNDI name
Credentials
Network
Database availability
Pool configuration
```

---

# 56. Important Architect-Level Concept

Don't think:

```text
Application Server = Tomcat
```

Think:

```text
Runtime responsibilities
       ↓
Which runtime provides them?
       ↓
Which deployment model?
       ↓
Which infrastructure?
       ↓
Which operational requirements?
```

This is the mindset you need for architecture.

---

# 57. 20 Interview Questions

### Q1. What is an application server?

A runtime providing enterprise application services such as web processing, transactions, security, dependency injection, messaging, persistence integration, resource management, and deployment.

### Q2. Is Tomcat a full Jakarta EE application server?

No. Tomcat primarily provides servlet/JSP web-container functionality.

### Q3. What is WAR?

A Web Application Archive containing a web application.

### Q4. What is EAR?

An Enterprise Archive that can package multiple enterprise modules.

### Q5. What is JNDI?

A naming mechanism used to locate resources such as datasources and messaging resources.

### Q6. Why use a datasource?

To provide managed database connections, commonly with connection pooling.

### Q7. Why use connection pooling?

To avoid repeatedly creating expensive database connections.

### Q8. What is classloader isolation?

A mechanism allowing different applications/modules to load and isolate their dependencies.

### Q9. What causes `NoSuchMethodError`?

A common cause is incompatible versions of classes/libraries at runtime.

### Q10. What is a context path?

The URL prefix identifying a deployed web application.

### Q11. What is clustering?

Running multiple application-server instances as a coordinated deployment to support scalability and/or availability.

### Q12. What is sticky session?

A load-balancing technique that routes a user's requests consistently to one server.

### Q13. What is graceful shutdown?

A controlled shutdown that stops new work and allows appropriate existing work/resources to finish safely.

### Q14. What is a health check?

A mechanism to determine whether an application instance is alive or ready to receive traffic.

### Q15. What is JNDI datasource injection?

Obtaining a server-managed datasource through a configured JNDI resource.

### Q16. Why shouldn't credentials be hard-coded?

Because source code is not an appropriate secure secret-management mechanism.

### Q17. What is application deployment?

Making an application archive available and initialized by the runtime.

### Q18. Why are logs important?

They provide diagnostic and operational information about application behavior and failures.

### Q19. What is rolling deployment?

Updating instances progressively rather than replacing every instance simultaneously.

### Q20. What is the difference between web container and application server?

A web container primarily manages web components such as Servlets, while a full Jakarta EE application server provides a broader set of enterprise services.

---

# 58. Architect-Level Questions

Think through these carefully.

### Scenario 1

You have:

```text
1000 requests/sec
2 application servers
PostgreSQL
JMS
```

Questions:

* How many DB connections should the pool have?
* What happens if the DB reaches its connection limit?
* How will you scale application servers?
* How will you handle sessions?
* What happens if one server dies?

---

### Scenario 2

You deploy:

```text
employee.war
```

and receive:

```text
ClassNotFoundException
```

Investigate:

```text
WAR packaging
        ↓
WEB-INF/lib
        ↓
Server modules
        ↓
Classloader
        ↓
Dependency versions
```

---

### Scenario 3

Your deployment works locally but fails in production.

Investigate:

```text
Configuration
Environment variables
JNDI
Datasource
Security
Network
Database
Dependencies
Server version
Java version
```

---

# 59. Day 25 Hands-On Exercises

Complete these:

### Beginner

1. Install an application server.
2. Start/stop it.
3. Deploy a WAR.
4. Undeploy a WAR.
5. Change the context path.
6. Change HTTP port.
7. Read server logs.
8. Access the management console.

### Intermediate

9. Configure a datasource.
10. Access it using JNDI.
11. Configure a connection pool.
12. Configure a JMS queue.
13. Send a JMS message.
14. Consume the message.
15. Configure application security.
16. Deploy two applications.
17. Investigate classloader behavior.
18. Configure health checks.

### Advanced

19. Deploy two application-server instances.
20. Put a load balancer in front.
21. Test session behavior.
22. Test server failure.
23. Configure graceful shutdown.
24. Implement correlation IDs.
25. Perform rolling deployment.

---

# 60. Day 25 Mastery Checklist

You should be able to explain:

* [ ] What is an application server?
* [ ] Web server vs servlet container vs application server
* [ ] Tomcat vs full Jakarta EE server
* [ ] WildFly
* [ ] Payara
* [ ] GlassFish
* [ ] WAR
* [ ] JAR
* [ ] EAR
* [ ] Deployment lifecycle
* [ ] Context path
* [ ] JNDI
* [ ] Datasource
* [ ] Connection pooling
* [ ] JMS resources
* [ ] Server security
* [ ] Classloading
* [ ] Dependency isolation
* [ ] Server configuration
* [ ] Management console
* [ ] CLI administration
* [ ] Logging
* [ ] Health checks
* [ ] Graceful shutdown
* [ ] Clustering
* [ ] Session replication
* [ ] Sticky sessions
* [ ] Rolling deployment
* [ ] Blue-green deployment
* [ ] Canary deployment
* [ ] Production troubleshooting

---

# 61. Your Mental Model for Day 25

Memorize this architecture:

```text
                         CLIENT
                           |
                           v
                    LOAD BALANCER
                           |
             ┌─────────────┴─────────────┐
             ↓                           ↓
       APP SERVER A                 APP SERVER B
             |                           |
     ┌───────┼────────┐          ┌───────┼────────┐
     ↓       ↓        ↓          ↓       ↓        ↓
 Servlet   REST      CDI       Servlet   REST     CDI
     |       |        |          |       |        |
     └───────┴────────┘          └───────┴────────┘
             |                           |
             └───────────┬───────────────┘
                         ↓
                       JPA
                         ↓
                       JTA
                         ↓
                    DataSource
                         ↓
                     Database

                         +
                         |
                        JMS
                         |
                         v
                  Message Broker
```

### Golden Rule

> **Your Jakarta EE application contains business functionality; the application server provides the runtime infrastructure that allows that functionality to execute reliably, securely, transactionally, and at scale.**

---

## Day 25 → Day 26

**Day 26: JNDI, Datasources & Resource Management**

Tomorrow you should go much deeper into:

```text
JNDI
  ↓
Naming Contexts
  ↓
DataSource
  ↓
Connection Pool
  ↓
JPA/JDBC
  ↓
JTA
  ↓
JMS Resources
  ↓
Resource Injection
  ↓
Environment Configuration
  ↓
Production Resource Management
```

This is where your understanding moves from **“I can deploy a Jakarta EE application”** to **“I understand how enterprise infrastructure is wired into the application.”**
