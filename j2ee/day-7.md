# J2EE Mastery — Day 7

## JSTL + Expression Language (EL)

**Date:** October 9, 2026
**Goal:** Replace JSP scriptlets with clean, maintainable, MVC-friendly JSP views.

By the end of today, you should be comfortable writing JSP pages where **Java business logic is completely outside the JSP**.

---

# 1. What You'll Learn Today

### Expression Language

1. What JSP EL is
2. `${...}` syntax
3. EL variables
4. Property access
5. Bean properties
6. Collections
7. Maps
8. Arrays
9. Arithmetic operators
10. Relational operators
11. Logical operators
12. Empty operator
13. Conditional expressions
14. String operations
15. Scope resolution
16. EL implicit objects
17. Request/session/application parameters
18. Calling methods/functions
19. Null handling
20. EL security considerations

### JSTL

21. What JSTL is
22. JSTL tag libraries
23. Core tags
24. `c:out`
25. `c:set`
26. `c:remove`
27. `c:if`
28. `c:choose`
29. `c:when`
30. `c:otherwise`
31. `c:forEach`
32. `c:forTokens`
33. `c:url`
34. `c:param`
35. `c:redirect`
36. `c:catch`

### Practical

37. Refactor scriptlet JSP
38. Employee list
39. Employee details
40. Login/logout UI
41. Conditional rendering
42. Iterating collections
43. URL generation
44. MVC best practices

---

# 2. Why EL and JSTL?

Yesterday you saw legacy JSP:

```jsp
<%
    String name = (String) request.getAttribute("name");
%>

<h1>Hello <%= name %></h1>
```

This mixes Java code with presentation.

With EL:

```jsp
<h1>Hello ${name}</h1>
```

Much cleaner.

Similarly, instead of:

```jsp
<%
    for (Employee employee : employees) {
%>

    <p><%= employee.getName() %></p>

<%
    }
%>
```

Use JSTL:

```jsp
<c:forEach var="employee" items="${employees}">
    <p>${employee.name}</p>
</c:forEach>
```

### Mental model

```text
Servlet
   ↓
Service
   ↓
DAO
   ↓
Database
   ↓
Model
   ↓
request.setAttribute(...)
   ↓
JSP
   ↓
EL + JSTL
   ↓
HTML
```

---

# 3. Expression Language

JSP EL allows you to access Java objects and values without writing Java code directly inside JSP.

Basic syntax:

```jsp
${expression}
```

Example:

```java
request.setAttribute("name", "Pradeep");
```

JSP:

```jsp
<h1>Hello ${name}</h1>
```

Output:

```text
Hello Pradeep
```

---

# 4. Accessing Bean Properties

Suppose:

```java
public class Employee {

    private Long id;
    private String name;
    private String department;

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getDepartment() {
        return department;
    }
}
```

Servlet:

```java
request.setAttribute("employee", employee);
```

JSP:

```jsp
${employee.name}
```

EL internally resolves:

```java
employee.getName()
```

Similarly:

```jsp
${employee.id}
```

maps conceptually to:

```java
employee.getId()
```

and:

```jsp
${employee.department}
```

maps to:

```java
employee.getDepartment()
```

---

# 5. EL vs Java Getter

Java:

```java
employee.getName()
```

JSP EL:

```jsp
${employee.name}
```

Java:

```java
employee.getDepartment()
```

JSP:

```jsp
${employee.department}
```

This is one of the most important concepts today.

---

# 6. Nested Properties

Suppose:

```java
employee.getDepartment().getManager().getName()
```

EL can represent this as:

```jsp
${employee.department.manager.name}
```

Very useful for rendering domain objects.

---

# 7. Collections

Servlet:

```java
List<Employee> employees = employeeService.findAll();

request.setAttribute("employees", employees);
```

JSP:

```jsp
${employees}
```

To iterate, use JSTL.

```jsp
<c:forEach var="employee" items="${employees}">
    ${employee.name}
</c:forEach>
```

---

# 8. Maps

Suppose:

```java
Map<String, String> employeeMap = new HashMap<>();

employeeMap.put("name", "Pradeep");
employeeMap.put("department", "Engineering");

request.setAttribute("employee", employeeMap);
```

JSP:

```jsp
${employee.name}
```

You can also use:

```jsp
${employee["name"]}
```

---

# 9. Arrays

Suppose:

```java
String[] departments = {
    "Engineering",
    "HR",
    "Finance"
};

request.setAttribute("departments", departments);
```

JSTL:

```jsp
<c:forEach var="department" items="${departments}">
    <p>${department}</p>
</c:forEach>
```

---

# 10. EL Operators

## Arithmetic

```jsp
${10 + 5}
```

Result:

```text
15
```

Other operators:

```text
+
-
*
/
%
```

Example:

```jsp
${employee.salary * 12}
```

---

# 11. Relational Operators

```jsp
${salary > 50000}
```

Operators:

```text
==
!=
>
<
>=
<=
```

Alternative textual forms:

```text
eq
ne
gt
lt
ge
le
```

For example:

```jsp
${employee.salary gt 50000}
```

---

# 12. Logical Operators

```text
&&
||
!
```

and textual equivalents:

```text
and
or
not
```

Example:

```jsp
${employee.active && employee.salary > 50000}
```

---

# 13. The `empty` Operator

Extremely useful in JSP.

```jsp
${empty employees}
```

Checks whether the value is null or empty.

Example:

```jsp
<c:if test="${empty employees}">
    <p>No employees found.</p>
</c:if>
```

You can also use:

```jsp
${not empty employees}
```

---

# 14. Conditional Expression

EL supports the ternary operator:

```jsp
${employee.active ? 'Active' : 'Inactive'}
```

Example:

```jsp
<span>
    ${employee.active ? 'ACTIVE' : 'INACTIVE'}
</span>
```

---

# 15. EL Scopes

Remember the four JSP scopes:

```text
page
request
session
application
```

Suppose:

```java
request.setAttribute("name", "Pradeep");
```

Then:

```jsp
${name}
```

can resolve the attribute.

You can explicitly access scopes:

```jsp
${pageScope.name}
```

```jsp
${requestScope.name}
```

```jsp
${sessionScope.name}
```

```jsp
${applicationScope.name}
```

---

# 16. EL Implicit Objects

Important EL implicit objects include:

| Object             | Purpose                           |
| ------------------ | --------------------------------- |
| `pageContext`      | JSP page context                  |
| `param`            | Request parameters                |
| `paramValues`      | Multiple parameter values         |
| `header`           | HTTP headers                      |
| `headerValues`     | Multiple header values            |
| `cookie`           | Cookies                           |
| `initParam`        | Context initialization parameters |
| `pageScope`        | Page attributes                   |
| `requestScope`     | Request attributes                |
| `sessionScope`     | Session attributes                |
| `applicationScope` | Application attributes            |

---

# 17. Request Parameters

Suppose URL:

```text
/employees?id=100
```

Access it:

```jsp
${param.id}
```

Equivalent conceptually to:

```java
request.getParameter("id")
```

Multiple values:

```jsp
${paramValues.id}
```

---

# 18. HTTP Headers

Access a header:

```jsp
${header["User-Agent"]}
```

Or:

```jsp
${header.User-Agent}
```

depending on the header/property syntax.

---

# 19. Cookies

You can access cookies using:

```jsp
${cookie.JSESSIONID.value}
```

This demonstrates the EL `cookie` implicit object.

For authentication/security logic, however, keep the actual decision-making in filters/controllers rather than JSP.

---

# 20. JSTL

**JSTL = Jakarta Standard Tag Library** in modern Jakarta EE terminology; older applications commonly use the older Java EE/JSTL namespaces and dependencies.

JSTL gives JSP reusable tags for common presentation tasks.

Instead of:

```jsp
<%
for (...) {
    ...
}
%>
```

you write:

```jsp
<c:forEach>
```

---

# 21. JSTL Core Library

A common JSP declaration is:

```jsp
<%@ taglib prefix="c"
           uri="jakarta.tags.core" %>
```

The exact URI/dependency depends on the JSTL/Jakarta EE version your application uses, so always match it to your container and JSTL implementation.

---

# 22. `c:out`

Use:

```jsp
<c:out value="${employee.name}" />
```

Instead of:

```jsp
<%= employee.getName() %>
```

One important advantage is that JSTL output can provide escaping behavior appropriate for HTML rendering.

For example:

```jsp
<c:out value="${employee.name}" />
```

is preferable to blindly injecting untrusted content into HTML.

---

# 23. `c:set`

Set a variable:

```jsp
<c:set var="companyName" value="ABC Technologies" />
```

Then:

```jsp
${companyName}
```

You can also store into a scope:

```jsp
<c:set var="message"
       value="Welcome"
       scope="request" />
```

---

# 24. `c:remove`

Remove a scoped variable:

```jsp
<c:remove var="message" />
```

Or specify a scope:

```jsp
<c:remove var="message" scope="request" />
```

---

# 25. `c:if`

Conditional rendering:

```jsp
<c:if test="${employee.active}">
    <span>Active</span>
</c:if>
```

Another example:

```jsp
<c:if test="${employee.salary > 50000}">
    <span>Senior Salary Band</span>
</c:if>
```

---

# 26. `c:choose`

For multiple conditions:

```jsp
<c:choose>

    <c:when test="${employee.salary >= 100000}">
        Executive
    </c:when>

    <c:when test="${employee.salary >= 50000}">
        Senior
    </c:when>

    <c:otherwise>
        Junior
    </c:otherwise>

</c:choose>
```

This is similar to:

```java
if
else if
else
```

but belongs in the **view layer**.

---

# 27. `c:forEach`

One of the most important JSTL tags.

```jsp
<c:forEach var="employee"
           items="${employees}">

    <p>${employee.name}</p>

</c:forEach>
```

---

# 28. `varStatus`

You can obtain iteration information:

```jsp
<c:forEach var="employee"
           items="${employees}"
           varStatus="status">

    ${status.index}
    ${employee.name}

</c:forEach>
```

Useful properties include:

```text
index
count
first
last
```

Example:

```jsp
<c:forEach var="employee"
           items="${employees}"
           varStatus="status">

    <tr>
        <td>${status.count}</td>
        <td>${employee.name}</td>
    </tr>

</c:forEach>
```

---

# 29. Employee Table

Servlet:

```java
@WebServlet("/employees")
public class EmployeeListServlet extends HttpServlet {

    private final EmployeeService employeeService =
            new EmployeeService();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        List<Employee> employees =
                employeeService.findAll();

        request.setAttribute("employees", employees);

        request.getRequestDispatcher(
                "/WEB-INF/views/employees.jsp"
        ).forward(request, response);
    }
}
```

JSP:

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ taglib prefix="c" uri="jakarta.tags.core" %>

<html>
<head>
    <title>Employees</title>
</head>

<body>

<h1>Employees</h1>

<c:choose>

    <c:when test="${empty employees}">
        <p>No employees found.</p>
    </c:when>

    <c:otherwise>

        <table border="1">

            <thead>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>Department</th>
            </tr>
            </thead>

            <tbody>

            <c:forEach var="employee"
                       items="${employees}">

                <tr>
                    <td>
                        <c:out value="${employee.id}" />
                    </td>

                    <td>
                        <c:out value="${employee.name}" />
                    </td>

                    <td>
                        <c:out value="${employee.department}" />
                    </td>
                </tr>

            </c:forEach>

            </tbody>

        </table>

    </c:otherwise>

</c:choose>

</body>
</html>
```

Notice what is **not** inside the JSP:

```text
❌ SQL
❌ DAO
❌ business logic
❌ authentication logic
❌ Java loops
❌ database connection
```

---

# 30. `c:url`

Generate URLs safely:

```jsp
<c:url var="employeeUrl"
       value="/employees" />
```

Then:

```jsp
<a href="${employeeUrl}">
    Employees
</a>
```

This is particularly useful when URL rewriting/session tracking is required.

---

# 31. `c:param`

Add parameters:

```jsp
<c:url var="employeeUrl" value="/employee">
    <c:param name="id" value="${employee.id}" />
</c:url>
```

Then:

```jsp
<a href="${employeeUrl}">
    View Employee
</a>
```

Conceptually generates:

```text
/employee?id=101
```

with appropriate URL encoding.

---

# 32. Employee Details Page

Servlet:

```java
@WebServlet("/employee")
public class EmployeeDetailsServlet extends HttpServlet {

    private final EmployeeService employeeService =
            new EmployeeService();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        Long id = Long.valueOf(
                request.getParameter("id")
        );

        Employee employee =
                employeeService.findById(id);

        request.setAttribute("employee", employee);

        request.getRequestDispatcher(
                "/WEB-INF/views/employee-details.jsp"
        ).forward(request, response);
    }
}
```

JSP:

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ taglib prefix="c" uri="jakarta.tags.core" %>

<h1>Employee Details</h1>

<p>
    ID:
    <c:out value="${employee.id}" />
</p>

<p>
    Name:
    <c:out value="${employee.name}" />
</p>

<p>
    Department:
    <c:out value="${employee.department}" />
</p>

<p>
    Status:
    <c:choose>

        <c:when test="${employee.active}">
            Active
        </c:when>

        <c:otherwise>
            Inactive
        </c:otherwise>

    </c:choose>
</p>
```

---

# 33. Login Example

Suppose the authentication filter places the authenticated user into the session:

```java
session.setAttribute("loggedInUser", user);
```

JSP:

```jsp
<h1>
    Welcome ${sessionScope.loggedInUser.name}
</h1>
```

You can conditionally render:

```jsp
<c:if test="${not empty sessionScope.loggedInUser}">

    Welcome
    <c:out value="${sessionScope.loggedInUser.name}" />

</c:if>
```

Again, the JSP is **displaying authentication state**, not deciding whether the request is authorized.

---

# 34. Login/Logout Navigation

```jsp
<c:choose>

    <c:when test="${not empty sessionScope.loggedInUser}">

        <span>
            Welcome
            <c:out value="${sessionScope.loggedInUser.name}" />
        </span>

        <a href="${pageContext.request.contextPath}/logout">
            Logout
        </a>

    </c:when>

    <c:otherwise>

        <a href="${pageContext.request.contextPath}/login">
            Login
        </a>

    </c:otherwise>

</c:choose>
```

---

# 35. `pageContext`

One useful EL implicit object is:

```jsp
${pageContext.request.contextPath}
```

Suppose your application is deployed as:

```text
employee-app
```

It can resolve to:

```text
/employee-app
```

Then:

```jsp
<a href="${pageContext.request.contextPath}/employees">
```

helps avoid hardcoding deployment-specific context paths.

---

# 36. Important: EL Null Handling

Suppose:

```java
request.setAttribute("employee", null);
```

Then:

```jsp
${employee.name}
```

does not behave like normal Java dereferencing and throwing a conventional `NullPointerException`.

EL is designed for view-friendly property resolution and null handling.

Still, don't use this as an excuse to ignore application correctness.

Better:

```jsp
<c:if test="${not empty employee}">
    ${employee.name}
</c:if>
```

---

# 37. Scriptlet vs EL/JSTL

### Old JSP

```jsp
<%
for (Employee e : employees) {
    if (e.isActive()) {
%>

    <%= e.getName() %>

<%
    }
}
%>
```

### Modern style

```jsp
<c:forEach var="employee"
           items="${employees}">

    <c:if test="${employee.active}">

        <c:out value="${employee.name}" />

    </c:if>

</c:forEach>
```

The second approach clearly separates:

```text
Java application logic
        ↓
JSP presentation
```

---

# 38. JSTL Formatting Libraries

JSTL also has formatting functionality.

Depending on your Jakarta EE/JSTL version, you may use the appropriate formatting tag library.

Typical capabilities include:

```text
dates
numbers
currencies
locales
internationalization
message bundles
```

Example conceptually:

```jsp
<fmt:formatNumber
        value="${employee.salary}"
        type="currency" />
```

This is useful for presentation formatting while keeping formatting logic out of Java controllers.

---

# 39. Internationalization

Enterprise applications may support:

```text
English
Hindi
French
German
Japanese
etc.
```

Instead of hardcoding:

```jsp
<h1>Welcome</h1>
```

you can use message bundles and formatting tags.

Conceptually:

```jsp
<fmt:message key="welcome.message" />
```

The actual text can come from resource bundles.

---

# 40. JSTL SQL Tags — Should You Use Them?

JSTL historically provides SQL tags.

You may encounter:

```jsp
<sql:query>
```

in tutorials.

**Do not use JSP/JSTL SQL tags for production enterprise architecture.**

Avoid:

```text
JSP
 ↓
SQL
 ↓
Database
```

Prefer:

```text
JSP
 ↓
Servlet
 ↓
Service
 ↓
DAO/Repository
 ↓
Database
```

---

# 41. Production MVC Architecture

Your application should look like:

```text
Browser
   |
   v
Filter
   |
   v
Controller Servlet
   |
   v
Service
   |
   v
DAO
   |
   v
Database
```

Response:

```text
Database
   |
   v
DAO
   |
   v
Service
   |
   v
Servlet
   |
   | request.setAttribute()
   v
JSP
   |
   | EL + JSTL
   v
HTML
   |
   v
Browser
```

---

# 42. Recommended Project Structure

```text
employee-app/
│
├── pom.xml
│
└── src/
    └── main/
        ├── java/
        │   └── com/example/employee/
        │       ├── controller/
        │       │   ├── LoginServlet.java
        │       │   ├── LogoutServlet.java
        │       │   ├── EmployeeListServlet.java
        │       │   └── EmployeeDetailsServlet.java
        │       │
        │       ├── service/
        │       │   └── EmployeeService.java
        │       │
        │       ├── dao/
        │       │   └── EmployeeDAO.java
        │       │
        │       ├── model/
        │       │   └── Employee.java
        │       │
        │       └── filter/
        │           └── AuthenticationFilter.java
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
                    └── employee-details.jsp
```

---

# 43. Why `WEB-INF/views`?

Putting JSP files under:

```text
WEB-INF/
```

prevents direct browser access to those resources through ordinary URL navigation.

For example:

```text
/WEB-INF/views/employees.jsp
```

should normally be reached through a controller:

```java
request.getRequestDispatcher(
    "/WEB-INF/views/employees.jsp"
).forward(request, response);
```

This reinforces:

```text
Browser
   ↓
Controller
   ↓
JSP
```

rather than:

```text
Browser
   ↓
JSP directly
```

---

# 44. Day 7 Production Rules

Memorize these:

### Rule 1

Don't put Java business logic in JSP.

### Rule 2

Avoid scriptlets:

```jsp
<% ... %>
```

### Rule 3

Prefer:

```jsp
${...}
```

and JSTL.

### Rule 4

Don't put SQL in JSP.

### Rule 5

Don't put authentication decisions in JSP.

### Rule 6

Don't expose stack traces to users.

### Rule 7

Escape untrusted HTML output.

### Rule 8

Use proper scopes.

### Rule 9

Keep JSPs focused on presentation.

### Rule 10

Use controllers/services/DAOs for application logic.

---

# 45. Interview Questions

## Q1. What is JSP EL?

Expression Language allows JSP pages to access values and object properties using expressions such as:

```jsp
${employee.name}
```

without embedding Java code in the JSP.

---

## Q2. What does `${employee.name}` mean?

It resolves the `name` property of the `employee` object, typically through JavaBean-style property access such as:

```java
employee.getName()
```

---

## Q3. What is JSTL?

JSTL provides standard JSP tags for common presentation operations such as:

```text
conditions
loops
URL generation
output
formatting
internationalization
```

---

## Q4. Why use JSTL?

Instead of embedding Java code:

```jsp
<%
if (...) {
}
%>
```

you can use:

```jsp
<c:if>
```

This improves separation of concerns and JSP maintainability.

---

## Q5. What is `c:forEach`?

It is used to iterate over collections, arrays, maps, and other supported iterable data.

```jsp
<c:forEach var="employee"
           items="${employees}">
    ${employee.name}
</c:forEach>
```

---

## Q6. Difference between `c:if` and `c:choose`?

`c:if` is useful for a single condition.

```jsp
<c:if test="${employee.active}">
```

`c:choose` supports multiple alternatives:

```jsp
<c:choose>
    <c:when test="...">
    </c:when>
    <c:otherwise>
    </c:otherwise>
</c:choose>
```

---

## Q7. What is `empty`?

It checks whether an EL value is null or empty.

```jsp
${empty employees}
```

---

## Q8. What is `param`?

It provides access to request parameters.

```jsp
${param.id}
```

is conceptually equivalent to:

```java
request.getParameter("id")
```

---

## Q9. What is `sessionScope`?

It allows explicit access to attributes stored in the HTTP session:

```jsp
${sessionScope.loggedInUser}
```

---

## Q10. Why shouldn't JSP contain SQL?

Because it violates separation of concerns.

Correct:

```text
JSP → Controller → Service → DAO → DB
```

Not:

```text
JSP → Database
```

---

# 46. Day 7 Coding Exercises

Complete these without looking at the solution first.

### Easy

1. Display a request attribute using EL.
2. Display an Employee object's properties.
3. Display a session attribute.
4. Check whether a collection is empty.
5. Implement `c:if`.
6. Implement `c:choose`.
7. Iterate over a list using `c:forEach`.
8. Display iteration index/count.
9. Generate an application-relative URL.
10. Generate a URL with `c:param`.

### Medium

11. Build an Employee List JSP.
12. Build Employee Details JSP.
13. Build Login JSP.
14. Display different navigation for logged-in/logged-out users.
15. Display different salary categories.
16. Display "No employees found" for an empty collection.
17. Add employee details links.
18. Add pagination UI.
19. Add department filtering UI.
20. Add status filtering UI.

### Advanced

21. Refactor a scriptlet-heavy JSP.
22. Build an MVC employee application.
23. Implement authentication-aware navigation.
24. Implement error JSP using EL/JSTL.
25. Implement internationalized messages.
26. Implement currency/date presentation.
27. Add reusable JSP fragments.
28. Protect JSPs under `WEB-INF/views`.
29. Add output escaping for user-generated data.
30. Build the complete Employee Management UI using EL + JSTL.

---

# 47. Day 7 Mini Project

## Employee Management System — Version 3

Build:

```text
Login
  ↓
Dashboard
  ↓
Employee List
  ↓
Employee Details
  ↓
Create Employee
  ↓
Edit Employee
  ↓
Delete Employee
```

Architecture:

```text
                    Browser
                       |
                       v
                 Authentication
                     Filter
                       |
                       v
                Controller Servlet
                       |
                       v
                     Service
                       |
                       v
                      DAO
                       |
                       v
                   Database
                       |
                       v
                     Model
                       |
                       v
                  JSP View
                       |
                  EL + JSTL
                       |
                       v
                     HTML
```

### JSP requirements

Your JSPs should contain:

```text
EL
JSTL
HTML
CSS
small presentation conditions
```

They should **not** contain:

```text
Java scriptlets
SQL
DAO calls
business rules
database connections
authentication decisions
```

---

# 48. Your Day 7 Mastery Test

You should be able to answer these from memory:

```text
1. What is JSP EL?
2. Why does ${employee.name} work?
3. What is JSTL?
4. What is c:out?
5. What is c:set?
6. What is c:if?
7. What is c:choose?
8. What is c:forEach?
9. What is varStatus?
10. What is c:url?
11. What is c:param?
12. What is empty?
13. What is requestScope?
14. What is sessionScope?
15. What is applicationScope?
16. What is param?
17. What is cookie?
18. How do you access a JavaBean property from EL?
19. Why avoid scriptlets?
20. Why avoid SQL in JSP?
21. How does Servlet → JSP data transfer work?
22. Why use WEB-INF/views?
23. How do you render an empty collection?
24. How do you conditionally display UI?
25. How do you iterate over employees?
```

---

# 49. The Most Important Mental Model

Don't think:

```text
JSP = Java + HTML
```

Think:

```text
JSP = Presentation/View
```

Your application should follow:

```text
              ┌──────────────┐
              │    Browser   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │   Servlet    │
              │  Controller  │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │   Service    │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │     DAO      │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │   Database   │
              └──────────────┘

                     ▲
                     │
                  Model
                     │
                     │
              ┌──────┴───────┐
              │     JSP      │
              │  EL + JSTL   │
              └──────────────┘
                     │
                     ▼
                   HTML
```

### Day 7 golden rule

> **Servlets prepare the data. Services implement business operations. DAOs access data. JSP + EL + JSTL render the data.**

Once this separation becomes natural, you are ready for **Day 8: MVC Architecture in J2EE**, where we'll take the Servlet + Service + DAO + JSP pieces and design a proper enterprise MVC application.
