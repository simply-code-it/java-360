# J2EE Mastery — Day 2

## Servlets Fundamentals

**Day 2 — October 4, 2026**

Today you move from **HTTP theory → actual Java server-side programming**.

By the end of Day 2, you should be able to build a Servlet application, understand the Servlet lifecycle, receive HTTP requests, generate responses, map URLs, and explain what the Servlet container does internally.

---

# 1. What Is a Servlet?

A **Servlet** is a Java class that runs inside a Servlet container and handles client requests, typically HTTP requests.

Simplified:

```text
Browser
   │
   │ HTTP Request
   ▼
Servlet Container
   │
   ▼
Servlet
   │
   │ Java code
   ▼
HTTP Response
   │
   ▼
Browser
```

A Servlet can:

* Receive HTTP requests
* Read request parameters
* Read headers
* Read cookies
* Access sessions
* Execute business logic
* Call services/DAOs
* Generate responses
* Return HTML/JSON/XML
* Redirect clients

---

# 2. Why Do We Need Servlets?

Before Servlets, Java developers had to deal with lower-level server programming.

Servlets provide an abstraction:

```text
HTTP
 ↓
Servlet Container
 ↓
Servlet
 ↓
Java Application
```

Instead of manually handling:

* sockets
* HTTP parsing
* threads
* request routing
* response generation

the container manages much of the infrastructure.

Your application focuses on the request-processing logic.

---

# 3. Servlet API

Historically:

```java
javax.servlet.*
javax.servlet.http.*
```

Modern Jakarta EE:

```java
jakarta.servlet.*
jakarta.servlet.http.*
```

For modern Jakarta EE applications, you'll commonly see:

```java
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
```

For legacy Java EE/J2EE applications, you will frequently encounter `javax.servlet.*`.

This distinction is important when maintaining older enterprise applications.

---

# 4. Servlet Hierarchy

The basic hierarchy is:

```text
Servlet
  │
  ▼
GenericServlet
  │
  ▼
HttpServlet
```

### Servlet

The fundamental interface.

### GenericServlet

Provides a protocol-independent implementation.

### HttpServlet

Designed specifically for HTTP.

For normal web applications, you'll usually extend:

```java
HttpServlet
```

---

# 5. Your First Servlet

Example:

```java
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("text/plain");

        response.getWriter().println("Hello from Servlet!");
    }
}
```

Request:

```text
GET /hello
```

Response:

```text
Hello from Servlet!
```

---

# 6. Understanding `HttpServlet`

Most HTTP Servlets extend:

```java
HttpServlet
```

It provides methods such as:

```java
doGet()
doPost()
doPut()
doDelete()
doHead()
doOptions()
doTrace()
```

The most commonly used are:

```java
doGet()
doPost()
doPut()
doDelete()
```

---

# 7. `doGet()`

Used primarily to handle GET requests.

```java
@Override
protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response)
        throws IOException {

    response.setContentType("text/plain");

    response.getWriter().println("GET request received");
}
```

Client:

```http
GET /hello
```

Container eventually dispatches the request to:

```text
doGet()
```

---

# 8. `doPost()`

Used to handle POST requests.

```java
@Override
protected void doPost(
        HttpServletRequest request,
        HttpServletResponse response)
        throws IOException {

    response.setContentType("text/plain");

    response.getWriter().println("POST request received");
}
```

Request:

```http
POST /users
```

The Servlet container routes the request to:

```text
doPost()
```

---

# 9. Servlet URL Mapping

The container needs to know:

> Which Servlet should process this URL?

For example:

```text
GET /hello
```

should invoke:

```text
HelloServlet
```

There are two major approaches.

### Annotation

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
}
```

### Deployment descriptor

```xml
<servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.example.HelloServlet</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>HelloServlet</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```

Modern applications commonly prefer annotations for simple configuration.

---

# 10. `@WebServlet`

Example:

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        response.getWriter().println("Hello");
    }
}
```

Now:

```text
http://localhost:8080/myapp/hello
```

can be routed to `HelloServlet`.

---

# 11. Multiple URL Patterns

You can map multiple URLs.

```java
@WebServlet({
    "/hello",
    "/welcome",
    "/greeting"
})
public class GreetingServlet extends HttpServlet {
}
```

All three can map to the same Servlet.

---

# 12. URL Patterns

You can use patterns such as:

```text
/hello
/users/*
*.action
```

Examples:

```java
@WebServlet("/users/*")
```

matches paths under:

```text
/users/1
/users/10
/users/admin
```

A mapping such as:

```java
@WebServlet("*.action")
```

can match extension-style URLs.

---

# 13. Servlet Lifecycle

This is **one of the most important Day 2 topics**.

Servlet lifecycle:

```text
Servlet Class
     ↓
Class Loading
     ↓
Servlet Instantiation
     ↓
init()
     ↓
service()
     ↓
doGet()/doPost()
     ↓
destroy()
```

The container manages this lifecycle.

---

# 14. `init()`

Called when the Servlet is initialized.

Example:

```java
@Override
public void init() throws ServletException {
    System.out.println("Servlet initialized");
}
```

Typical uses:

* initialization
* loading configuration
* preparing resources

Avoid putting request-specific data here.

---

# 15. `service()`

`HttpServlet` implements HTTP dispatching.

Conceptually:

```text
HTTP Request
     ↓
service()
     ↓
┌────┴─────┐
▼          ▼
GET       POST
▼          ▼
doGet()   doPost()
```

You generally don't override `service()` when ordinary `doGet()`/`doPost()` handling is sufficient.

---

# 16. `destroy()`

Called when the container removes the Servlet.

```java
@Override
public void destroy() {
    System.out.println("Servlet destroyed");
}
```

Used for cleanup.

---

# 17. Important Lifecycle Interview Question

### Does `init()` execute for every request?

**No.**

Typically:

```text
Servlet created
     ↓
init()       ← once
     ↓
service()    ← many requests
     ↓
destroy()    ← once
```

The exact timing of initialization can depend on configuration/container behavior, including load-on-startup.

---

# 18. Lazy Initialization

Servlets may be initialized when the first request arrives.

Conceptually:

```text
Application starts
       ↓
Servlet not yet initialized
       ↓
First request
       ↓
Create Servlet
       ↓
init()
       ↓
service()
```

---

# 19. Load on Startup

You can configure a Servlet to initialize when the application starts.

Historically in `web.xml`:

```xml
<servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.example.HelloServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
```

The exact ordering semantics are based on the configured values.

---

# 20. `HttpServletRequest`

This object represents the incoming HTTP request.

Example:

```java
protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response)
        throws IOException {
}
```

You can retrieve:

* Parameters
* Headers
* Cookies
* URI
* HTTP method
* Session
* Request attributes
* Query string
* Request body

---

# 21. Request Parameters

Request:

```text
/users?id=101
```

Read it:

```java
String id = request.getParameter("id");
```

Result:

```text
101
```

---

# 22. Multiple Parameters

Request:

```text
/users?name=John&age=30
```

Code:

```java
String name = request.getParameter("name");
String age = request.getParameter("age");
```

You can convert:

```java
int age = Integer.parseInt(
        request.getParameter("age")
);
```

In production code, validate the value before parsing.

---

# 23. `getParameter()` vs `getAttribute()`

Very important.

### Parameter

Comes from the client request.

```java
request.getParameter("name");
```

Example:

```text
/users?name=John
```

### Attribute

Usually placed into the request by server-side application code.

```java
request.setAttribute("user", user);
```

Retrieve:

```java
User user =
        (User) request.getAttribute("user");
```

Think:

```text
Parameter
   ↓
Client → Server

Attribute
   ↓
Server → Server
```

---

# 24. Request URI

You can retrieve:

```java
String uri = request.getRequestURI();
```

For:

```text
/myapp/users/101
```

you can get the request URI.

Other useful methods include:

```java
request.getMethod();
request.getQueryString();
request.getContextPath();
request.getServletPath();
```

---

# 25. HTTP Method

```java
String method = request.getMethod();
```

For:

```http
GET /users
```

result:

```text
GET
```

For:

```http
POST /users
```

result:

```text
POST
```

---

# 26. Request Headers

Example:

```http
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer token
```

Read:

```java
String userAgent =
        request.getHeader("User-Agent");
```

Or:

```java
String authorization =
        request.getHeader("Authorization");
```

---

# 27. Enumerating Headers

You can inspect headers:

```java
Enumeration<String> headers =
        request.getHeaderNames();

while (headers.hasMoreElements()) {

    String name = headers.nextElement();

    String value = request.getHeader(name);

    System.out.println(name + ": " + value);
}
```

Useful for debugging and understanding HTTP.

---

# 28. `HttpServletResponse`

Represents the response sent back to the client.

You can control:

* Status code
* Headers
* Content type
* Response body
* Cookies
* Redirects

Example:

```java
response.setStatus(
        HttpServletResponse.SC_OK
);
```

---

# 29. Setting Content Type

For plain text:

```java
response.setContentType("text/plain");
```

For HTML:

```java
response.setContentType("text/html");
```

For JSON:

```java
response.setContentType("application/json");
```

You can also specify encoding where appropriate:

```java
response.setContentType(
        "text/plain;charset=UTF-8"
);
```

---

# 30. Writing the Response

```java
PrintWriter writer =
        response.getWriter();

writer.println("Hello World");
```

Or:

```java
response.getWriter()
        .println("Hello World");
```

---

# 31. Setting Status Codes

Example:

```java
response.setStatus(
        HttpServletResponse.SC_CREATED
);
```

Equivalent conceptually to:

```http
201 Created
```

Other constants include:

```java
SC_OK
SC_CREATED
SC_BAD_REQUEST
SC_UNAUTHORIZED
SC_FORBIDDEN
SC_NOT_FOUND
SC_INTERNAL_SERVER_ERROR
```

---

# 32. Redirect

You can redirect a client:

```java
response.sendRedirect("/login");
```

Conceptually:

```text
Client
  ↓
Request
  ↓
Server
  ↓
302/redirect response
  ↓
Client
  ↓
New request
  ↓
/login
```

A redirect causes the client to make another request.

---

# 33. Forward

Forwarding is different.

The server can forward internally:

```java
RequestDispatcher dispatcher =
        request.getRequestDispatcher(
                "/WEB-INF/views/home.jsp"
        );

dispatcher.forward(request, response);
```

Conceptually:

```text
Client
  ↓
Servlet
  ↓
JSP
  ↓
Response
```

The browser doesn't make a new HTTP request merely because the server forwarded internally.

You'll study this deeply on Day 3.

---

# 34. Servlet Threading Model

This is **critical for production Java development**.

A Servlet container commonly handles multiple requests concurrently.

Conceptually:

```text
                 Servlet
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Request 1    Request 2    Request 3
    Thread A     Thread B     Thread C
```

This means you must be careful with shared mutable state.

---

# 35. Dangerous Servlet Code

Don't do this:

```java
public class CounterServlet extends HttpServlet {

    private int counter = 0;

    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response) {

        counter++;
    }
}
```

Multiple requests can access `counter` concurrently.

This creates concurrency problems.

---

# 36. Better Approach

Keep request-specific state in local variables:

```java
protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response) {

    int counter = 0;

    // request-specific processing
}
```

For shared state, use appropriate concurrency-safe mechanisms and preferably external state stores where appropriate.

This connects directly to the concurrency and distributed-system concepts you'll study later.

---

# 37. Servlet Instance vs Request

A common interview misconception is:

> One Servlet object is created for every request.

Generally, **no**.

A container typically creates/manages a Servlet instance and invokes it concurrently for multiple requests.

Conceptually:

```text
              One Servlet
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
    Thread 1    Thread 2    Thread 3
       │           │           │
   Request A    Request B    Request C
```

Therefore:

**Don't store request-specific mutable state in Servlet instance fields.**

---

# 38. ServletConfig

`ServletConfig` provides configuration information for an individual Servlet.

Conceptually:

```text
Servlet
   ↓
ServletConfig
   ↓
Servlet-specific configuration
```

Example:

```java
ServletConfig config =
        getServletConfig();
```

Then:

```java
String value =
        config.getInitParameter("environment");
```

---

# 39. ServletContext

`ServletContext` represents the web application context.

Conceptually:

```text
Web Application
       │
       ▼
ServletContext
   ┌───┼────┐
   ▼   ▼    ▼
Servlet Servlet Servlet
```

You can access it using:

```java
ServletContext context =
        getServletContext();
```

It can be used for application-wide resources and attributes.

---

# 40. ServletConfig vs ServletContext

Memorize this:

| ServletConfig               | ServletContext                    |
| --------------------------- | --------------------------------- |
| Specific Servlet            | Entire web application            |
| Servlet-level configuration | Application-level context         |
| One per Servlet             | Typically one per web application |

---

# 41. Your First Real Servlet

Let's create:

```text
GET /hello?name=Pradeep
```

Response:

```text
Hello, Pradeep!
```

Code:

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        String name =
                request.getParameter("name");

        if (name == null || name.isBlank()) {
            name = "Guest";
        }

        response.setContentType(
                "text/plain;charset=UTF-8"
        );

        response.getWriter()
                .println("Hello, " + name + "!");
    }
}
```

Request:

```text
http://localhost:8080/myapp/hello?name=Pradeep
```

Response:

```text
Hello, Pradeep!
```

---

# 42. POST Example

HTML:

```html
<form method="post" action="register">
    <input name="name">
    <input name="email">
    <button type="submit">
        Register
    </button>
</form>
```

Servlet:

```java
@WebServlet("/register")
public class RegistrationServlet extends HttpServlet {

    @Override
    protected void doPost(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        String name =
                request.getParameter("name");

        String email =
                request.getParameter("email");

        response.setContentType("text/plain");

        response.getWriter().println(
                "Registered: " + name + ", " + email
        );
    }
}
```

---

# 43. Servlet Lifecycle Example

Write this and run it:

```java
@WebServlet("/lifecycle")
public class LifecycleServlet extends HttpServlet {

    @Override
    public void init() throws ServletException {
        System.out.println("INIT");
    }

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        System.out.println("REQUEST");

        response.setContentType("text/plain");

        response.getWriter()
                .println("Lifecycle test");
    }

    @Override
    public void destroy() {
        System.out.println("DESTROY");
    }
}
```

You should observe something conceptually like:

```text
INIT

REQUEST
REQUEST
REQUEST

DESTROY
```

Notice:

```text
init()    → once
doGet()   → many times
destroy() → once
```

---

# 44. Servlet Container Request Processing

Understand this flow deeply:

```text
HTTP Request
     │
     ▼
Servlet Container
     │
     ├── Parse request
     │
     ├── Identify application
     │
     ├── Match URL
     │
     ├── Find Servlet
     │
     ├── Create/manage request
     │
     ├── Create/manage response
     │
     ▼
Servlet.service()
     │
     ▼
HttpServlet
     │
     ├── GET  → doGet()
     ├── POST → doPost()
     ├── PUT  → doPut()
     └── DELETE → doDelete()
```

---

# 45. Complete Day 2 Architecture

You should now visualize:

```text
                    Browser
                       │
                       │ HTTP
                       ▼
                Servlet Container
                       │
                URL Mapping
                       │
                       ▼
                 HelloServlet
                       │
                 ┌─────┴─────┐
                 │           │
          HttpServletRequest  │
                 │           │
                 └─────┬─────┘
                       ▼
                  Java Logic
                       │
                       ▼
               HttpServletResponse
                       │
                       ▼
                    Browser
```

---

# 46. Production Rules for Servlets

Start developing these habits immediately.

### Rule 1 — Don't put business logic directly in Servlets

Avoid:

```java
doPost() {
    // 500 lines of business logic
}
```

Prefer:

```text
Servlet
   ↓
Service
   ↓
DAO
```

---

### Rule 2 — Don't store request state in fields

Bad:

```java
private String username;
```

Better:

```java
String username =
        request.getParameter("username");
```

---

### Rule 3 — Validate input

Never blindly trust:

```java
request.getParameter(...)
```

Validate:

* null
* empty
* length
* format
* range
* authorization

---

### Rule 4 — Don't build SQL with user input

Never:

```java
String sql =
    "SELECT * FROM users WHERE name='"
    + name
    + "'";
```

You'll learn proper JDBC practices on Day 9.

---

### Rule 5 — Don't expose internal errors

Avoid returning:

```text
NullPointerException at UserDAO.java:42
```

to users.

Return a safe error response and log diagnostic details internally.

---

# 47. Day 2 Interview Questions

### Beginner

1. What is a Servlet?
2. Why do we need Servlets?
3. What is a Servlet container?
4. What is `HttpServlet`?
5. What is `GenericServlet`?
6. What is `doGet()`?
7. What is `doPost()`?
8. What is `@WebServlet`?
9. What is `web.xml`?
10. What is Servlet mapping?

### Intermediate

11. Explain the Servlet lifecycle.
12. When is `init()` called?
13. How many times is `init()` called?
14. When is `destroy()` called?
15. What is `service()`?
16. How does `service()` dispatch HTTP requests?
17. What is `HttpServletRequest`?
18. What is `HttpServletResponse`?
19. What is `ServletConfig`?
20. What is `ServletContext`?

### Advanced

21. Is one Servlet object created for every request?
22. Are Servlets thread-safe by default?
23. What happens when 100 users access the same Servlet simultaneously?
24. Why should instance variables be treated carefully in Servlets?
25. What is lazy Servlet initialization?
26. What is `load-on-startup`?
27. What is the difference between redirect and forward?
28. What is the difference between request parameters and attributes?
29. What happens internally when a request reaches a Servlet?
30. How does a Servlet container manage concurrent requests?

---

# 48. Day 2 Coding Challenges

Complete these **10 exercises**.

### Exercise 1 — Hello Servlet

```text
GET /hello
```

Return:

```text
Hello World
```

### Exercise 2 — Name

```text
GET /hello?name=Pradeep
```

Return:

```text
Hello Pradeep
```

### Exercise 3 — Calculator

```text
GET /calculate?a=10&b=20&operation=add
```

Return:

```text
30
```

Support:

```text
add
subtract
multiply
divide
```

### Exercise 4 — User Registration

Create:

```text
POST /register
```

Parameters:

```text
name
email
password
```

Validate all fields.

### Exercise 5 — HTTP Method Handler

Create a Servlet that responds differently to:

```text
GET
POST
PUT
DELETE
```

### Exercise 6 — Request Inspector

Display:

```text
Method
URI
Query String
User-Agent
Content-Type
```

### Exercise 7 — Status Codes

Create endpoints returning:

```text
200
201
400
401
403
404
500
```

### Exercise 8 — Redirect

Create:

```text
GET /old
```

that redirects to:

```text
/new
```

### Exercise 9 — Lifecycle

Print:

```text
INIT
REQUEST
DESTROY
```

and verify the lifecycle experimentally.

### Exercise 10 — Thread Safety

Create a counter Servlet and intentionally demonstrate what happens when multiple requests modify shared state.

Then redesign it safely.

---

# 49. Day 2 Mini Project

Build:

# Employee Servlet Application — Version 1

Endpoints:

```text
GET  /employees
GET  /employees?id=101
POST /employees
PUT  /employees
DELETE /employees?id=101
```

For now, **don't connect a database**.

Use an in-memory collection:

```java
Map<Long, Employee>
```

Architecture:

```text
Browser/Postman
       │
       ▼
EmployeeServlet
       │
       ▼
EmployeeService
       │
       ▼
InMemoryEmployeeRepository
```

This gives you the foundation you'll later replace with:

```text
Servlet
   ↓
Service
   ↓
DAO
   ↓
JDBC
   ↓
Database
```

---

# 50. Day 2 Mastery Checklist

Before moving to Day 3, make sure you can explain:

### Servlet fundamentals

* [ ] What is a Servlet?
* [ ] Why Servlet containers exist
* [ ] Servlet hierarchy
* [ ] `HttpServlet`
* [ ] `doGet()`
* [ ] `doPost()`
* [ ] URL mapping
* [ ] `@WebServlet`
* [ ] `web.xml`

### Lifecycle

* [ ] Servlet instantiation
* [ ] `init()`
* [ ] `service()`
* [ ] `doGet()`
* [ ] `doPost()`
* [ ] `destroy()`
* [ ] Lazy initialization
* [ ] Load-on-startup

### Request/Response

* [ ] `HttpServletRequest`
* [ ] `HttpServletResponse`
* [ ] Parameters
* [ ] Attributes
* [ ] Headers
* [ ] URI
* [ ] Query string
* [ ] Status codes
* [ ] Content type
* [ ] Response body

### Enterprise concepts

* [ ] Servlet threading
* [ ] Shared mutable state
* [ ] `ServletConfig`
* [ ] `ServletContext`
* [ ] Redirect
* [ ] Forward

---

# The Most Important Mental Model of Day 2

Memorize this:

```text
                   HTTP Request
                        │
                        ▼
              ┌───────────────────┐
              │ Servlet Container │
              └─────────┬─────────┘
                        │
                  URL Mapping
                        │
                        ▼
                 Servlet Instance
                        │
                        ▼
                    service()
                        │
             ┌──────────┼──────────┐
             │          │          │
            GET        POST       PUT
             │          │          │
             ▼          ▼          ▼
          doGet()    doPost()    doPut()
             │          │          │
             └──────────┼──────────┘
                        │
                        ▼
                Business Layer
                        │
                        ▼
               HTTP Response
                        │
                        ▼
                     Client
```

**The key lesson:** a Servlet is not the entire application. It is the **HTTP/web entry point into your Java application**, managed by the Servlet container.

**Day 3** will build on this with **advanced Servlets: `RequestDispatcher`, forward vs redirect, filters, request attributes, `ServletContext`, `ServletConfig`, listeners, error handling, multipart/file upload, cookies, and practical MVC architecture**.
