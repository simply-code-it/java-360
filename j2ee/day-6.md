# J2EE Mastery — Day 6

## JSP Fundamentals — The Traditional Java Web Presentation Layer

**Date:** October 8, 2026
**Day:** 6 of 30
**Today's objective:** Master JSP from fundamentals through the servlet container lifecycle, JSP syntax, implicit objects, scopes, directives, actions, error handling, and the correct **Servlet → JSP** MVC architecture.

> **Modern note:** JSP is still important for understanding and maintaining legacy J2EE/Jakarta EE applications, but for new applications, teams may choose other presentation technologies. For your J2EE mastery, you should understand JSP deeply because many enterprise systems still use it.

---

# 1. Where JSP Fits in J2EE

So far you've learned:

```text
Day 1 → HTTP/Web Architecture
Day 2 → Servlets
Day 3 → Advanced Servlets
Day 4 → Sessions/Auth
Day 5 → Filters/Listeners
Day 6 → JSP
```

The traditional architecture is:

```text
Browser
   |
   | HTTP Request
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
Servlet
   |
   ↓
JSP
   |
   ↓
HTML
   |
   ↓
Browser
```

The key idea:

> **Servlets should handle request/control flow; JSP should primarily render the view.**

---

# 2. What Is JSP?

JSP stands for:

**JavaServer Pages**

It is a server-side view technology used to generate dynamic HTML.

Example:

```jsp
<h1>Hello ${username}</h1>
```

The server generates HTML:

```html
<h1>Hello Pradeep</h1>
```

The browser receives HTML, not JSP.

```text
JSP
 ↓
Server-side processing
 ↓
HTML
 ↓
Browser
```

---

# 3. JSP vs Servlet

A JSP is ultimately processed by the Servlet container.

Conceptually:

```text
JSP
 ↓
Translation
 ↓
Servlet source
 ↓
Compilation
 ↓
Servlet class
 ↓
Execution
```

Therefore:

> **JSP is essentially a view-oriented abstraction built on top of the Servlet model.**

This is one of the most important concepts today.

---

# 4. JSP Lifecycle

Understand this lifecycle:

```text
        JSP file
           |
           ↓
       Translation
           |
           ↓
     Servlet source
           |
           ↓
       Compilation
           |
           ↓
      Servlet class
           |
           ↓
      Class loading
           |
           ↓
      Instantiation
           |
           ↓
       jspInit()
           |
           ↓
        _jspService()
           |
           ↓
       jspDestroy()
```

The important methods are:

```text
jspInit()
_jspService()
jspDestroy()
```

---

# 5. JSP Translation

Suppose you create:

```text
hello.jsp
```

with:

```jsp
<h1>Hello World</h1>
```

The container conceptually transforms it into a servlet.

Something conceptually similar to:

```java
public class hello_jsp
        extends HttpServlet {

    @Override
    public void _jspService(
            HttpServletRequest request,
            HttpServletResponse response) {

        response.getWriter()
                .write("<h1>Hello World</h1>");
    }
}
```

The actual generated servlet code is container-specific.

---

# 6. Why JSP Was Created

Before JSP, developers often generated HTML directly from Servlets.

Example:

```java
response.getWriter().println(
    "<html>"
);

response.getWriter().println(
    "<h1>Employee List</h1>"
);
```

This quickly becomes difficult to maintain.

JSP provides:

```text
HTML
+
Dynamic server-side data
```

So presentation becomes easier to write.

---

# 7. Bad Architecture

Avoid:

```text
JSP
 ↓
SQL
 ↓
Database
```

And avoid:

```text
JSP
 ↓
business logic
 ↓
database
```

JSP should primarily be the **view**.

---

# 8. Good Architecture

Use:

```text
Browser
   ↓
Servlet/Controller
   ↓
Service
   ↓
DAO
   ↓
Database
   ↓
Servlet/Controller
   ↓
JSP
   ↓
HTML
```

Example:

```text
GET /employees
       ↓
EmployeeServlet
       ↓
EmployeeService
       ↓
EmployeeDAO
       ↓
Database
       ↓
List<Employee>
       ↓
request.setAttribute(...)
       ↓
employee-list.jsp
```

---

# 9. Your First JSP

Create:

```text
src/main/webapp/hello.jsp
```

Content:

```jsp
<!DOCTYPE html>
<html>
<head>
    <title>Hello JSP</title>
</head>

<body>

    <h1>Hello JSP</h1>

    <p>Welcome to J2EE!</p>

</body>
</html>
```

Access:

```text
/hello.jsp
```

---

# 10. JSP Directives

JSP has directives that provide instructions to the JSP container.

Three important directives:

```text
page
include
taglib
```

---

# 11. Page Directive

Example:

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
```

You can configure things such as:

```text
contentType
pageEncoding
import
session
errorPage
isErrorPage
```

---

# 12. Import in JSP

Legacy JSP code may use:

```jsp
<%@ page import="java.util.List" %>
```

Multiple imports:

```jsp
<%@ page import="java.util.List,java.util.ArrayList" %>
```

However, in modern JSP development, you should minimize Java code inside JSP.

---

# 13. JSP Page Encoding

Prefer:

```jsp
<%@ page pageEncoding="UTF-8" %>
```

Often combined with:

```jsp
<%@ page
    contentType="text/html;charset=UTF-8"
    pageEncoding="UTF-8"
%>
```

This helps ensure correct character encoding.

---

# 14. JSP Scriptlet

Legacy JSP allows:

```jsp
<%
    String username = "Pradeep";
%>
```

This is called a **scriptlet**.

You can then write:

```jsp
<h1>
    Hello <%= username %>
</h1>
```

However:

> **Avoid scriptlets in new JSP code.**

Prefer:

```text
EL
+
JSTL
```

You'll master those tomorrow.

---

# 15. JSP Expression

Syntax:

```jsp
<%= expression %>
```

Example:

```jsp
<%= 10 + 20 %>
```

Output:

```text
30
```

Another:

```jsp
<%= new java.util.Date() %>
```

---

# 16. JSP Declaration

Syntax:

```jsp
<%!
    int counter = 0;
%>
```

This declares members of the generated servlet.

This is dangerous for request-specific state because the generated servlet may process concurrent requests.

Avoid this pattern.

---

# 17. Scriptlet vs Expression vs Declaration

| Syntax       | Purpose     |
| ------------ | ----------- |
| `<% ... %>`  | Scriptlet   |
| `<%= ... %>` | Expression  |
| `<%! ... %>` | Declaration |
| `<%@ ... %>` | Directive   |

Memorize these.

---

# 18. Why Scriptlets Are Bad

Suppose:

```jsp
<%
    for(Employee employee : employees) {
        // lots of Java
    }
%>
```

Soon the JSP becomes:

```text
HTML
Java
HTML
Java
SQL
Java
HTML
Java
```

This produces poor separation of concerns.

Better:

```text
Servlet
 ↓
Service
 ↓
DAO
 ↓
Model
 ↓
JSP
 ↓
EL/JSTL
```

---

# 19. JSP Implicit Objects

JSP provides several predefined objects.

Important ones:

```text
request
response
session
application
out
config
pageContext
page
exception
```

You should know these extremely well.

---

# 20. `request`

Represents:

```text
HttpServletRequest
```

Example:

```jsp
<%
    String username =
        request.getParameter("username");
%>
```

You can access:

```text
parameters
attributes
headers
URI
HTTP method
```

---

# 21. `response`

Represents:

```text
HttpServletResponse
```

Example:

```jsp
<%
    response.setContentType(
        "text/html"
    );
%>
```

In practice, much response configuration should happen outside the view when possible.

---

# 22. `session`

Represents:

```text
HttpSession
```

Example:

```jsp
<%
    String username =
        (String) session.getAttribute(
            "username"
        );
%>
```

Again, modern JSP should generally use EL instead:

```jsp
${sessionScope.username}
```

---

# 23. `application`

Represents:

```text
ServletContext
```

Example:

```jsp
<%
    Object value =
        application.getAttribute(
            "appName"
        );
%>
```

Equivalent conceptual scope:

```text
applicationScope
```

---

# 24. `out`

`out` is a JSP writer.

Example:

```jsp
<%
    out.println("Hello");
%>
```

But normally:

```jsp
Hello
```

or EL/JSTL should be preferred over manually writing through `out`.

---

# 25. `config`

Represents:

```text
ServletConfig
```

Conceptually:

```jsp
<%
    String value =
        config.getInitParameter(
            "name"
        );
%>
```

---

# 26. `pageContext`

`PageContext` provides access to the JSP page environment and different scopes.

Example:

```jsp
<%
    pageContext.setAttribute(
        "message",
        "Hello"
    );
%>
```

It can also help access:

```text
request
session
application
page
```

---

# 27. `page`

Represents the current JSP-generated servlet instance.

Conceptually:

```java
this
```

It is rarely used directly in modern JSP development.

---

# 28. `exception`

Available only when:

```jsp
<%@ page isErrorPage="true" %>
```

Then:

```jsp
<%= exception.getMessage() %>
```

can access the exception.

---

# 29. JSP Implicit Objects — Cheat Sheet

| Object        | Type/Concept          |
| ------------- | --------------------- |
| `request`     | `HttpServletRequest`  |
| `response`    | `HttpServletResponse` |
| `session`     | `HttpSession`         |
| `application` | `ServletContext`      |
| `out`         | JSP writer            |
| `config`      | `ServletConfig`       |
| `pageContext` | `PageContext`         |
| `page`        | Current JSP servlet   |
| `exception`   | Error exception       |

---

# 30. JSP Scopes

JSP has four major scopes:

```text
page
request
session
application
```

Think:

```text
Page
 ↓
Request
 ↓
Session
 ↓
Application
```

But they represent different lifetimes.

---

# 31. Page Scope

Exists only during the current JSP page execution.

```jsp
<%
    pageContext.setAttribute(
        "message",
        "Hello"
    );
%>
```

Scope:

```text
Current JSP execution
```

---

# 32. Request Scope

Exists for the lifetime of the current request.

Servlet:

```java
request.setAttribute(
        "employee",
        employee
);
```

JSP:

```jsp
${employee}
```

This is extremely useful for MVC.

---

# 33. Session Scope

Lives across requests for the same session.

Servlet:

```java
session.setAttribute(
        "username",
        "Pradeep"
);
```

JSP:

```jsp
${sessionScope.username}
```

---

# 34. Application Scope

Application-wide scope.

```java
ServletContext context =
        getServletContext();

context.setAttribute(
        "appName",
        "Employee Portal"
);
```

JSP:

```jsp
${applicationScope.appName}
```

Be careful with mutable application-wide state because many requests may access it concurrently.

---

# 35. Scope Comparison

| Scope       | Lifetime        | Typical Use                         |
| ----------- | --------------- | ----------------------------------- |
| Page        | Current JSP     | Temporary view data                 |
| Request     | Current request | Controller → JSP                    |
| Session     | User session    | Login/user state                    |
| Application | Application     | Shared configuration/reference data |

---

# 36. MVC With JSP

This is the architecture you should master.

```text
             Browser
                |
                ↓
           Controller
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
              Model
                |
                ↓
          request attribute
                |
                ↓
               JSP
                |
                ↓
              HTML
                |
                ↓
             Browser
```

---

# 37. Employee Example

Request:

```text
GET /employees
```

Servlet:

```java
@WebServlet("/employees")
public class EmployeeServlet
        extends HttpServlet {

    private EmployeeService service =
            new EmployeeService();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        List<Employee> employees =
                service.findAll();

        request.setAttribute(
                "employees",
                employees
        );

        RequestDispatcher dispatcher =
                request.getRequestDispatcher(
                        "/WEB-INF/views/employees.jsp"
                );

        dispatcher.forward(
                request,
                response
        );
    }
}
```

---

# 38. Why `WEB-INF`?

A common pattern is:

```text
src/main/webapp/
│
├── css/
├── js/
├── images/
│
└── WEB-INF/
    └── views/
        ├── employees.jsp
        ├── employee.jsp
        └── login.jsp
```

Resources under:

```text
WEB-INF
```

cannot normally be accessed directly through a browser URL.

Instead:

```text
Servlet
 ↓
RequestDispatcher
 ↓
WEB-INF/views/employees.jsp
```

This is useful for protecting view templates from direct access.

---

# 39. Employee JSP — Legacy Style

You may encounter:

```jsp
<%
    List<Employee> employees =
        (List<Employee>)
            request.getAttribute(
                "employees"
            );
%>

<%
for(Employee employee : employees) {
%>

    <h2>
        <%= employee.getName() %>
    </h2>

<%
}
%>
```

It works.

But this isn't the style you want to build toward.

---

# 40. Better JSP

Tomorrow we'll use:

```text
Expression Language
+
JSTL
```

For example:

```jsp
<c:forEach
        var="employee"
        items="${requestScope.employees}">

    <h2>${employee.name}</h2>

</c:forEach>
```

Much cleaner.

---

# 41. JSP Expression Language Preview

EL:

```text
Expression Language
```

Example:

```jsp
${username}
```

Instead of:

```jsp
<%= session.getAttribute("username") %>
```

Another:

```jsp
${employee.name}
```

Instead of:

```jsp
<%= employee.getName() %>
```

You'll study EL deeply on Day 7.

---

# 42. JSP Include Directive

Syntax:

```jsp
<%@ include file="header.jsp" %>
```

This is a **translation-time include**.

Conceptually:

```text
header.jsp
    +
current.jsp
    ↓
translation
    ↓
one generated servlet
```

Useful for static/common JSP fragments.

---

# 43. JSP Include Action

Different syntax:

```jsp
<jsp:include page="header.jsp" />
```

This is a request-time include.

Conceptually:

```text
JSP A
  ↓
include
  ↓
JSP B
```

This distinction is an important interview question.

---

# 44. Include Directive vs Include Action

| Include Directive                        | Include Action                        |
| ---------------------------------------- | ------------------------------------- |
| `<%@ include %>`                         | `<jsp:include>`                       |
| Translation time                         | Request time                          |
| Static inclusion concept                 | Dynamic/request-time inclusion        |
| Content becomes part of translation unit | Included resource executes separately |

Memorize:

```text
include directive → translation time

jsp:include → request time
```

---

# 45. JSP Forward Action

JSP can forward:

```jsp
<jsp:forward page="/login.jsp" />
```

But generally, application flow should be controlled by the controller layer rather than putting navigation/business decisions into JSP.

---

# 46. JSP Actions

Important standard actions:

```text
<jsp:include>
<jsp:forward>
<jsp:useBean>
<jsp:setProperty>
<jsp:getProperty>
```

You'll encounter these especially in older enterprise applications.

---

# 47. `jsp:useBean`

Legacy example:

```jsp
<jsp:useBean
    id="employee"
    class="com.example.Employee"
    scope="request"
/>
```

This creates or retrieves a bean according to the configured scope.

Modern applications generally use MVC/controller logic rather than relying heavily on JSP bean actions.

---

# 48. Error Handling

You can specify an error page:

```jsp
<%@ page
    errorPage="/WEB-INF/views/error.jsp"
%>
```

Then the error JSP:

```jsp
<%@ page isErrorPage="true" %>

<h1>Something went wrong</h1>

<p>
    An unexpected error occurred.
</p>
```

---

# 49. Better Enterprise Error Handling

A centralized approach is generally preferable:

```text
Request
   ↓
Filter
   ↓
Servlet
   ↓
Exception
   ↓
Error handling mechanism
   ↓
Error JSP
```

Don't expose:

```text
stack traces
database errors
internal class names
SQL statements
filesystem paths
credentials
```

to users.

---

# 50. JSP Security

JSP security is extremely important.

## Never trust user input

Bad:

```jsp
<%= request.getParameter("name") %>
```

if the value is inserted into HTML without appropriate escaping.

Suppose input is:

```html
<script>alert('x')</script>
```

If output is not escaped properly, it can become an XSS vulnerability.

---

# 51. XSS

XSS = Cross-Site Scripting.

Dangerous:

```jsp
<%= userInput %>
```

Prefer output mechanisms that perform appropriate HTML escaping, such as JSTL's escaping tags where applicable.

For example:

```jsp
<c:out value="${userInput}" />
```

You'll learn this in detail on Day 7.

---

# 52. Never Put SQL in JSP

Bad:

```jsp
<%
    Connection connection = ...
    PreparedStatement ps = ...
%>
```

JSP should not become your database layer.

Correct:

```text
JSP
 ↓
Servlet
 ↓
Service
 ↓
DAO
 ↓
Database
```

---

# 53. Never Put Business Logic in JSP

Bad:

```jsp
<%
    if(employee.getSalary() > 100000) {
        // complex business rules
    }
%>
```

Better:

```text
Service
 ↓
Business decision
 ↓
Model/View data
 ↓
JSP
```

The JSP should mostly render the result.

---

# 54. JSP Compilation and Caching

A container may translate and compile JSPs.

After compilation, the resulting servlet can be reused for subsequent requests.

Conceptually:

```text
First request
     ↓
Translate
     ↓
Compile
     ↓
Load
     ↓
Execute

Later requests
     ↓
Existing generated servlet
     ↓
Execute
```

Exact behavior depends on the Servlet/JSP container and configuration.

---

# 55. JSP Thread Safety

Remember:

> A JSP ultimately becomes a servlet.

Therefore, you should not assume each request gets a separate JSP instance.

Avoid shared mutable state.

Bad:

```jsp
<%!
    private int counter = 0;
%>
```

Multiple requests may access the same generated servlet instance concurrently.

---

# 56. JSP Configuration

A JSP can configure:

```jsp
<%@ page
    contentType="text/html;charset=UTF-8"
    pageEncoding="UTF-8"
    session="true"
%>
```

You should understand:

```text
contentType
pageEncoding
session
errorPage
isErrorPage
```

---

# 57. JSP Session Configuration

JSP sessions are enabled by default in typical configurations, but you can explicitly configure:

```jsp
<%@ page session="false" %>
```

Then:

```text
session
```

is not available as the normal JSP implicit session object.

This can be useful for pages that don't require session state.

---

# 58. JSP Architecture — Full Example

Suppose user requests:

```text
GET /employees
```

Flow:

```text
Browser
   |
   ↓
Load Balancer
   |
   ↓
Servlet Container
   |
   ↓
Logging Filter
   |
   ↓
Authentication Filter
   |
   ↓
EmployeeServlet
   |
   ↓
EmployeeService
   |
   ↓
EmployeeDAO
   |
   ↓
Database
   |
   ↓
List<Employee>
   |
   ↓
request.setAttribute()
   |
   ↓
RequestDispatcher
   |
   ↓
employees.jsp
   |
   ↓
HTML
   |
   ↓
Browser
```

This is the classic J2EE MVC architecture you should be able to draw in interviews.

---

# 59. Production JSP Structure

A reasonable legacy enterprise application:

```text
src/
└── main/
    ├── java/
    │   └── com/example/
    │       ├── controller/
    │       ├── service/
    │       ├── dao/
    │       ├── model/
    │       ├── filter/
    │       └── listener/
    │
    └── webapp/
        ├── css/
        ├── js/
        ├── images/
        │
        └── WEB-INF/
            ├── views/
            │   ├── login.jsp
            │   ├── dashboard.jsp
            │   └── employees.jsp
            │
            └── web.xml
```

---

# 60. Complete Employee MVC Example

## Controller

```java
@WebServlet("/employees")
public class EmployeeServlet
        extends HttpServlet {

    private final EmployeeService
            employeeService =
            new EmployeeService();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        List<Employee> employees =
                employeeService.findAll();

        request.setAttribute(
                "employees",
                employees
        );

        request.getRequestDispatcher(
                "/WEB-INF/views/employees.jsp"
        ).forward(
                request,
                response
        );
    }
}
```

---

# 61. JSP View

For now, using traditional syntax:

```jsp
<!DOCTYPE html>
<html>

<head>
    <title>Employees</title>
</head>

<body>

<h1>Employees</h1>

<%
    List<Employee> employees =
        (List<Employee>)
        request.getAttribute("employees");
%>

<%
    for (Employee employee : employees) {
%>

    <div>
        <h2>
            <%= employee.getName() %>
        </h2>

        <p>
            Department:
            <%= employee.getDepartment() %>
        </p>
    </div>

<%
    }
%>

</body>

</html>
```

This is deliberately shown because you need to recognize legacy JSP code.

Tomorrow we'll replace most of this Java code with EL/JSTL.

---

# 62. What You Should NOT Do

Avoid this:

```text
JSP
 ├── SQL
 ├── JDBC
 ├── business rules
 ├── authentication
 ├── transactions
 └── HTML
```

Instead:

```text
JSP
 ↓
Presentation only

Servlet
 ↓
Request/control flow

Service
 ↓
Business logic

DAO
 ↓
Data access
```

---

# 63. JSP vs HTML

HTML:

```html
<h1>Hello</h1>
```

is static.

JSP:

```jsp
<h1>Hello ${username}</h1>
```

can render server-side dynamic data.

Conceptually:

```text
HTML
 ↓
Static document

JSP
 ↓
Dynamic server-side view
```

---

# 64. JSP vs JavaScript

Don't confuse JSP with JavaScript.

JSP:

```text
Server-side
```

JavaScript:

```text
Usually client-side
```

Flow:

```text
Browser
   |
   | HTTP request
   ↓
JSP on server
   |
   ↓
HTML + JavaScript
   |
   ↓
Browser
   |
   ↓
JavaScript executes
```

---

# 65. JSP vs Servlet — Interview Answer

A strong answer:

> A Servlet is primarily a server-side request-processing component, while JSP is primarily a server-side view technology for generating dynamic HTML. JSP pages are translated into servlet implementations by the container, so both ultimately operate within the Servlet architecture. In an MVC application, the Servlet acts as the controller and JSP acts as the view.

---

# 66. Important Interview Questions

### Q1. What is JSP?

A server-side view technology used to generate dynamic web content, particularly HTML.

---

### Q2. Is JSP a Servlet?

Not directly, but a JSP is translated into a servlet by the JSP container.

---

### Q3. What is the JSP lifecycle?

```text
Translation
Compilation
Class loading
Instantiation
jspInit()
_jspService()
jspDestroy()
```

---

### Q4. What is `_jspService()`?

The container-generated method that handles JSP requests. It is generated by the JSP container and is not normally overridden by developers.

---

### Q5. Difference between `jspInit()` and `_jspService()`?

```text
jspInit()
→ initialization

_jspService()
→ request processing
```

---

### Q6. What is a JSP directive?

A directive provides instructions to the JSP container.

Examples:

```text
page
include
taglib
```

---

### Q7. What is a JSP scriptlet?

Java code embedded using:

```jsp
<% ... %>
```

It's a legacy technique and should generally be avoided in modern JSP development.

---

### Q8. What is a JSP expression?

```jsp
<%= expression %>
```

It evaluates an expression and writes the result to the response.

---

### Q9. What is a JSP declaration?

```jsp
<%! ... %>
```

It declares members in the generated servlet.

---

### Q10. What are JSP implicit objects?

Important examples:

```text
request
response
session
application
out
config
pageContext
page
exception
```

---

### Q11. What are JSP scopes?

```text
page
request
session
application
```

---

### Q12. Difference between request and session scope?

Request scope exists for one request.

Session scope can persist across multiple requests belonging to a session.

---

### Q13. What is `WEB-INF`?

A protected web application directory whose resources aren't normally directly accessible through browser URLs.

---

### Q14. Difference between include directive and `<jsp:include>`?

```text
<%@ include %>
→ translation-time

<jsp:include>
→ request-time
```

---

### Q15. Why shouldn't business logic be placed in JSP?

It violates separation of concerns and makes presentation code difficult to test, maintain, secure, and evolve.

---

# 67. More Advanced Interview Questions

### Q16. Why can JSP have concurrency problems?

Because the generated JSP servlet can serve multiple requests concurrently, so shared mutable fields can cause race conditions.

---

### Q17. Why use request attributes?

They provide a convenient mechanism for passing model/view data from a controller servlet to a JSP during the same request.

---

### Q18. How do you pass data from Servlet to JSP?

```java
request.setAttribute(
    "employees",
    employees
);
```

Then forward:

```java
request.getRequestDispatcher(
    "/WEB-INF/views/employees.jsp"
).forward(
    request,
    response
);
```

---

### Q19. Why use `forward()` instead of redirecting to a JSP?

A forward keeps the same request and therefore preserves request attributes.

A redirect creates a new HTTP request.

---

### Q20. Why use JSP instead of generating HTML in Servlet?

It provides cleaner separation between presentation markup and request-processing code.

---

# 68. Day 6 Hands-On Exercises

## Beginner

### Exercise 1

Create:

```text
hello.jsp
```

Display:

```text
Hello J2EE
```

---

### Exercise 2

Create a JSP that displays:

```text
Name
Age
Department
```

---

### Exercise 3

Create a Servlet:

```text
/employee
```

and pass an Employee object to JSP using:

```java
request.setAttribute()
```

---

### Exercise 4

Read:

```text
username
```

from request parameters.

---

### Exercise 5

Create a JSP demonstrating all major implicit objects.

---

# 69. Intermediate Exercises

### Exercise 6

Build:

```text
GET /employees
```

and render employee data.

---

### Exercise 7

Move JSP files into:

```text
WEB-INF/views
```

and forward to them through a Servlet.

---

### Exercise 8

Create:

```text
header.jsp
footer.jsp
```

and include them using:

```jsp
<jsp:include>
```

---

### Exercise 9

Create a custom error JSP.

---

### Exercise 10

Implement:

```text
LoginServlet
     ↓
Dashboard.jsp
```

using session state.

---

# 70. Advanced Exercises

### Exercise 11

Build:

```text
Employee List
Employee Details
Create Employee
Update Employee
Delete Employee
```

using:

```text
Servlet
Service
DAO
JSP
```

---

### Exercise 12

Create:

```text
/WEB-INF/views/
```

with:

```text
login.jsp
dashboard.jsp
employees.jsp
employee-form.jsp
error.jsp
```

---

### Exercise 13

Implement an MVC application without putting Java business logic into JSP.

---

### Exercise 14

Take a scriptlet-heavy JSP and refactor it into:

```text
Servlet
+
EL
+
JSTL
```

---

### Exercise 15

Implement an employee search page:

```text
GET /employees?department=IT
```

Flow:

```text
Browser
 ↓
Servlet
 ↓
Service
 ↓
DAO
 ↓
Database
 ↓
JSP
```

---

# 71. Day 6 Mini Project

## Employee Management MVC Application

Build:

```text
employee-management/
│
├── controller/
│   ├── LoginServlet
│   ├── LogoutServlet
│   ├── DashboardServlet
│   ├── EmployeeListServlet
│   ├── EmployeeDetailsServlet
│   └── EmployeeCreateServlet
│
├── service/
│   ├── AuthenticationService
│   └── EmployeeService
│
├── dao/
│   ├── UserDAO
│   └── EmployeeDAO
│
├── model/
│   ├── User
│   └── Employee
│
├── filter/
│   ├── AuthenticationFilter
│   └── LoggingFilter
│
└── webapp/
    ├── css/
    ├── js/
    │
    └── WEB-INF/
        └── views/
            ├── login.jsp
            ├── dashboard.jsp
            ├── employees.jsp
            ├── employee-details.jsp
            ├── employee-form.jsp
            └── error.jsp
```

Architecture:

```text
                    Browser
                       |
                       ↓
                     Filter
                       |
                       ↓
                  Controller
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
                     Model
                       |
                       ↓
                request attribute
                       |
                       ↓
                      JSP
                       |
                       ↓
                     HTML
                       |
                       ↓
                    Browser
```

---

# 72. Day 6 Production Rules

Memorize these rules:

### Rule 1

**Don't put SQL in JSP.**

### Rule 2

**Don't put business logic in JSP.**

### Rule 3

**Don't put authentication logic in JSP.**

### Rule 4

**Don't put mutable shared state in JSP declarations.**

### Rule 5

Prefer:

```text
EL + JSTL
```

over:

```text
Scriptlets
```

### Rule 6

Keep JSP under:

```text
WEB-INF/views
```

when direct access isn't desired.

### Rule 7

Escape untrusted output.

### Rule 8

Don't expose stack traces to users.

### Rule 9

Use UTF-8 consistently.

### Rule 10

Keep controller, business, persistence, and presentation responsibilities separate.

---

# 73. Day 6 — What You Need to Memorize

```text
JSP
 ↓
Translation
 ↓
Servlet
 ↓
Compilation
 ↓
Execution
```

JSP lifecycle:

```text
jspInit()
    ↓
_jspService()
    ↓
jspDestroy()
```

Directives:

```text
page
include
taglib
```

Script syntax:

```text
<%   %>   → scriptlet
<%=  %>   → expression
<%!  %>   → declaration
<%@  %>   → directive
```

Implicit objects:

```text
request
response
session
application
out
config
pageContext
page
exception
```

Scopes:

```text
page
request
session
application
```

Include:

```text
<%@ include %>
        ↓
translation-time

<jsp:include>
        ↓
request-time
```

---

# 74. Day 6 Final Mental Model

You should now visualize J2EE like this:

```text
                         CLIENT
                           |
                           | HTTP
                           ↓
                    ┌───────────────┐
                    │    FILTERS    │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │   SERVLET     │
                    │  CONTROLLER   │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │   SERVICE     │
                    │ BUSINESS LOGIC│
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │      DAO      │
                    │ DATA ACCESS   │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │   DATABASE    │
                    └───────┬───────┘
                            │
                            ↓
                       DOMAIN MODEL
                            |
                            ↓
                    request.setAttribute()
                            |
                            ↓
                    ┌───────────────┐
                    │      JSP      │
                    │     VIEW      │
                    └───────┬───────┘
                            ↓
                           HTML
                            ↓
                         CLIENT
```

### The single most important principle from Day 6

> **Don't think of JSP as "Java code inside HTML." Think of JSP as a server-side View that ultimately becomes a Servlet and should primarily render data prepared by the Controller/Service layers.**

### Day 7

The natural next step is **JSTL + Expression Language (EL)**: `${...}`, property resolution, implicit EL objects, operators, scopes, `c:if`, `c:forEach`, `c:choose`, `c:out`, URL tags, formatting, functions, escaping/XSS prevention, and refactoring the Day 6 Employee application from scriptlets into clean MVC JSP.
