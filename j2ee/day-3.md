# J2EE Mastery — Day 3

## Advanced Servlets: Request Processing, Dispatching, Filters, Listeners & Error Handling

**Day 3 — October 5, 2026**

Yesterday you learned the Servlet fundamentals:

```text
HTTP Request
     ↓
Servlet Container
     ↓
Servlet
     ↓
doGet()/doPost()
     ↓
HTTP Response
```

Today we'll turn that into a more realistic enterprise web layer:

```text
Client
  ↓
Filter
  ↓
Servlet
  ↓
Service
  ↓
DAO
  ↓
Response
```

The most important goal today is to understand **how requests travel through a Java web application** and how the Servlet container provides infrastructure around your application.

---

# 1. Day 3 Learning Objectives

By the end of today, you should understand:

* `RequestDispatcher`
* Forward vs redirect
* Include
* Request attributes
* `ServletContext`
* `ServletConfig`
* Filters
* `FilterChain`
* Filter ordering
* Listener APIs
* Application lifecycle
* Session lifecycle
* Request lifecycle
* Error handling
* Custom error pages
* Exception handling
* Cookies
* Basic file upload concepts
* Servlet security considerations
* Enterprise request-processing architecture

---

# 2. Request Processing Pipeline

A production-style Servlet application can look like:

```text
                     HTTP Request
                          │
                          ▼
                  Servlet Container
                          │
                          ▼
                    Authentication
                       Filter
                          │
                          ▼
                    Logging Filter
                          │
                          ▼
                    CORS Filter
                          │
                          ▼
                       Servlet
                          │
                          ▼
                       Service
                          │
                          ▼
                         DAO
                          │
                          ▼
                       Database
                          │
                          ▼
                     HTTP Response
```

Filters can intercept requests **before and after** the Servlet.

---

# 3. RequestDispatcher

`RequestDispatcher` allows server-side dispatching of a request to another resource.

Common operations:

```java
forward()
include()
```

Example:

```java
RequestDispatcher dispatcher =
        request.getRequestDispatcher("/home");

dispatcher.forward(request, response);
```

---

# 4. Forward

Suppose:

```text
GET /login
```

is handled by:

```text
LoginServlet
```

The Servlet can forward internally:

```text
LoginServlet
      ↓
HomeServlet
```

Example:

```java
RequestDispatcher dispatcher =
        request.getRequestDispatcher("/home");

dispatcher.forward(request, response);
```

Conceptually:

```text
Client
  │
  │ GET /login
  ▼
LoginServlet
  │
  │ forward()
  ▼
HomeServlet
  │
  ▼
Response
  │
  ▼
Client
```

The server performs the dispatch internally.

---

# 5. Forward vs Redirect

This is a very common interview question.

## Forward

```java
request.getRequestDispatcher("/home")
       .forward(request, response);
```

Flow:

```text
Client
  ↓
Server
  ↓
Servlet A
  ↓
Servlet B
  ↓
Response
```

The client doesn't make another HTTP request just because of the forward.

## Redirect

```java
response.sendRedirect("/home");
```

Flow:

```text
Client
  ↓
Server
  ↓
302/3xx Response
  ↓
Client
  ↓
New HTTP Request
  ↓
/home
```

### Key differences

| Forward                                | Redirect                            |
| -------------------------------------- | ----------------------------------- |
| Server-side dispatch                   | Client-side navigation              |
| Usually one client request             | Causes another client request       |
| Request attributes can be preserved    | Request attributes don't carry over |
| URL generally remains the original URL | Browser URL changes                 |
| Internal resources can be targeted     | New URL is requested                |

---

# 6. When Should You Use Forward?

Typical use:

```text
Servlet
   ↓
JSP
```

Example:

```java
request.setAttribute("employee", employee);

RequestDispatcher dispatcher =
        request.getRequestDispatcher(
                "/WEB-INF/views/employee.jsp"
        );

dispatcher.forward(request, response);
```

The Servlet prepares data.

The JSP renders it.

This is classic MVC.

---

# 7. When Should You Use Redirect?

A very common pattern is:

# Post/Redirect/Get

```text
POST /employees
      ↓
Create employee
      ↓
Redirect
      ↓
GET /employees
```

Why?

It prevents accidental form resubmission when the user refreshes the resulting page.

Example:

```java
response.sendRedirect(
        request.getContextPath() + "/employees"
);
```

---

# 8. `RequestDispatcher.include()`

`include()` allows one resource to include the output of another.

Example:

```java
RequestDispatcher dispatcher =
        request.getRequestDispatcher("/header");

dispatcher.include(request, response);
```

Conceptually:

```text
Servlet
 ├── Header
 ├── Main Content
 └── Footer
```

It is historically useful for composing server-side responses.

---

# 9. Request Attributes

Request attributes allow server-side components to share objects during the same request.

Set:

```java
request.setAttribute(
        "employee",
        employee
);
```

Retrieve:

```java
Employee employee =
        (Employee) request.getAttribute(
                "employee"
        );
```

Typical MVC flow:

```text
Servlet
   ↓
request.setAttribute()
   ↓
forward()
   ↓
JSP
   ↓
request.getAttribute()
```

---

# 10. Parameter vs Attribute

Memorize this distinction.

### Parameter

Usually originates from the client.

```java
request.getParameter("id");
```

Example:

```text
GET /employee?id=101
```

### Attribute

Usually created by server-side code.

```java
request.setAttribute("employee", employee);
```

Think:

```text
Parameter
Client → Server

Attribute
Server → Server
```

---

# 11. ServletContext

`ServletContext` represents the context of the web application.

Conceptually:

```text
             Web Application
                    │
             ServletContext
          ┌─────────┼─────────┐
          ▼         ▼         ▼
      Servlet A  Servlet B  Servlet C
```

Access it:

```java
ServletContext context =
        getServletContext();
```

---

# 12. ServletContext Attributes

You can store application-level attributes.

```java
context.setAttribute(
        "applicationName",
        "Employee Portal"
);
```

Retrieve:

```java
String name =
        (String) context.getAttribute(
                "applicationName"
        );
```

Because this context is shared across the application, be careful with mutable shared state and concurrency.

---

# 13. ServletConfig

`ServletConfig` contains configuration specific to a Servlet.

Example:

```java
ServletConfig config =
        getServletConfig();
```

Then:

```java
String environment =
        config.getInitParameter(
                "environment"
        );
```

---

# 14. ServletConfig vs ServletContext

Very important:

```text
ServletConfig
     ↓
Specific Servlet

ServletContext
     ↓
Entire Web Application
```

Example:

```text
Application
│
├── LoginServlet
│     └── ServletConfig
│
├── EmployeeServlet
│     └── ServletConfig
│
└── ServletContext
      ├── LoginServlet
      └── EmployeeServlet
```

---

# 15. Filters

A Filter is an object that can intercept requests and responses.

Basic flow:

```text
Request
   ↓
Filter
   ↓
Servlet
   ↓
Filter
   ↓
Response
```

A filter implements:

```java
Filter
```

---

# 16. Why Do We Need Filters?

Imagine you have 100 Servlets.

You need authentication on all of them.

Without filters:

```text
Servlet A → authentication
Servlet B → authentication
Servlet C → authentication
...
Servlet Z → authentication
```

That's duplication.

With a filter:

```text
                 Authentication Filter
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
         Servlet A   Servlet B   Servlet C
```

One cross-cutting component can protect many endpoints.

---

# 17. Common Filter Use Cases

Filters are useful for:

* Authentication
* Authorization checks
* Logging
* Request tracing
* CORS
* Character encoding
* Compression
* Security headers
* Request timing
* Audit logging
* Rate limiting
* Input preprocessing

---

# 18. Filter Lifecycle

A Filter has a lifecycle similar to:

```text
Create
  ↓
init()
  ↓
doFilter()
  ↓
doFilter()
  ↓
...
  ↓
destroy()
```

The container manages the lifecycle.

---

# 19. Creating a Filter

Example:

```java
@WebFilter("/*")
public class LoggingFilter implements Filter {

    @Override
    public void init(
            FilterConfig filterConfig)
            throws ServletException {

        System.out.println(
                "LoggingFilter initialized"
        );
    }

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        System.out.println(
                "Request received"
        );

        chain.doFilter(request, response);

        System.out.println(
                "Response completed"
        );
    }

    @Override
    public void destroy() {
        System.out.println(
                "LoggingFilter destroyed"
        );
    }
}
```

---

# 20. `FilterChain`

This is one of the most important concepts.

```java
chain.doFilter(
        request,
        response
);
```

It means:

> Continue processing the request through the remaining filters and eventually the target resource.

Example:

```text
Request
   ↓
Filter A
   ↓
Filter B
   ↓
Filter C
   ↓
Servlet
   ↓
Response
```

Filter A:

```java
chain.doFilter(request, response);
```

allows processing to continue.

---

# 21. What Happens If You Don't Call `chain.doFilter()`?

Suppose:

```java
if (!authenticated) {
    response.sendError(401);
    return;
}

chain.doFilter(request, response);
```

If authentication fails:

```text
Request
   ↓
Authentication Filter
   ↓
401 Response
```

The Servlet isn't called.

This is exactly how an authentication filter can stop unauthorized requests.

---

# 22. Authentication Filter

Example:

```java
@WebFilter("/api/*")
public class AuthenticationFilter
        implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest =
                (HttpServletRequest) request;

        HttpServletResponse httpResponse =
                (HttpServletResponse) response;

        HttpSession session =
                httpRequest.getSession(false);

        boolean authenticated =
                session != null &&
                session.getAttribute("user") != null;

        if (!authenticated) {
            httpResponse.sendError(
                    HttpServletResponse.SC_UNAUTHORIZED
            );
            return;
        }

        chain.doFilter(
                request,
                response
        );
    }
}
```

This is a simplified example; production authentication should use a proper security architecture rather than relying solely on ad-hoc session checks.

---

# 23. Filter Before and After Processing

A powerful pattern is:

```java
long start =
        System.nanoTime();

chain.doFilter(
        request,
        response
);

long duration =
        System.nanoTime() - start;
```

This allows you to measure request processing time.

Flow:

```text
Filter
  │
  ├── Before
  │
  ▼
Servlet
  │
  ├── Processing
  │
  ▼
Filter
  │
  └── After
```

This is useful for:

* Performance monitoring
* Logging
* Tracing
* Auditing

---

# 24. Multiple Filters

Suppose you have:

```text
AuthenticationFilter
LoggingFilter
AuditFilter
```

A request may travel like:

```text
Request
   ↓
AuthenticationFilter
   ↓
LoggingFilter
   ↓
AuditFilter
   ↓
Servlet
   ↓
AuditFilter
   ↓
LoggingFilter
   ↓
AuthenticationFilter
   ↓
Response
```

Think of filters as nested processing layers.

---

# 25. Filter Ordering

When multiple filters apply to a request, their ordering matters.

For example:

```text
Security
   ↓
Logging
   ↓
Application
```

versus:

```text
Logging
   ↓
Security
   ↓
Application
```

can produce different behavior and logs.

In modern Jakarta applications, ordering can be controlled through configuration mechanisms such as filter mappings.

---

# 26. Listeners

Listeners allow your application to react to lifecycle events.

There are several listener categories.

Important ones:

```text
ServletContextListener
HttpSessionListener
ServletRequestListener
```

---

# 27. ServletContextListener

Used for web application lifecycle events.

Example:

```java
@WebListener
public class ApplicationListener
        implements ServletContextListener {

    @Override
    public void contextInitialized(
            ServletContextEvent event) {

        System.out.println(
                "Application started"
        );
    }

    @Override
    public void contextDestroyed(
            ServletContextEvent event) {

        System.out.println(
                "Application stopped"
        );
    }
}
```

Flow:

```text
Application starts
       ↓
contextInitialized()
       ↓
Application runs
       ↓
contextDestroyed()
       ↓
Application stops
```

---

# 28. HttpSessionListener

Used to observe session creation/destruction.

```java
@WebListener
public class SessionListener
        implements HttpSessionListener {

    @Override
    public void sessionCreated(
            HttpSessionEvent event) {

        System.out.println(
                "Session created"
        );
    }

    @Override
    public void sessionDestroyed(
            HttpSessionEvent event) {

        System.out.println(
                "Session destroyed"
        );
    }
}
```

Potential use cases:

* Session monitoring
* Active-user metrics
* Resource cleanup
* Auditing

Be careful about storing global counters in application memory in a clustered environment.

---

# 29. ServletRequestListener

Observes request lifecycle.

```java
@WebListener
public class RequestListener
        implements ServletRequestListener {

    @Override
    public void requestInitialized(
            ServletRequestEvent event) {

        System.out.println(
                "Request started"
        );
    }

    @Override
    public void requestDestroyed(
            ServletRequestEvent event) {

        System.out.println(
                "Request completed"
        );
    }
}
```

---

# 30. Listener vs Filter

Important distinction.

### Filter

Intercepts request/response processing.

```text
Request
  ↓
Filter
  ↓
Servlet
```

### Listener

Observes lifecycle events.

```text
Application started
       ↓
Listener notified
```

Think:

```text
Filter   → intercept
Listener → observe
Servlet  → process
```

---

# 31. Error Handling

Servlet applications need controlled error handling.

Potential errors:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
500 Internal Server Error
```

Don't expose internal stack traces to users.

Bad:

```text
java.lang.NullPointerException
at com.company.dao.UserDAO...
```

Better:

```json
{
    "status": 500,
    "message": "Internal server error"
}
```

while logging diagnostic details internally.

---

# 32. `sendError()`

Example:

```java
response.sendError(
        HttpServletResponse.SC_NOT_FOUND,
        "Employee not found"
);
```

This tells the container that an error occurred.

---

# 33. Exception Handling

Servlet methods can throw:

```java
ServletException
IOException
```

Example:

```java
@Override
protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response)
        throws ServletException, IOException {

    // processing
}
```

For application-level errors, establish a consistent exception-handling strategy.

---

# 34. Error Pages

You can configure custom error pages.

Example:

```xml
<error-page>
    <error-code>404</error-code>
    <location>/error/404.jsp</location>
</error-page>
```

Another:

```xml
<error-page>
    <error-code>500</error-code>
    <location>/error/500.jsp</location>
</error-page>
```

Modern applications can also implement centralized API error handling rather than returning HTML error pages.

---

# 35. Cookies

Servlets provide APIs for cookies.

Create:

```java
Cookie cookie =
        new Cookie("theme", "dark");

response.addCookie(cookie);
```

Read:

```java
Cookie[] cookies =
        request.getCookies();
```

Then:

```java
for (Cookie cookie : cookies) {
    System.out.println(
            cookie.getName()
            + "="
            + cookie.getValue()
    );
}
```

---

# 36. Secure Cookie Attributes

For authentication/session cookies, understand:

```text
Secure
HttpOnly
SameSite
Path
Domain
Max-Age
```

### Secure

Cookie should be sent over HTTPS.

### HttpOnly

Helps prevent JavaScript from reading the cookie.

### SameSite

Controls cross-site cookie behavior and can reduce certain CSRF risks.

These attributes should be considered carefully for authentication cookies.

---

# 37. Session Cookie

A traditional Java web application may use a session cookie such as:

```text
JSESSIONID
```

Conceptually:

```text
Browser
   │
   │ JSESSIONID=ABC
   ▼
Servlet Container
   │
   ▼
Session
```

The session data itself is normally maintained server-side unless the application is configured otherwise.

---

# 38. Session Invalidation

When a user logs out:

```java
HttpSession session =
        request.getSession(false);

if (session != null) {
    session.invalidate();
}
```

This terminates the server-side session.

---

# 39. File Upload

Servlet applications can process multipart requests.

HTML:

```html
<form
    method="post"
    enctype="multipart/form-data"
    action="/upload">

    <input type="file" name="file">

    <button type="submit">
        Upload
    </button>
</form>
```

Servlet:

```java
@MultipartConfig
@WebServlet("/upload")
public class UploadServlet
        extends HttpServlet {

    @Override
    protected void doPost(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException,
            ServletException {

        Part file =
                request.getPart("file");

        String fileName =
                file.getSubmittedFileName();

        // Process safely
    }
}
```

---

# 40. File Upload Security

Never blindly trust:

```java
file.getSubmittedFileName()
```

Production considerations include:

* Maximum file size
* Allowed file types
* MIME validation
* Filename sanitization
* Virus/malware scanning
* Storage outside application deployment directory
* Access control
* Random server-generated filenames
* Object storage for large-scale systems

This becomes particularly important when you build distributed file-storage systems later.

---

# 41. Character Encoding

A filter is often used to consistently configure request encoding.

Conceptually:

```java
request.setCharacterEncoding("UTF-8");
response.setCharacterEncoding("UTF-8");
```

A filter can apply this across many endpoints.

---

# 42. MVC With Servlets

Now combine everything you've learned.

```text
                    Browser
                       │
                       ▼
                  HTTP Request
                       │
                       ▼
                Authentication
                    Filter
                       │
                       ▼
                Logging Filter
                       │
                       ▼
                EmployeeServlet
                       │
                       ▼
                EmployeeService
                       │
                       ▼
                 EmployeeDAO
                       │
                       ▼
                    Database
                       │
                       ▼
                 Employee Object
                       │
                       ▼
                Request Attribute
                       │
                       ▼
                  JSP / JSON
                       │
                       ▼
                   Response
```

This is the architecture you should begin thinking in.

---

# 43. Don't Put Everything in the Servlet

Bad architecture:

```java
doPost() {

    // authentication

    // validation

    // SQL

    // business rules

    // transaction

    // response generation

    // 500 lines...
}
```

Better:

```text
Servlet
   ↓
Service
   ↓
DAO
   ↓
Database
```

Servlet responsibilities:

* Receive request
* Parse input
* Validate basic request structure
* Invoke service
* Decide response/dispatch

Service responsibilities:

* Business rules
* Business transactions
* Business orchestration

DAO responsibilities:

* Database access

---

# 44. Complete Example

Let's create:

```text
GET /employees?id=100
```

### Servlet

```java
@WebServlet("/employees")
public class EmployeeServlet
        extends HttpServlet {

    private final EmployeeService service =
            new EmployeeService();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        String idParam =
                request.getParameter("id");

        if (idParam == null) {
            response.sendError(
                    HttpServletResponse.SC_BAD_REQUEST
            );
            return;
        }

        long id;

        try {
            id = Long.parseLong(idParam);
        } catch (NumberFormatException e) {
            response.sendError(
                    HttpServletResponse.SC_BAD_REQUEST
            );
            return;
        }

        Employee employee =
                service.findById(id);

        if (employee == null) {
            response.sendError(
                    HttpServletResponse.SC_NOT_FOUND
            );
            return;
        }

        request.setAttribute(
                "employee",
                employee
        );

        RequestDispatcher dispatcher =
                request.getRequestDispatcher(
                        "/WEB-INF/views/employee.jsp"
                );

        try {
            dispatcher.forward(
                    request,
                    response
            );
        } catch (ServletException e) {
            throw new IOException(e);
        }
    }
}
```

The important architecture is:

```text
HTTP
 ↓
Servlet
 ↓
Service
 ↓
Employee
 ↓
Request Attribute
 ↓
JSP
```

---

# 45. Day 3 Production Architecture

You should now understand this:

```text
                         Client
                           │
                           ▼
                      HTTP Request
                           │
                           ▼
                  ┌────────────────┐
                  │ Servlet        │
                  │ Container      │
                  └───────┬────────┘
                          │
                          ▼
                    Filter Chain
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Security      Logging       CORS
             │            │            │
             └────────────┼────────────┘
                          ▼
                       Servlet
                          │
                          ▼
                       Service
                          │
                          ▼
                         DAO
                          │
                          ▼
                      Database
                          │
                          ▼
                       Response
```

---

# 46. Request Lifecycle vs Application Lifecycle

Keep these separate.

### Application lifecycle

```text
Application Start
      ↓
ServletContext initialized
      ↓
Application running
      ↓
Servlets process requests
      ↓
Application shutdown
      ↓
ServletContext destroyed
```

### Request lifecycle

```text
Request
   ↓
Request initialization
   ↓
Filters
   ↓
Servlet
   ↓
Response
   ↓
Request destroyed
```

### Session lifecycle

```text
Login/session creation
        ↓
Session active
        ↓
Multiple requests
        ↓
Logout/timeout/invalidation
        ↓
Session destroyed
```

---

# 47. Three Lifecycle Concepts

Memorize:

```text
Application
    ↓
ServletContextListener

Session
    ↓
HttpSessionListener

Request
    ↓
ServletRequestListener
```

This is a frequent interview topic.

---

# 48. Day 3 Interview Questions

## Beginner

1. What is `RequestDispatcher`?
2. What is forwarding?
3. What is redirecting?
4. What is the difference between forward and redirect?
5. What is `include()`?
6. What is a request attribute?
7. What is `ServletContext`?
8. What is `ServletConfig`?
9. What is a Filter?
10. What is `FilterChain`?

## Intermediate

11. Why are Filters useful?
12. What happens if `chain.doFilter()` isn't called?
13. How can a Filter implement authentication?
14. What is a Listener?
15. What is `ServletContextListener`?
16. What is `HttpSessionListener`?
17. What is `ServletRequestListener`?
18. Filter vs Listener?
19. Parameter vs Attribute?
20. ServletConfig vs ServletContext?

## Advanced

21. Explain the complete Filter → Servlet request flow.
22. How does a Servlet container manage multiple Filters?
23. How would you implement centralized request logging?
24. How would you implement authentication using a Filter?
25. How would you prevent unauthorized access to `/admin/*`?
26. How does forward differ from redirect internally?
27. Why is Post/Redirect/Get useful?
28. How would you implement global error handling?
29. How would you safely implement file uploads?
30. What thread-safety issues can arise with `ServletContext` attributes?

---

# 49. Day 3 Coding Exercises

Complete these in order.

### Exercise 1 — Forward

Create:

```text
/login
/home
```

Forward from `/login` to `/home`.

---

### Exercise 2 — Redirect

Create:

```text
/old
/new
```

Redirect `/old` → `/new`.

Observe the browser URL.

---

### Exercise 3 — Include

Create:

```text
/header
/content
/footer
```

Build a response by including all three.

---

### Exercise 4 — Request Attribute

Create:

```text
/user
```

Set:

```text
name
email
```

as request attributes and forward to a JSP.

---

### Exercise 5 — Logging Filter

Log:

```text
HTTP method
URI
start time
end time
duration
status
```

---

### Exercise 6 — Authentication Filter

Protect:

```text
/admin/*
```

Unauthenticated users should receive:

```text
401
```

---

### Exercise 7 — Application Listener

Print:

```text
Application started
Application stopped
```

using `ServletContextListener`.

---

### Exercise 8 — Session Listener

Track:

```text
Session created
Session destroyed
```

---

### Exercise 9 — Custom Error Page

Create custom pages for:

```text
404
500
```

---

### Exercise 10 — File Upload

Create:

```text
POST /upload
```

and implement basic file validation.

---

# 50. Day 3 Mini Project

Upgrade yesterday's Employee application.

### Version 2

```text
EmployeeWebApp
│
├── Filter
│   ├── LoggingFilter
│   └── AuthenticationFilter
│
├── Servlet
│   ├── LoginServlet
│   ├── LogoutServlet
│   └── EmployeeServlet
│
├── Service
│   └── EmployeeService
│
├── Repository
│   └── EmployeeRepository
│
└── JSP
    ├── login.jsp
    ├── employees.jsp
    └── employee.jsp
```

Request flow:

```text
Browser
   ↓
AuthenticationFilter
   ↓
LoggingFilter
   ↓
EmployeeServlet
   ↓
EmployeeService
   ↓
Repository
   ↓
In-memory Map
```

Don't introduce JDBC yet. That's coming later.

---

# 51. Day 3 Mastery Test

Without looking at notes, explain this diagram:

```text
                         Request
                            │
                            ▼
                    Servlet Container
                            │
                            ▼
                      Filter Chain
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
             Security     Logging      CORS
                │           │           │
                └───────────┼───────────┘
                            ▼
                         Servlet
                            │
                     ┌──────┴──────┐
                     ▼             ▼
                 Service       RequestDispatcher
                     │             │
                     ▼             ▼
                    DAO           JSP
                     │
                     ▼
                  Database
```

You should be able to answer:

* Who manages the Servlet?
* Who manages the Filter?
* Why do we need `FilterChain`?
* What happens if a filter doesn't call `chain.doFilter()`?
* What is the difference between forward and redirect?
* Where do request attributes live?
* What is `ServletContext`?
* What is `ServletConfig`?
* When does a `ServletContextListener` execute?
* When does an `HttpSessionListener` execute?
* Where should authentication logic live?
* Where should business logic live?
* Where should SQL live?

---

# 52. Day 3 Cheat Sheet

```text
Servlet
→ Processes requests

Filter
→ Intercepts requests/responses

FilterChain
→ Continues filter processing

Listener
→ Observes lifecycle events

RequestDispatcher
→ Server-side dispatching

forward()
→ Internal server dispatch

sendRedirect()
→ Client makes another request

include()
→ Include another resource's output

ServletConfig
→ Servlet-specific configuration

ServletContext
→ Application-wide context

Parameter
→ Usually client-provided input

Attribute
→ Server-side request/application data

HttpSession
→ Server-side session state

Cookie
→ Client-side state sent with requests
```

---

# 53. Your Day 3 Mental Model

The most important thing to internalize today is:

```text
                         WEB APPLICATION
                               │
                               ▼
                         HTTP Request
                               │
                               ▼
                      Servlet Container
                               │
                               ▼
                         Filter Chain
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
         Authentication     Logging          CORS
               │               │               │
               └───────────────┼───────────────┘
                               ▼
                            Servlet
                               │
                               ▼
                            Service
                               │
                               ▼
                              DAO
                               │
                               ▼
                           Database
                               │
                               ▼
                         HTTP Response
```

If **Day 2 was about understanding a Servlet**, **Day 3 is about understanding everything surrounding the Servlet**.

That foundation is extremely important because when you later learn **Spring MVC/Spring Boot**, many concepts will become much easier: Spring's `DispatcherServlet`, filters, interceptors, controllers, dependency injection, exception handling, sessions, and security all build on or interact with the same fundamental HTTP request-processing model.
