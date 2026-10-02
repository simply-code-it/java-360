# J2EE Mastery — Day 5

## Filters, Listeners & Enterprise Request Processing

**Date:** October 7, 2026
**Day 5 Goal:** Master Servlet **Filters, Filter Chains, Listeners, lifecycle events, cross-cutting concerns, request interception, auditing, logging, security, metrics, and production request pipelines**.

Since Day 3 introduced the basics of Filters and Listeners, today goes **much deeper into how to design and use them in enterprise applications**.

---

# 1. What You'll Master Today

By the end of Day 5, you should understand:

1. What cross-cutting concerns are
2. Servlet Filters
3. Filter lifecycle
4. `FilterConfig`
5. `FilterChain`
6. `doFilter()`
7. Before/after processing
8. Multiple filters
9. Filter ordering
10. URL-based filter mapping
11. Dispatcher types
12. `REQUEST`
13. `FORWARD`
14. `INCLUDE`
15. `ERROR`
16. `ASYNC`
17. Authentication filters
18. Authorization filters
19. Logging filters
20. Timing/metrics filters
21. CORS filters
22. Security-header filters
23. Encoding filters
24. Audit filters
25. Servlet listeners
26. Application lifecycle listeners
27. Session lifecycle listeners
28. Request lifecycle listeners
29. Attribute listeners
30. Listener vs Filter
31. Thread-safety considerations
32. Error handling
33. Production filter architecture
34. Mini project

---

# 2. The Problem Filters Solve

Imagine you have:

```text
DashboardServlet
OrderServlet
PaymentServlet
ProfileServlet
EmployeeServlet
ReportServlet
```

You need:

```text
Authentication
Logging
Authorization
Request ID
Performance measurement
CORS
Security headers
Audit logging
```

Without filters, you might write this repeatedly:

```java
authenticate();

logRequest();

checkAuthorization();

addSecurityHeaders();

startTimer();

processBusinessLogic();

stopTimer();
```

in every servlet.

That's bad architecture.

---

# 3. Cross-Cutting Concerns

A **cross-cutting concern** affects many parts of an application.

Examples:

```text
Security
Logging
Monitoring
Auditing
Transactions
Caching
Exception handling
Tracing
Request correlation
```

Instead of:

```text
Servlet A → authentication
Servlet B → authentication
Servlet C → authentication
```

we want:

```text
                 AuthenticationFilter
                         |
             ┌───────────┼───────────┐
             ↓           ↓           ↓
          Servlet A   Servlet B   Servlet C
```

This is one of the most important reasons Filters exist.

---

# 4. What Is a Servlet Filter?

A Filter intercepts requests and/or responses before they reach the target resource and/or after that resource returns.

Conceptually:

```text
Client
  |
  ↓
Filter
  |
  ↓
Servlet
  |
  ↓
Filter
  |
  ↓
Client
```

A filter can:

```text
inspect
modify
allow
reject
log
authenticate
authorize
measure
wrap
```

the request or response.

---

# 5. Filter Interface

Modern Jakarta Servlet:

```java
import jakarta.servlet.Filter;
```

A filter implements:

```java
public interface Filter {

    void init(FilterConfig filterConfig);

    void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain
    );

    void destroy();
}
```

---

# 6. Basic Filter

```java
@WebFilter("/*")
public class LoggingFilter implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        System.out.println("Request received");

        chain.doFilter(request, response);

        System.out.println("Response completed");
    }
}
```

The key line is:

```java
chain.doFilter(request, response);
```

---

# 7. Understanding `FilterChain`

Suppose:

```text
Filter A
Filter B
Filter C
Servlet
```

The execution looks like:

```text
Filter A
   |
   ↓
Filter B
   |
   ↓
Filter C
   |
   ↓
Servlet
```

When the servlet finishes:

```text
Servlet
   |
   ↓
Filter C
   |
   ↓
Filter B
   |
   ↓
Filter A
```

This creates a chain.

---

# 8. The Most Important Filter Concept

Think:

```java
before();

chain.doFilter(request, response);

after();
```

Example:

```java
@Override
public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain)
        throws IOException, ServletException {

    long start = System.nanoTime();

    // BEFORE
    System.out.println("Before servlet");

    chain.doFilter(request, response);

    // AFTER
    long duration =
            System.nanoTime() - start;

    System.out.println(
            "Request took " + duration + " ns"
    );
}
```

This pattern is extremely important.

---

# 9. What If We Don't Call `chain.doFilter()`?

Then processing stops.

Example:

```java
if (!authenticated) {

    response.sendError(
            HttpServletResponse.SC_UNAUTHORIZED
    );

    return;
}
```

No:

```java
chain.doFilter(...)
```

means:

```text
Filter
  ↓
STOP
```

The servlet isn't executed.

This is how authentication filters can block unauthorized requests.

---

# 10. Authentication Filter

```java
@WebFilter("/protected/*")
public class AuthenticationFilter
        implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest req =
                (HttpServletRequest) request;

        HttpServletResponse res =
                (HttpServletResponse) response;

        HttpSession session =
                req.getSession(false);

        boolean authenticated =
                session != null &&
                session.getAttribute("userId")
                        != null;

        if (!authenticated) {

            res.sendRedirect(
                    req.getContextPath()
                            + "/login"
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

---

# 11. Filter Lifecycle

A filter has a lifecycle:

```text
Filter class loaded
       ↓
Object created
       ↓
init()
       ↓
doFilter()
       ↓
doFilter()
       ↓
doFilter()
       ↓
destroy()
```

Important:

```text
init()       → once
doFilter()   → many times
destroy()    → once
```

---

# 12. `init()`

Use `init()` for initialization.

```java
@Override
public void init(FilterConfig filterConfig) {

    System.out.println(
            "Filter initialized"
    );
}
```

For example:

```text
load configuration
initialize metrics
initialize reusable resources
```

Don't put per-request logic here.

---

# 13. `destroy()`

Called when the filter is being taken out of service.

```java
@Override
public void destroy() {

    System.out.println(
            "Filter destroyed"
    );
}
```

Useful for cleanup.

---

# 14. FilterConfig

`FilterConfig` provides filter configuration.

```java
@Override
public void init(
        FilterConfig filterConfig) {

    String environment =
            filterConfig.getInitParameter(
                    "environment"
            );
}
```

---

# 15. Configuring Filter Parameters

Using annotations:

```java
@WebFilter(
    value = "/*",
    initParams = {
        @WebInitParam(
            name = "environment",
            value = "production"
        )
    }
)
```

Then:

```java
String environment =
        filterConfig.getInitParameter(
                "environment"
        );
```

---

# 16. `web.xml` Filter Configuration

Legacy applications commonly use:

```xml
<filter>
    <filter-name>LoggingFilter</filter-name>

    <filter-class>
        com.example.filter.LoggingFilter
    </filter-class>
</filter>

<filter-mapping>
    <filter-name>LoggingFilter</filter-name>

    <url-pattern>/*</url-pattern>
</filter-mapping>
```

You should know both:

```text
@WebFilter
web.xml
```

because enterprise applications often contain legacy configuration.

---

# 17. URL Pattern

You can restrict a filter.

```java
@WebFilter("/admin/*")
```

Only:

```text
/admin/users
/admin/reports
/admin/settings
```

are intercepted.

Another:

```java
@WebFilter("/api/*")
```

protects API endpoints.

---

# 18. Servlet Name Mapping

Filters can also be mapped to specific servlets.

Conceptually:

```xml
<filter-mapping>

    <filter-name>AuditFilter</filter-name>

    <servlet-name>OrderServlet</servlet-name>

</filter-mapping>
```

Useful when a concern applies to a particular servlet rather than an entire URL space.

---

# 19. Dispatcher Types

Filters can be triggered for different dispatcher types.

Important types:

```text
REQUEST
FORWARD
INCLUDE
ERROR
ASYNC
```

This is an important interview topic.

---

# 20. REQUEST

Normal browser request:

```text
GET /dashboard
```

The filter processes it.

Example:

```java
@WebFilter(
    value = "/*",
    dispatcherTypes = DispatcherType.REQUEST
)
```

---

# 21. FORWARD

Suppose:

```java
request.getRequestDispatcher(
        "/dashboard.jsp"
).forward(request, response);
```

The request is forwarded internally.

A filter can be configured to participate in that dispatch.

---

# 22. INCLUDE

Example:

```java
request.getRequestDispatcher(
        "/header.jsp"
).include(request, response);
```

The included resource can have filter processing depending on dispatcher configuration.

---

# 23. ERROR

Suppose a servlet throws an exception.

The container may dispatch to an error resource.

Filters can participate in error dispatching when configured.

```java
dispatcherTypes =
        DispatcherType.ERROR
```

---

# 24. ASYNC

Servlet applications can support asynchronous processing.

For example:

```text
Request
  ↓
Servlet
  ↓
Async processing
  ↓
Response later
```

Filters need to be designed appropriately when asynchronous processing is involved.

---

# 25. Multiple Filters

Suppose we have:

```text
LoggingFilter
AuthenticationFilter
AuthorizationFilter
AuditFilter
MetricsFilter
```

Pipeline:

```text
Client
  ↓
Logging
  ↓
Authentication
  ↓
Authorization
  ↓
Audit
  ↓
Metrics
  ↓
Servlet
```

On the way back:

```text
Servlet
  ↓
Metrics
  ↓
Audit
  ↓
Authorization
  ↓
Authentication
  ↓
Logging
  ↓
Client
```

---

# 26. Why Filter Ordering Matters

Suppose authorization executes before authentication.

```text
Authorization
      ↓
"Who is this user?"
      ↓
Authentication hasn't happened
```

This can cause incorrect behavior.

A typical conceptual order is:

```text
Request ID
    ↓
Logging
    ↓
Security headers
    ↓
Authentication
    ↓
Authorization
    ↓
Business servlet
```

The exact ordering should be determined by your application requirements and container/framework configuration.

---

# 27. Logging Filter

A production logging filter should capture things such as:

```text
HTTP method
URI
status
duration
request ID
user identity when available
```

Example:

```java
@WebFilter("/*")
public class RequestLoggingFilter
        implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest req =
                (HttpServletRequest) request;

        HttpServletResponse res =
                (HttpServletResponse) response;

        long start =
                System.currentTimeMillis();

        try {

            chain.doFilter(
                    request,
                    response
            );

        } finally {

            long duration =
                    System.currentTimeMillis()
                    - start;

            System.out.println(
                    req.getMethod()
                    + " "
                    + req.getRequestURI()
                    + " "
                    + res.getStatus()
                    + " "
                    + duration
                    + "ms"
            );
        }
    }
}
```

Notice:

```java
finally
```

This helps ensure timing/logging happens even when downstream processing throws an exception.

---

# 28. Never Log Sensitive Data

Don't log:

```text
password
credit card number
authentication tokens
session IDs
private keys
secrets
```

Avoid:

```java
System.out.println(
    "password=" + password
);
```

Production logging must be designed around data minimization.

---

# 29. Request ID / Correlation ID

Distributed systems need to connect logs belonging to one request.

Example:

```text
X-Request-ID: 7e2a8c
```

Flow:

```text
Client
  ↓
Load Balancer
  ↓
Service A
  ↓
Service B
  ↓
Database
```

Logs:

```text
requestId=7e2a8c ServiceA...
requestId=7e2a8c ServiceB...
requestId=7e2a8c DB...
```

Now you can trace a request across components.

---

# 30. Request ID Filter

Conceptually:

```java
String requestId =
        request.getHeader("X-Request-ID");

if (requestId == null ||
    requestId.isBlank()) {

    requestId =
            UUID.randomUUID().toString();
}

response.setHeader(
        "X-Request-ID",
        requestId
);
```

In production, be careful about blindly trusting externally supplied IDs. Validate format/length and establish clear trust boundaries.

---

# 31. Performance Monitoring Filter

You can measure:

```text
request start
       ↓
servlet processing
       ↓
request end
```

Then:

```text
duration = end - start
```

Metrics could include:

```text
request_count
request_latency
error_count
status_code_count
```

This becomes the foundation of observability.

---

# 32. Security Header Filter

Applications can add security-related headers.

Examples include:

```text
Content-Security-Policy
X-Content-Type-Options
Referrer-Policy
Strict-Transport-Security
```

Example:

```java
response.setHeader(
    "X-Content-Type-Options",
    "nosniff"
);
```

Don't blindly copy security headers from random examples; their values should match your application's resource and browser requirements.

---

# 33. Character Encoding Filter

Legacy Servlet applications sometimes use a filter to ensure consistent encoding.

```java
@WebFilter("/*")
public class EncodingFilter
        implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        request.setCharacterEncoding("UTF-8");

        response.setCharacterEncoding("UTF-8");

        chain.doFilter(
                request,
                response
        );
    }
}
```

---

# 34. CORS Filter

CORS controls whether browser JavaScript from one origin can access resources from another origin.

Example:

```text
Frontend:
https://frontend.example.com

Backend:
https://api.example.com
```

These are different origins.

A CORS response may contain headers such as:

```http
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
```

But CORS configuration should be restrictive rather than simply:

```text
*
```

for authenticated/private APIs.

---

# 35. Audit Filter

Auditing asks:

```text
Who performed an action?
When?
What resource?
What operation?
What was the result?
```

Example:

```text
USER 101
DELETE
Employee 500
2026-10-07T12:20:00
SUCCESS
```

Useful for:

```text
banking
healthcare
enterprise administration
financial systems
compliance
```

---

# 36. Authentication vs Authorization vs Audit

Keep these separate:

```text
Authentication
    ↓
Who are you?

Authorization
    ↓
What can you do?

Audit
    ↓
What did you do?
```

Excellent interview concept.

---

# 37. Servlet Listeners

A listener observes lifecycle events.

Examples:

```text
Application started
Application stopped
Session created
Session destroyed
Request created
Request destroyed
Attribute added
Attribute removed
```

---

# 38. Why Listeners?

Imagine you want:

```text
When application starts:
    initialize configuration
```

or:

```text
When session starts:
    increment active users
```

or:

```text
When request ends:
    record request statistics
```

Listeners provide lifecycle hooks.

---

# 39. ServletContextListener

Used for application lifecycle.

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

---

# 40. Application Lifecycle

```text
Application deployment
        ↓
Context initialization
        ↓
Application running
        ↓
Requests
        ↓
Application shutdown
        ↓
Context destruction
```

Listener:

```text
contextInitialized()
contextDestroyed()
```

---

# 41. HttpSessionListener

```java
@WebListener
public class SessionLifecycleListener
        implements HttpSessionListener {

    @Override
    public void sessionCreated(
            HttpSessionEvent event) {

        System.out.println(
                "Session created: "
                + event.getSession().getId()
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

---

# 42. Tracking Active Sessions

Conceptually:

```java
public class SessionCounter
        implements HttpSessionListener {

    private final AtomicInteger activeSessions =
            new AtomicInteger();

    @Override
    public void sessionCreated(
            HttpSessionEvent event) {

        activeSessions.incrementAndGet();
    }

    @Override
    public void sessionDestroyed(
            HttpSessionEvent event) {

        activeSessions.decrementAndGet();
    }
}
```

Why `AtomicInteger`?

Because requests/sessions can be processed concurrently.

---

# 43. ServletRequestListener

Tracks request lifecycle.

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

# 44. Listener Types You Should Know

Important Servlet listener interfaces include:

```text
ServletContextListener
ServletContextAttributeListener

HttpSessionListener
HttpSessionAttributeListener
HttpSessionBindingListener
HttpSessionActivationListener

ServletRequestListener
ServletRequestAttributeListener
```

You don't need to memorize every method immediately.

Understand **what lifecycle each listener observes**.

---

# 45. Filter vs Listener

This is an important interview question.

| Filter                        | Listener                                    |
| ----------------------------- | ------------------------------------------- |
| Intercepts requests/responses | Observes lifecycle events                   |
| Can block requests            | Doesn't primarily act as request gatekeeper |
| Can modify request/response   | Observes events                             |
| Uses `FilterChain`            | No filter chain                             |
| Authentication                | Application/session/request lifecycle       |
| Logging                       | Session monitoring                          |
| Authorization                 | Application initialization                  |
| CORS                          | Attribute lifecycle                         |

Mental model:

```text
Filter
   ↓
"I want to intercept processing."

Listener
   ↓
"I want to observe lifecycle events."
```

---

# 46. Filter vs Servlet

```text
Servlet
    ↓
Handles application request

Filter
    ↓
Intercepts processing around the servlet
```

Example:

```text
Request
   ↓
Authentication Filter
   ↓
Logging Filter
   ↓
OrderServlet
```

---

# 47. Filter vs Interceptor

You'll encounter this later in Spring.

Traditional Servlet:

```text
Filter
```

Spring MVC:

```text
HandlerInterceptor
```

Conceptually:

```text
HTTP Request
     ↓
Servlet Filter
     ↓
Spring DispatcherServlet
     ↓
Spring Interceptor
     ↓
Controller
```

This distinction becomes important when moving from J2EE/Servlets to Spring Boot.

---

# 48. Thread Safety

This is critical.

A Filter is generally instantiated by the container and can process concurrent requests.

Don't do:

```java
public class BadFilter implements Filter {

    private String currentUser;

    public void doFilter(...) {

        currentUser = ...;

        chain.doFilter(...);
    }
}
```

Why?

Multiple requests can execute concurrently:

```text
Thread 1 → currentUser = Alice
Thread 2 → currentUser = Bob
```

Thread 1 may accidentally observe Bob.

---

# 49. Safe Filter Design

Prefer local variables:

```java
public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain) {

    String user =
            request.getParameter("user");

    // local variable

    chain.doFilter(
            request,
            response
    );
}
```

Instance fields should generally contain only immutable/shared configuration or thread-safe resources.

---

# 50. Exception Handling in Filters

Suppose:

```java
chain.doFilter(request, response);
```

throws an exception.

You can handle/log it:

```java
try {

    chain.doFilter(
            request,
            response
    );

} catch (Exception e) {

    // Log appropriately

    throw e;
}
```

Or use:

```java
finally
```

for cleanup.

---

# 51. Filter Wrappers

Servlet provides wrappers such as:

```text
HttpServletRequestWrapper
HttpServletResponseWrapper
```

These allow you to decorate requests/responses.

This is related to the **Decorator design pattern**.

Example use cases:

```text
modify request behavior
capture response body
sanitize input
add headers
measure output
```

---

# 52. Response Wrapper Concept

Suppose:

```text
Servlet
   ↓
response.getWriter()
   ↓
HTML
```

A response wrapper can intercept the response.

```text
Servlet
   ↓
Response Wrapper
   ↓
Actual Response
```

This is an advanced Servlet concept worth understanding before moving to JSP/MVC.

---

# 53. Enterprise Request Pipeline

Now combine everything:

```text
                    CLIENT
                      |
                      ↓
                   HTTPS
                      |
                      ↓
             Servlet Container
                      |
                      ↓
              Request ID Filter
                      |
                      ↓
              Logging Filter
                      |
                      ↓
          Security Headers Filter
                      |
                      ↓
          Authentication Filter
                      |
                      ↓
          Authorization Filter
                      |
                      ↓
               Audit Filter
                      |
                      ↓
                 Servlet
                      |
                      ↓
                 Service
                      |
                      ↓
                   DAO
                      |
                      ↓
                 Database
                      |
                      ↓
                 Response
```

This is a very useful architecture mental model.

---

# 54. Enterprise Filter Categories

You should be able to categorize filters.

### Infrastructure

```text
Encoding
Request ID
Compression
Logging
Metrics
```

### Security

```text
Authentication
Authorization
CSRF
Security headers
CORS
```

### Business/Compliance

```text
Audit
Tenant identification
Access policy
```

---

# 55. Multi-Tenant Applications

Suppose:

```text
Tenant A → company-a.example
Tenant B → company-b.example
```

A filter can identify the tenant.

```text
Request
   ↓
TenantFilter
   ↓
tenantId = companyA
   ↓
Servlet
   ↓
Service
```

Then services can operate within the correct tenant context.

Be careful to enforce tenant isolation at the authorization/data-access layers too; identifying a tenant in a filter alone is not sufficient.

---

# 56. Request Lifecycle + Listeners

Combine listeners and filters:

```text
Request Created
      ↓
Request Listener
      ↓
Filter 1
      ↓
Filter 2
      ↓
Servlet
      ↓
Filter 2
      ↓
Filter 1
      ↓
Request Destroyed
```

This is the lifecycle you should visualize.

---

# 57. Complete Day 5 Project

## Enterprise Employee Portal

Extend your Day 4 application.

Architecture:

```text
employee-portal/
│
├── servlet/
│   ├── LoginServlet
│   ├── LogoutServlet
│   ├── DashboardServlet
│   ├── EmployeeServlet
│   └── AdminServlet
│
├── filter/
│   ├── RequestIdFilter
│   ├── LoggingFilter
│   ├── AuthenticationFilter
│   ├── AuthorizationFilter
│   ├── AuditFilter
│   └── SecurityHeaderFilter
│
├── listener/
│   ├── ApplicationListener
│   ├── SessionListener
│   └── RequestListener
│
├── service/
│   ├── AuthenticationService
│   └── EmployeeService
│
├── dao/
│   ├── UserDAO
│   └── EmployeeDAO
│
└── model/
    ├── User
    └── Employee
```

---

# 58. Required Filter Pipeline

Implement:

```text
Request
   ↓
RequestIdFilter
   ↓
LoggingFilter
   ↓
SecurityHeaderFilter
   ↓
AuthenticationFilter
   ↓
AuthorizationFilter
   ↓
AuditFilter
   ↓
Servlet
```

---

# 59. Required Features

Your application should support:

### Public

```text
/login
/register
```

### Authenticated

```text
/dashboard
/profile
/employees
```

### Admin-only

```text
/admin
/admin/users
/admin/employees/delete
```

---

# 60. Logging Requirements

Log:

```text
requestId
HTTP method
URI
status code
duration
```

Example:

```text
requestId=abc123
method=GET
uri=/employees
status=200
duration=42ms
```

Don't log:

```text
passwords
tokens
session identifiers
sensitive personal information
```

---

# 61. Authentication Requirements

Your filter should:

```text
1. Get existing session
2. Check authenticated user
3. Allow public URLs
4. Reject protected URLs
5. Continue using chain.doFilter()
```

---

# 62. Authorization Requirements

Example:

```text
USER
 ↓
/dashboard
/profile
/employees

ADMIN
 ↓
Everything USER can access
+
/admin/*
```

Don't rely solely on hiding UI buttons.

Authorization must be enforced server-side.

---

# 63. Audit Requirements

Record important operations:

```text
LOGIN
LOGOUT
CREATE_EMPLOYEE
UPDATE_EMPLOYEE
DELETE_EMPLOYEE
```

Example:

```text
2026-10-07
user=101
action=DELETE_EMPLOYEE
employee=500
result=SUCCESS
```

---

# 64. Day 5 Advanced Exercises

### Level 1

1. Create a logging filter.
2. Create an authentication filter.
3. Create an encoding filter.
4. Create a session listener.
5. Create an application listener.

### Level 2

6. Implement multiple filters.
7. Test filter ordering.
8. Implement role-based authorization.
9. Implement request timing.
10. Implement request ID propagation.

### Level 3

11. Create a security-header filter.
12. Create an audit filter.
13. Implement a response wrapper.
14. Implement an error-dispatch filter.
15. Build a complete filter chain.

### Level 4

16. Build distributed request correlation.
17. Add metrics.
18. Add tenant identification.
19. Handle asynchronous processing.
20. Design a production-grade enterprise filter architecture.

---

# 65. Day 5 Interview Questions

## Q1. What is a Servlet Filter?

A Filter intercepts requests and responses around target web resources such as servlets, allowing cross-cutting concerns to be implemented independently of business logic.

---

## Q2. What is `FilterChain`?

`FilterChain` represents the remaining processing chain. Calling:

```java
chain.doFilter(request, response);
```

passes processing to the next filter or target resource.

---

## Q3. What happens if `chain.doFilter()` isn't called?

The request doesn't continue to the next filter/resource.

This can intentionally block unauthorized requests.

---

## Q4. What is filter lifecycle?

```text
init()
doFilter()
destroy()
```

---

## Q5. Is `doFilter()` called once?

No.

It can execute for many requests.

---

## Q6. Can multiple filters exist?

Yes.

They form a filter chain.

---

## Q7. Does filter order matter?

Yes.

Security, logging, authentication, authorization, and other filters can have dependencies on ordering.

---

## Q8. What is a cross-cutting concern?

A concern that affects multiple parts of the application.

Examples:

```text
logging
security
auditing
metrics
```

---

## Q9. Difference between Filter and Listener?

Filter intercepts processing.

Listener observes lifecycle events.

---

## Q10. What is `ServletContextListener`?

It observes application startup and shutdown events.

---

## Q11. What is `HttpSessionListener`?

It observes creation and destruction of HTTP sessions.

---

## Q12. What is `ServletRequestListener`?

It observes request initialization and destruction.

---

## Q13. What is `FilterConfig`?

It provides configuration and initialization parameters to a filter.

---

## Q14. Can a Filter modify the response?

Yes.

It can modify headers and can use response wrappers for more advanced response manipulation.

---

## Q15. Can a Filter modify a request?

Yes, including through request wrappers.

---

## Q16. Why should Filters be thread-safe?

Because a container may execute multiple requests concurrently using the same filter instance.

---

## Q17. How do you stop a request in a filter?

Don't call:

```java
chain.doFilter(...)
```

and return an appropriate response.

---

## Q18. What are dispatcher types?

Important ones are:

```text
REQUEST
FORWARD
INCLUDE
ERROR
ASYNC
```

---

## Q19. Why use `finally` in a logging filter?

To ensure cleanup/timing logic executes even if downstream processing throws an exception.

---

## Q20. Filter vs Spring Interceptor?

A Servlet Filter operates at the Servlet/container level.

A Spring MVC interceptor operates around Spring MVC handler/controller processing.

---

# 66. Day 5 Architecture Challenge

Imagine you have:

```text
10,000 requests/minute
```

Your application has:

```text
100+ servlets
```

Requirements:

```text
Authentication
Authorization
Logging
Metrics
Auditing
Request tracing
CORS
Security headers
```

Design:

```text
                   CLIENT
                      |
                      ↓
                 Load Balancer
                      |
                      ↓
              Servlet Container
                      |
              ┌───────┴────────┐
              ↓                ↓
         Request ID        Security
              ↓                ↓
           Logging        Authentication
              ↓                ↓
           Metrics        Authorization
              └───────┬────────┘
                      ↓
                   Audit
                      ↓
                  Servlet
                      ↓
                  Service
                      ↓
                    DAO
                      ↓
                  Database
```

Ask yourself:

* Which concerns belong in filters?
* Which belong in listeners?
* Which belong in services?
* Which data should be logged?
* How do you prevent sensitive data leakage?
* How do you preserve request IDs?
* How does concurrency affect filters?
* How would this work with multiple application servers?

These are **architect-level questions**, not merely Servlet questions.

---

# 67. Day 5 Mastery Checklist

Before moving to Day 6, you should be able to explain and implement:

### Filters

* [ ] Filter interface
* [ ] `init()`
* [ ] `doFilter()`
* [ ] `destroy()`
* [ ] `FilterConfig`
* [ ] `FilterChain`
* [ ] `chain.doFilter()`
* [ ] Before/after processing
* [ ] Multiple filters
* [ ] Filter ordering
* [ ] URL mapping
* [ ] Servlet mapping
* [ ] Dispatcher types

### Enterprise filters

* [ ] Logging
* [ ] Authentication
* [ ] Authorization
* [ ] Request ID
* [ ] Metrics
* [ ] CORS
* [ ] Security headers
* [ ] Encoding
* [ ] Auditing

### Listeners

* [ ] `ServletContextListener`
* [ ] `HttpSessionListener`
* [ ] `ServletRequestListener`
* [ ] Attribute listeners
* [ ] Application lifecycle
* [ ] Session lifecycle
* [ ] Request lifecycle

### Advanced

* [ ] Thread safety
* [ ] Request wrappers
* [ ] Response wrappers
* [ ] Error dispatch
* [ ] Async dispatch
* [ ] Distributed request correlation
* [ ] Production filter architecture

---

# 68. The Most Important Mental Model of Day 5

Memorize this:

```text
                         HTTP REQUEST
                              │
                              ↓
                    ┌──────────────────┐
                    │ Request Listener │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Request ID       │
                    │ Filter           │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Logging Filter   │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Security Filter  │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Authentication   │
                    │ Filter            │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Authorization    │
                    │ Filter            │
                    └────────┬─────────┘
                             ↓
                    ┌──────────────────┐
                    │ Audit Filter     │
                    └────────┬─────────┘
                             ↓
                         SERVLET
                             ↓
                          SERVICE
                             ↓
                            DAO
                             ↓
                         DATABASE
                             ↓
                         RESPONSE
                             │
                    ← Filters unwind ←
                             │
                             ↓
                       HTTP RESPONSE
```

### Your Day 5 one-line takeaway

> **Servlets implement request handling, Filters implement cross-cutting request/response processing, and Listeners observe application/request/session lifecycle events.**

Tomorrow, **Day 6 — JSP Fundamentals** moves into the presentation layer: **JSP lifecycle, JSP translation/compilation, directives, declarations, scriptlets, expressions, implicit objects, JSP scopes, EL introduction, JSP actions, and JSP vs Servlet architecture**.
