# J2EE Mastery — Day 1

## HTTP & Enterprise Web Architecture

**Day 1: October 3, 2026**

Today is about understanding the **foundation underneath every Java enterprise web application**. Before learning Servlets, JSP, REST, Spring, or Jakarta EE, you should understand exactly what happens when a client sends a request to a Java server.

---

# 1. What is J2EE?

**J2EE** originally stood for **Java 2 Platform, Enterprise Edition**.

It evolved through:

```text
J2EE
  ↓
Java EE
  ↓
Jakarta EE
```

J2EE/Java EE/Jakarta EE provides specifications and APIs for building enterprise applications.

Typical enterprise requirements include:

* HTTP communication
* Web applications
* REST APIs
* Database access
* Transactions
* Security
* Dependency injection
* Messaging
* Scheduling
* Distributed processing
* Application deployment

A simplified enterprise application:

```text
                 Client
                   │
                 HTTP
                   │
                   ▼
             Web Server
                   │
                   ▼
          Servlet Container
                   │
                   ▼
          Java Application
                   │
          ┌────────┴────────┐
          ▼                 ▼
       Database          Message Broker
```

---

# 2. Java SE vs J2EE/Jakarta EE

This distinction is important in interviews.

### Java SE

Java SE provides the core Java platform:

* Classes and objects
* OOP
* Collections
* Generics
* Exceptions
* Threads
* Streams
* I/O
* Networking
* JDBC APIs
* JVM

Example:

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello Java");
    }
}
```

### Jakarta EE

Jakarta EE adds enterprise application capabilities around Java.

Examples include:

* Servlets
* Jakarta REST
* Jakarta Persistence
* Jakarta CDI
* Jakarta Transactions
* Jakarta Messaging
* Jakarta Validation
* Jakarta Security

Think:

```text
Java SE
   +
Enterprise APIs
   +
Application Server
   =
Enterprise Java Application
```

---

# 3. What is an Enterprise Application?

An enterprise application is software designed to support organizational/business processes.

Examples:

* Banking systems
* E-commerce
* Insurance platforms
* Healthcare systems
* Telecom systems
* ERP systems
* Payment systems
* Trading platforms
* HR systems

Enterprise applications commonly need:

```text
Security
Scalability
Reliability
Transactions
Persistence
Concurrency
Integration
Monitoring
Auditing
```

For example, consider transferring money:

```text
User
 ↓
POST /transfer
 ↓
Authentication
 ↓
Authorization
 ↓
Validate request
 ↓
Start transaction
 ↓
Debit Account A
 ↓
Credit Account B
 ↓
Commit transaction
 ↓
Publish transaction event
 ↓
Return response
```

That's much more than simply executing a Java method.

---

# 4. What is a Web Application?

A web application is an application accessed through a network, typically using HTTP/HTTPS.

For example:

```text
https://example.com/products
```

The browser acts as the client.

The server processes the request and returns a response.

```text
Browser
   │
   │ HTTP Request
   ▼
Server
   │
   │ HTTP Response
   ▼
Browser
```

---

# 5. Client-Server Architecture

The fundamental model is:

```text
Client                    Server
  │                         │
  │────── Request ─────────>│
  │                         │
  │                         │ Process
  │                         │ request
  │                         │
  │<───── Response ─────────│
  │                         │
```

### Client

Examples:

* Browser
* Mobile application
* React application
* Angular application
* Another backend service
* Postman

### Server

Could contain:

* Web server
* Servlet container
* Java application
* Database
* Cache
* Message broker

---

# 6. What Happens When You Open a URL?

Suppose you enter:

```text
https://example.com/users/10
```

A simplified flow is:

```text
Browser
   ↓
DNS
   ↓
IP Address
   ↓
TCP connection
   ↓
TLS/HTTPS
   ↓
HTTP Request
   ↓
Web Server
   ↓
Application Server
   ↓
Java Application
   ↓
Database
   ↓
Java Application
   ↓
HTTP Response
   ↓
Browser
```

You should eventually be able to explain every stage.

---

# 7. DNS

DNS means:

**Domain Name System**

Humans use:

```text
example.com
```

Machines communicate using IP addresses.

DNS resolves:

```text
example.com
      ↓
IP address
```

Conceptually:

```text
Browser
   ↓
DNS lookup
   ↓
93.184.x.x
   ↓
Server
```

---

# 8. TCP

HTTP commonly operates over TCP.

TCP provides reliable communication.

Simplified:

```text
Client
   │
   │ TCP connection
   ▼
Server
```

For traditional HTTP/1.1 and HTTP/2 deployments, TCP is commonly involved.

HTTP/3 instead uses QUIC over UDP, which we'll cover later when discussing modern web architecture.

---

# 9. HTTP

HTTP means:

**HyperText Transfer Protocol**

It defines how clients and servers communicate.

Example request:

```http
GET /users/10 HTTP/1.1
Host: example.com
Accept: application/json
```

Example response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 10,
    "name": "John"
}
```

---

# 10. HTTP Request Structure

An HTTP request can contain:

```text
Request Line
Headers
Blank Line
Body
```

Example:

```http
POST /users HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer token

{
    "name": "John",
    "email": "john@example.com"
}
```

### Request line

```http
POST /users HTTP/1.1
```

Contains:

```text
HTTP Method
Path
HTTP Version
```

---

# 11. HTTP Methods

The major HTTP methods are:

| Method  | Typical purpose      |
| ------- | -------------------- |
| GET     | Retrieve data        |
| POST    | Create/process       |
| PUT     | Replace/update       |
| PATCH   | Partial update       |
| DELETE  | Delete               |
| HEAD    | Headers only         |
| OPTIONS | Supported operations |

Example REST API:

```text
GET     /users
GET     /users/10
POST    /users
PUT     /users/10
PATCH   /users/10
DELETE  /users/10
```

---

# 12. GET

Used primarily for retrieving resources.

```http
GET /users/10
```

Response:

```json
{
    "id": 10,
    "name": "John"
}
```

GET should generally be **safe** and should not be designed to modify server state.

---

# 13. POST

Usually used to create a resource or trigger an operation.

```http
POST /users
Content-Type: application/json

{
    "name": "John"
}
```

Possible response:

```http
201 Created
```

---

# 14. PUT

Typically replaces the representation of a resource.

```http
PUT /users/10

{
    "id": 10,
    "name": "John Smith"
}
```

PUT is generally considered **idempotent**.

---

# 15. PATCH

Used for partial modification.

```http
PATCH /users/10

{
    "name": "John Smith"
}
```

Only the specified fields need to change.

---

# 16. DELETE

Used to remove a resource.

```http
DELETE /users/10
```

Possible response:

```http
204 No Content
```

---

# 17. HTTP Headers

Headers carry metadata.

Example:

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer abc123
User-Agent: Mozilla/5.0
```

Important headers to learn:

### Content-Type

Describes the request/response body format.

```http
Content-Type: application/json
```

### Accept

Tells the server which response formats the client prefers.

```http
Accept: application/json
```

### Authorization

Carries authentication credentials.

```http
Authorization: Bearer <token>
```

### Cookie

Sends cookies to the server.

```http
Cookie: SESSIONID=abc123
```

---

# 18. HTTP Request Body

The body contains data sent to the server.

Example:

```http
POST /employees HTTP/1.1
Content-Type: application/json

{
    "name": "Pradeep",
    "department": "Engineering"
}
```

The body might contain:

* JSON
* XML
* Form data
* Multipart data
* Plain text
* Binary data

---

# 19. HTTP Response

A response contains:

```text
Status line
Headers
Blank line
Body
```

Example:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 101,
    "name": "Pradeep"
}
```

---

# 20. HTTP Status Codes

You must memorize the major ones.

## 2xx — Success

```text
200 OK
201 Created
202 Accepted
204 No Content
```

## 3xx — Redirection

```text
301 Moved Permanently
302 Found
304 Not Modified
307 Temporary Redirect
308 Permanent Redirect
```

## 4xx — Client-side/request problems

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
405 Method Not Allowed
409 Conflict
415 Unsupported Media Type
422 Unprocessable Content
429 Too Many Requests
```

## 5xx — Server-side failures

```text
500 Internal Server Error
501 Not Implemented
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

### Important interview question

**401 vs 403?**

Generally:

```text
401 → authentication is required/failed
403 → server understood the request but refuses authorization
```

---

# 21. HTTP Is Stateless

HTTP itself is stateless.

Suppose:

```text
Request 1:
GET /profile

Request 2:
GET /profile
```

The server does not inherently remember that both requests came from the same user.

State must be implemented using mechanisms such as:

* Cookies
* Sessions
* Tokens
* Database
* Cache

---

# 22. Cookies

A cookie is data stored by the client and sent with subsequent requests to matching scope.

Server:

```http
Set-Cookie: SESSIONID=abc123
```

Client:

```http
Cookie: SESSIONID=abc123
```

This is heavily used in traditional Java web applications.

---

# 23. Session

A server-side session allows the application to maintain state associated with a client.

Conceptually:

```text
Browser
   │
   │ SESSIONID=ABC
   ▼
Java Server
   │
   ▼
Session Store
```

The session might contain:

```text
userId = 101
role = ADMIN
loginTime = ...
```

Later we'll implement this with `HttpSession`.

---

# 24. URL Structure

Consider:

```text
https://example.com:8443/shop/products?id=100#reviews
```

Break it down:

```text
https
  ↓
Protocol

example.com
  ↓
Host

8443
  ↓
Port

/shop/products
  ↓
Path

?id=100
  ↓
Query parameter

#reviews
  ↓
Fragment
```

Important distinction:

**Fragment identifiers are normally handled by the client and are not sent as part of an HTTP request to the server.**

---

# 25. Port

A server can listen on different ports.

Common examples:

```text
HTTP   → 80
HTTPS  → 443
```

Java development commonly uses:

```text
Tomcat → 8080
```

So:

```text
http://localhost:8080
```

means:

```text
localhost
    +
port 8080
```

---

# 26. Web Server vs Application Server

This is extremely important for J2EE interviews.

### Web Server

Primarily handles web/HTTP concerns and static content.

Examples include:

* Apache HTTP Server
* Nginx

### Application Server

Provides runtime services for enterprise applications.

Historically/currently associated with Jakarta EE runtimes are servers such as:

* WildFly
* Payara
* GlassFish

Tomcat is primarily a **Servlet container**, rather than a full Jakarta EE application server.

---

# 27. Servlet Container

A Servlet container manages Servlet-based web applications.

Its responsibilities include:

* Creating servlet instances
* Managing servlet lifecycle
* Routing requests
* Managing threads
* Handling HTTP communication
* Sessions
* Filters
* Listeners

Conceptually:

```text
HTTP Request
     ↓
Servlet Container
     ↓
Servlet
     ↓
Business Logic
     ↓
HTTP Response
```

Tomcat is a widely used Servlet container.

---

# 28. Why Do We Need a Servlet Container?

Without a container, you'd have to manually implement things such as:

```text
Open socket
Receive HTTP request
Parse HTTP
Create request object
Create response object
Manage threads
Call application code
Manage sessions
Return HTTP response
```

The container handles these infrastructure concerns.

Your application can focus on business logic.

---

# 29. Java Web Application Architecture

A traditional Java web application could look like:

```text
                    Client
                      │
                      ▼
                   HTTP
                      │
                      ▼
              Servlet Container
                      │
               ┌──────┴──────┐
               ▼             ▼
             Filter       Servlet
                              │
                              ▼
                         Service Layer
                              │
                              ▼
                           DAO
                              │
                              ▼
                          Database
```

You'll build each part during the next few days.

---

# 30. Static vs Dynamic Content

### Static content

Doesn't require application processing for each request.

Examples:

```text
HTML
CSS
JavaScript
Images
```

### Dynamic content

Generated based on application state/data.

Example:

```text
GET /employees/100
```

The application might query:

```sql
SELECT * FROM employees WHERE id = 100;
```

and dynamically produce the response.

---

# 31. Traditional J2EE Request Flow

This is one of the most important diagrams for your Day 1.

```text
                 Browser
                    │
                    │ HTTP
                    ▼
             Reverse Proxy/
              Web Server
                    │
                    ▼
          Servlet Container
                    │
                    ▼
                 Filter
                    │
                    ▼
                Servlet
                    │
                    ▼
             Service Layer
                    │
                    ▼
                  DAO
                    │
                    ▼
               Database
                    │
                    ▼
                Response
                    │
                    ▼
                 Browser
```

By the end of this 30-day course, you should understand every box.

---

# 32. Request-Response Example

Suppose the user visits:

```text
GET /employees/101
```

### Step 1

Browser generates:

```http
GET /employees/101 HTTP/1.1
Host: localhost:8080
Accept: application/json
```

### Step 2

Request reaches the Java server.

### Step 3

Servlet container determines which Servlet should handle:

```text
/employees/*
```

### Step 4

Servlet receives:

```text
HttpServletRequest
```

and creates:

```text
HttpServletResponse
```

### Step 5

Application executes:

```java
Employee employee = employeeService.findById(101);
```

### Step 6

Service calls DAO:

```java
employeeDAO.findById(101);
```

### Step 7

DAO queries database.

### Step 8

Application generates response:

```json
{
    "id": 101,
    "name": "John",
    "department": "Engineering"
}
```

### Step 9

HTTP response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
    "id": 101,
    "name": "John",
    "department": "Engineering"
}
```

---

# 33. Important Concepts to Master Today

You should be able to define these without notes:

```text
J2EE
Java EE
Jakarta EE
Enterprise Application
Web Application
Client
Server
HTTP
HTTPS
Request
Response
HTTP Method
HTTP Header
HTTP Body
Status Code
Cookie
Session
URL
URI
Port
DNS
TCP
Web Server
Application Server
Servlet Container
Static Content
Dynamic Content
Statelessness
```

---

# 34. Day 1 Interview Questions

Try answering these yourself before looking at the answers.

### Q1. What is J2EE?

J2EE was Java's enterprise platform, now evolved into Java EE and Jakarta EE, providing APIs/specifications for enterprise application development.

### Q2. What is the difference between Java SE and Jakarta EE?

Java SE provides the core Java platform; Jakarta EE provides enterprise specifications and APIs for building server-side applications.

### Q3. What is HTTP?

HTTP is an application-layer protocol used for communication between clients and servers.

### Q4. What is the difference between HTTP and HTTPS?

HTTPS is HTTP secured using TLS.

### Q5. What does stateless mean in HTTP?

Each HTTP request is independent; HTTP itself doesn't inherently maintain application session state between requests.

### Q6. What is a Servlet container?

A Servlet container manages Servlet-based web applications, including servlet lifecycle, request routing, threading, filters, and sessions.

### Q7. Is Tomcat an application server?

Tomcat is primarily a Servlet container/web application runtime rather than a full Jakarta EE application server.

### Q8. What is the difference between 401 and 403?

401 generally indicates missing/failed authentication; 403 indicates that the request is understood but access is not permitted.

### Q9. What is a cookie?

A cookie is client-side data that can be sent with subsequent HTTP requests according to its scope and attributes.

### Q10. Why do we need sessions?

Sessions provide a mechanism for maintaining server-side state associated with a client across multiple requests.

### Q11. What is the difference between GET and POST?

GET is primarily used to retrieve resources, while POST is commonly used to create resources or submit/process data.

### Q12. What is idempotency?

An operation is idempotent if repeating the same operation has the same intended effect as performing it once. PUT and DELETE are generally designed to be idempotent; POST generally isn't.

---

# 35. Hands-On Exercise

Create this architecture on paper first:

```text
Browser
   ↓
HTTP
   ↓
Tomcat
   ↓
Servlet
   ↓
Service
   ↓
DAO
   ↓
MySQL/PostgreSQL
```

Then answer:

1. Who receives the HTTP request first?
2. Who decides which Servlet handles it?
3. Who manages the Servlet lifecycle?
4. Where should business logic live?
5. Where should SQL code live?
6. Where should database connections be managed?
7. How does the server identify a returning session?
8. What happens when the database is unavailable?
9. What HTTP status should represent a successful creation?
10. What status should represent an unavailable server?

---

# 36. Day 1 Mini Project

Don't build a complicated application yet.

Build:

## `EmployeeWebApp`

Initial architecture:

```text
EmployeeWebApp
│
├── src/main/java
│   └── com.example
│       └── servlet
│           └── HelloServlet.java
│
├── src/main/webapp
│   └── index.html
│
└── pom.xml
```

Your first goal is simply:

```text
Browser
   ↓
GET /hello
   ↓
Tomcat
   ↓
HelloServlet
   ↓
"Hello from J2EE"
```

Tomorrow, **Day 2**, we'll replace this simple request handling with a proper understanding and implementation of **Servlets, the Servlet lifecycle, `HttpServlet`, `doGet()`, `doPost()`, URL mapping, `web.xml`, annotations, request/response objects, and a complete CRUD-style Servlet application**.

### Day 1 mastery checkpoint

Before moving to Day 2, you should be able to draw this from memory:

```text
Client
  ↓
DNS
  ↓
TCP/TLS
  ↓
HTTP Request
  ↓
Web Server
  ↓
Servlet Container
  ↓
Servlet
  ↓
Service
  ↓
DAO
  ↓
Database
  ↓
HTTP Response
  ↓
Client
```

If you can explain **what happens at every arrow**, you've successfully completed the foundation for J2EE Day 1.
