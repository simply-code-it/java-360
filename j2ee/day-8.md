# J2EE Mastery — Day 8

## MVC Architecture + Enterprise Web Application Design

**Date:** October 10, 2026
**Goal:** Master how to structure a real J2EE/Jakarta EE web application using **MVC, Servlets, Services, DAOs, JSP, Filters, and Models**.

Today is important because you're moving from learning individual J2EE components to **architecting an application from those components**.

---

# 1. What You Will Master Today

By the end of Day 8, you should understand:

1. MVC architecture
2. Why MVC exists
3. Model, View, Controller
4. MVC request lifecycle
5. Servlet as Controller
6. JSP as View
7. Java objects as Model
8. Service layer
9. DAO/repository layer
10. Dependency flow
11. Front Controller pattern
12. Command pattern in web applications
13. PRG pattern
14. Request vs session vs application state
15. Authentication architecture
16. Authorization architecture
17. Exception handling architecture
18. Validation architecture
19. Transaction boundaries
20. Layered architecture
21. Package structure
22. `web.xml` vs annotations
23. Filters in MVC
24. Listeners in MVC
25. JSP fragments
26. DTOs
27. Entity vs DTO
28. Form submission
29. Redirect vs forward
30. Production-grade MVC design

---

# 2. What Is MVC?

**MVC = Model + View + Controller**

It separates an application into three major responsibilities.

```text
             MVC
              |
      ┌───────┼───────┐
      ↓       ↓       ↓
    Model   View   Controller
```

### Model

Represents application data/state.

Examples:

```java
Employee
Customer
Order
Product
Invoice
```

### View

Responsible for presentation.

Examples:

```text
JSP
HTML
CSS
JavaScript
```

### Controller

Receives requests and determines what should happen.

In traditional J2EE:

```text
Servlet = Controller
```

---

# 3. Why Do We Need MVC?

Imagine putting everything inside one Servlet:

```java
doGet() {
    // authentication

    // validation

    // SQL

    // business logic

    // calculations

    // HTML generation

    // error handling
}
```

This becomes difficult to:

```text
test
maintain
extend
debug
reuse
scale
```

MVC separates responsibilities:

```text
Servlet
   ↓
Service
   ↓
DAO
   ↓
Database
```

and:

```text
Servlet
   ↓
JSP
   ↓
HTML
```

---

# 4. MVC Architecture

A practical enterprise MVC architecture is:

```text
                  Browser
                     |
                     | HTTP Request
                     ↓
              ┌──────────────┐
              │    Filter    │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │  Controller  │
              │   Servlet    │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │   Service    │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │     DAO      │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │  Database    │
              └──────────────┘

                     ↑
                     │
                   Model
                     │
                     ↓
              ┌──────────────┐
              │     JSP      │
              │ EL + JSTL    │
              └──────┬───────┘
                     ↓
                   HTML
                     ↓
                  Browser
```

---

# 5. The Three MVC Components

## Model

The model represents application data.

Example:

```java
public class Employee {

    private Long id;
    private String name;
    private String department;
    private BigDecimal salary;

    // constructors
    // getters
    // setters
}
```

---

# 6. View

The view renders the model.

Example:

```jsp
<h1>Employee</h1>

<p>
    Name:
    <c:out value="${employee.name}" />
</p>

<p>
    Department:
    <c:out value="${employee.department}" />
</p>
```

The JSP should not know:

```text
how database works
how transactions work
how authentication works
how business rules work
```

---

# 7. Controller

Controller receives the HTTP request.

Example:

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

        request.setAttribute(
                "employees",
                employees
        );

        request.getRequestDispatcher(
                "/WEB-INF/views/employees.jsp"
        ).forward(request, response);
    }
}
```

The controller coordinates the request.

---

# 8. Controller Should NOT Become a God Class

Bad:

```java
doGet() {

    // validation
    // authentication
    // SQL
    // business rules
    // calculations
    // transaction
    // email
    // HTML
}
```

Better:

```text
Controller
    ↓
Service
    ↓
DAO
```

The Controller should primarily:

```text
receive request
      ↓
extract input
      ↓
validate basic request shape
      ↓
call service
      ↓
prepare model
      ↓
select view/redirect
```

---

# 9. Service Layer

The service layer contains business operations.

```java
public class EmployeeService {

    private final EmployeeDAO employeeDAO;

    public EmployeeService(EmployeeDAO employeeDAO) {
        this.employeeDAO = employeeDAO;
    }

    public List<Employee> findAll() {
        return employeeDAO.findAll();
    }

    public Employee findById(Long id) {
        return employeeDAO.findById(id);
    }

    public void create(Employee employee) {
        validate(employee);
        employeeDAO.save(employee);
    }

    private void validate(Employee employee) {
        if (employee.getName() == null ||
            employee.getName().isBlank()) {

            throw new IllegalArgumentException(
                    "Employee name is required"
            );
        }
    }
}
```

---

# 10. DAO Layer

DAO means:

**Data Access Object**

Its responsibility is persistence.

```java
public class EmployeeDAO {

    public List<Employee> findAll() {
        // JDBC/JPA implementation
        return List.of();
    }

    public Employee findById(Long id) {
        // database lookup
        return null;
    }

    public void save(Employee employee) {
        // INSERT
    }

    public void update(Employee employee) {
        // UPDATE
    }

    public void delete(Long id) {
        // DELETE
    }
}
```

---

# 11. Dependency Flow

A strong mental model:

```text
Controller
    ↓
Service
    ↓
DAO
    ↓
Database
```

Not:

```text
Controller → Database
```

Not:

```text
JSP → Database
```

Not:

```text
JSP → Service
```

The goal is clear responsibility boundaries.

---

# 12. Complete GET Request

User requests:

```text
GET /employees
```

Flow:

```text
Browser
   ↓
HTTP Request
   ↓
Filter
   ↓
EmployeeListServlet
   ↓
EmployeeService
   ↓
EmployeeDAO
   ↓
Database
   ↓
EmployeeDAO
   ↓
EmployeeService
   ↓
Servlet
   ↓
request.setAttribute()
   ↓
employees.jsp
   ↓
HTML
   ↓
Browser
```

---

# 13. POST Request

Suppose the user submits:

```text
POST /employees
```

Request:

```text
name=Pradeep
department=Engineering
salary=75000
```

Flow:

```text
Browser
   ↓
POST /employees
   ↓
Filter
   ↓
EmployeeCreateServlet
   ↓
Validate Input
   ↓
EmployeeService
   ↓
EmployeeDAO
   ↓
Database
```

After successful creation:

```text
POST
 ↓
Redirect
 ↓
GET /employees
```

This leads to an important pattern.

---

# 14. PRG Pattern

**PRG = Post/Redirect/Get**

Instead of:

```text
POST
 ↓
JSP
```

use:

```text
POST
 ↓
process
 ↓
redirect
 ↓
GET
 ↓
JSP
```

Example:

```java
response.sendRedirect(
    request.getContextPath() + "/employees"
);
```

---

# 15. Why PRG?

Without PRG:

```text
POST
 ↓
HTML
 ↓
Refresh
 ↓
Browser may repeat POST
```

This can accidentally create duplicate operations.

With PRG:

```text
POST
 ↓
302 Redirect
 ↓
GET
 ↓
HTML
```

Refreshing the page repeats the GET rather than the original POST.

---

# 16. Forward vs Redirect

### Forward

```java
request.getRequestDispatcher(
    "/WEB-INF/views/employees.jsp"
).forward(request, response);
```

Server-side operation.

URL normally remains:

```text
/employees
```

Request attributes remain available.

---

### Redirect

```java
response.sendRedirect("/employees");
```

Client receives a redirect and makes another request.

URL changes.

Request attributes do not carry over to the new request.

---

# 17. MVC Request Table

| Situation                | Typical action             |
| ------------------------ | -------------------------- |
| Display page             | Forward                    |
| Display validation error | Forward                    |
| Successful POST          | Redirect                   |
| Successful DELETE        | Redirect                   |
| Successful UPDATE        | Redirect                   |
| Authentication failure   | Redirect or error response |
| Render JSP               | Forward                    |

---

# 18. DTOs

A **DTO = Data Transfer Object**.

Don't always expose your persistence entity directly to the view or client.

Example:

```java
public class EmployeeResponse {

    private Long id;
    private String name;
    private String department;

    // getters/setters
}
```

Your entity might contain:

```text
id
name
department
salary
passwordHash
internalStatus
audit fields
```

The view may only need:

```text
id
name
department
```

DTOs allow you to control what crosses boundaries.

---

# 19. Entity vs DTO

### Entity

Represents persistence/domain state.

```text
EmployeeEntity
```

### DTO

Represents data crossing an application boundary.

```text
EmployeeResponse
EmployeeCreateRequest
EmployeeUpdateRequest
```

Architecture:

```text
Database
   ↓
Entity
   ↓
Service
   ↓
DTO
   ↓
Controller
   ↓
JSP
```

---

# 20. Form DTO

For employee creation:

```java
public class EmployeeCreateRequest {

    private String name;
    private String department;
    private BigDecimal salary;

    // getters/setters
}
```

Controller:

```java
String name =
        request.getParameter("name");

String department =
        request.getParameter("department");

String salary =
        request.getParameter("salary");
```

Then construct a request object.

---

# 21. Validation

Validation occurs at multiple levels.

### HTTP/input validation

```text
Is parameter present?
Is ID numeric?
Is email correctly formatted?
```

### Business validation

```text
Is salary allowed?
Does department exist?
Can this employee be promoted?
```

### Database constraints

```text
NOT NULL
UNIQUE
FOREIGN KEY
CHECK
```

Don't rely on only one layer.

---

# 22. Authentication Architecture

Authentication asks:

> Who are you?

Example:

```text
Login
 ↓
Credentials
 ↓
Authentication Service
 ↓
Session
 ↓
Authenticated User
```

A filter can enforce authentication:

```java
@WebFilter("/employees/*")
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

        HttpSession session = req.getSession(false);

        if (session == null ||
            session.getAttribute("loggedInUser") == null) {

            res.sendRedirect(
                    req.getContextPath() + "/login"
            );

            return;
        }

        chain.doFilter(request, response);
    }
}
```

---

# 23. Authorization

Authorization asks:

> What are you allowed to do?

Example:

```text
ADMIN
MANAGER
EMPLOYEE
```

Suppose:

```text
GET /employees
```

may be accessible to:

```text
ADMIN
MANAGER
```

while:

```text
DELETE /employees/10
```

may require:

```text
ADMIN
```

Authorization should be implemented in the application/security layer—not merely hidden in the JSP.

---

# 24. Filter + MVC

A production request pipeline might look like:

```text
                    HTTP Request
                         |
                         ↓
              ┌───────────────────┐
              │ Encoding Filter   │
              └─────────┬─────────┘
                        ↓
              ┌───────────────────┐
              │ Security Filter   │
              └─────────┬─────────┘
                        ↓
              ┌───────────────────┐
              │ Logging Filter    │
              └─────────┬─────────┘
                        ↓
                  Controller
                        ↓
                    Service
                        ↓
                      DAO
                        ↓
                    Database
```

---

# 25. Front Controller Pattern

As your application grows, having dozens of independent Servlets can become difficult to manage.

For example:

```text
EmployeeListServlet
EmployeeCreateServlet
EmployeeUpdateServlet
EmployeeDeleteServlet

CustomerListServlet
CustomerCreateServlet
CustomerUpdateServlet

OrderListServlet
OrderCreateServlet
...
```

A **Front Controller** provides a centralized entry point.

```text
             Browser
                |
                v
        Front Controller
                |
       ┌────────┼────────┐
       ↓        ↓        ↓
   Employee  Customer   Order
   Command   Command    Command
```

This is a common architectural pattern in enterprise web applications.

---

# 26. Front Controller Example

```java
@WebServlet("/app/*")
public class FrontControllerServlet
        extends HttpServlet {

    @Override
    protected void service(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        String path =
                request.getPathInfo();

        if ("/employees".equals(path)) {
            // employee handling
        } else if ("/customers".equals(path)) {
            // customer handling
        } else if ("/orders".equals(path)) {
            // order handling
        } else {
            response.sendError(
                    HttpServletResponse.SC_NOT_FOUND
            );
        }
    }
}
```

For a real application, don't let this become another God class.

---

# 27. Command Pattern

A cleaner Front Controller can delegate:

```text
FrontController
      |
      ├── EmployeeListCommand
      ├── EmployeeCreateCommand
      ├── EmployeeDeleteCommand
      └── EmployeeUpdateCommand
```

Example:

```java
public interface Command {

    String execute(
            HttpServletRequest request,
            HttpServletResponse response)
            throws Exception;
}
```

Then:

```java
public class EmployeeListCommand
        implements Command {

    @Override
    public String execute(
            HttpServletRequest request,
            HttpServletResponse response) {

        // call service

        return "/WEB-INF/views/employees.jsp";
    }
}
```

This connects nicely with the **Command design pattern** you are studying in LLD.

---

# 28. JSP Fragments

Don't duplicate:

```html
<header>
```

and:

```html
<footer>
```

in every JSP.

Create reusable fragments:

```text
WEB-INF/views/
    fragments/
        header.jsp
        footer.jsp
        navigation.jsp
```

Then:

```jsp
<jsp:include page="/WEB-INF/views/fragments/header.jsp" />
```

---

# 29. Typical JSP Layout

```text
WEB-INF/views/

    fragments/
        header.jsp
        navigation.jsp
        footer.jsp

    auth/
        login.jsp

    employee/
        list.jsp
        details.jsp
        create.jsp
        edit.jsp

    error/
        404.jsp
        500.jsp
```

This becomes much easier to maintain.

---

# 30. Application State

Understand these carefully.

### Request Scope

Lives for one HTTP request.

```java
request.setAttribute(
    "employees",
    employees
);
```

Ideal for:

```text
search results
form errors
page data
```

---

### Session Scope

Lives across multiple requests from a session.

```java
session.setAttribute(
    "loggedInUser",
    user
);
```

Ideal for:

```text
authentication state
shopping cart
user preferences
```

Don't store huge objects.

---

### Application Scope

Shared across the entire web application.

```java
servletContext.setAttribute(
    "applicationConfig",
    config
);
```

Be careful with concurrent access.

---

# 31. Exception Handling

Suppose DAO throws:

```java
SQLException
```

Don't expose:

```text
java.sql.SQLException:
Connection refused...
```

to the user.

Architecture:

```text
DAO Exception
     ↓
Service
     ↓
Application Exception
     ↓
Controller/Global Error Handler
     ↓
Error JSP
```

Example:

```java
public class EmployeeNotFoundException
        extends RuntimeException {

    public EmployeeNotFoundException(Long id) {
        super("Employee not found: " + id);
    }
}
```

---

# 32. Error Pages

In `web.xml`:

```xml
<error-page>
    <error-code>404</error-code>
    <location>/WEB-INF/views/error/404.jsp</location>
</error-page>

<error-page>
    <error-code>500</error-code>
    <location>/WEB-INF/views/error/500.jsp</location>
</error-page>
```

This provides centralized error presentation.

---

# 33. Transaction Boundary

A very important enterprise concept:

```text
Controller
    ↓
Service  ← Transaction boundary
    ↓
DAO
    ↓
Database
```

For example:

```text
Transfer Money
    |
    ├── Debit Account
    |
    └── Credit Account
```

These operations should belong to one logical transaction.

Don't make the JSP or Controller responsible for transaction management.

---

# 34. Layered Architecture

Your Day 8 architecture should look like:

```text
┌─────────────────────────────┐
│          Web Layer          │
│ Servlet / Filter / JSP      │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│       Application Layer     │
│          Services           │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│       Persistence Layer     │
│       DAO / Repository      │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│         Database            │
└─────────────────────────────┘
```

---

# 35. Production Package Structure

A solid starting point:

```text
src/main/java/com/example/employee/

├── controller/
│   ├── LoginServlet.java
│   ├── LogoutServlet.java
│   ├── EmployeeListServlet.java
│   ├── EmployeeCreateServlet.java
│   ├── EmployeeUpdateServlet.java
│   └── EmployeeDeleteServlet.java
│
├── service/
│   └── EmployeeService.java
│
├── dao/
│   ├── EmployeeDAO.java
│   └── JdbcEmployeeDAO.java
│
├── model/
│   ├── Employee.java
│   └── User.java
│
├── dto/
│   ├── EmployeeCreateRequest.java
│   └── EmployeeResponse.java
│
├── filter/
│   ├── AuthenticationFilter.java
│   └── LoggingFilter.java
│
├── exception/
│   ├── EmployeeNotFoundException.java
│   └── ApplicationException.java
│
└── util/
    └── DatabaseUtil.java
```

Views:

```text
src/main/webapp/

├── css/
├── js/
└── WEB-INF/
    └── views/
        ├── auth/
        │   └── login.jsp
        │
        ├── employee/
        │   ├── list.jsp
        │   ├── details.jsp
        │   ├── create.jsp
        │   └── edit.jsp
        │
        ├── fragments/
        │   ├── header.jsp
        │   ├── navigation.jsp
        │   └── footer.jsp
        │
        └── error/
            ├── 404.jsp
            └── 500.jsp
```

---

# 36. Complete Employee Create Flow

Let's connect everything.

### Step 1 — User opens form

```text
GET /employees/create
```

Controller:

```java
request.getRequestDispatcher(
    "/WEB-INF/views/employee/create.jsp"
).forward(request, response);
```

---

### Step 2 — User submits form

```text
POST /employees
```

Controller extracts:

```java
String name =
        request.getParameter("name");

String department =
        request.getParameter("department");

String salary =
        request.getParameter("salary");
```

---

### Step 3 — Controller creates request object

```java
EmployeeCreateRequest input =
        new EmployeeCreateRequest();

input.setName(name);
input.setDepartment(department);
input.setSalary(
        new BigDecimal(salary)
);
```

---

### Step 4 — Service

```java
employeeService.create(input);
```

Service performs:

```text
validation
business rules
mapping
transaction
DAO calls
```

---

### Step 5 — DAO

```java
employeeDAO.save(employee);
```

Database:

```sql
INSERT INTO employee (...)
VALUES (...);
```

---

### Step 6 — Redirect

```java
response.sendRedirect(
    request.getContextPath() +
    "/employees"
);
```

---

### Step 7 — GET

```text
GET /employees
```

The list is loaded.

---

### Step 8 — JSP

```jsp
<c:forEach var="employee"
           items="${employees}">

    <tr>
        <td>
            <c:out value="${employee.name}" />
        </td>
    </tr>

</c:forEach>
```

---

# 37. Complete Architecture

You should now be able to visualize:

```text
                    Browser
                       |
                       v
                 HTTP Request
                       |
                       v
              ┌────────────────┐
              │    Filters     │
              │ Auth/Logging   │
              └───────┬────────┘
                      |
                      v
              ┌────────────────┐
              │   Controller   │
              │    Servlet     │
              └───────┬────────┘
                      |
                      v
              ┌────────────────┐
              │    Service     │
              │ Business Logic │
              └───────┬────────┘
                      |
                      v
              ┌────────────────┐
              │      DAO       │
              │ Data Access    │
              └───────┬────────┘
                      |
                      v
                 ┌─────────┐
                 │   DB    │
                 └────┬────┘
                      |
                      v
                   Entity
                      |
                      v
                    DTO
                      |
                      v
                   Servlet
                      |
                      v
                 request.setAttribute()
                      |
                      v
                    JSP
                      |
                    EL/JSTL
                      |
                      v
                    HTML
                      |
                      v
                   Browser
```

---

# 38. Anti-Patterns to Avoid

### ❌ Servlet → SQL

```java
doGet() {
    Connection connection = ...
    Statement statement = ...
}
```

Avoid.

---

### ❌ JSP → DAO

```jsp
<%
employeeDAO.findAll();
%>
```

Avoid.

---

### ❌ JSP → Business Logic

```jsp
<%
if (salary > ...)
```

Avoid.

---

### ❌ Giant Controller

```text
Controller
 ├── validation
 ├── authentication
 ├── authorization
 ├── business rules
 ├── SQL
 ├── email
 ├── transactions
 └── HTML
```

Avoid.

---

### ❌ Entity Everywhere

Don't automatically expose your database entity to every layer.

Use DTOs where boundary separation matters.

---

# 39. MVC vs Layered Architecture

These are related but not identical.

### MVC

Primarily separates:

```text
Model
View
Controller
```

### Layered architecture

Separates:

```text
Web
Service
Persistence
Database
```

In enterprise applications they can work together:

```text
MVC
+
Layered Architecture
```

Example:

```text
Controller ← MVC
   ↓
Service    ← Layer
   ↓
DAO        ← Layer
   ↓
Database

JSP        ← MVC View
```

---

# 40. MVC vs Front Controller

MVC describes responsibilities.

Front Controller describes a centralized request-entry pattern.

You can have:

```text
MVC + Front Controller
```

together.

For example:

```text
Browser
   ↓
FrontController
   ↓
Command
   ↓
Service
   ↓
DAO
   ↓
JSP
```

---

# 41. Interview Questions

### Q1. What is MVC?

MVC is an architectural pattern separating application responsibilities into Model, View, and Controller.

---

### Q2. What is the Controller in traditional J2EE?

Usually a Servlet.

---

### Q3. What is the View?

Typically JSP in traditional J2EE applications.

---

### Q4. What is the Model?

Application/domain data represented by Java objects, DTOs, entities, etc.

---

### Q5. Why shouldn't business logic be placed in Servlets?

Because it makes controllers difficult to test, maintain, reuse, and evolve.

Business logic belongs primarily in services/domain/application components.

---

### Q6. What is PRG?

Post/Redirect/Get.

```text
POST
 ↓
Redirect
 ↓
GET
```

It helps prevent accidental form resubmission.

---

### Q7. Forward vs redirect?

Forward is server-side and preserves the current request.

Redirect instructs the client to make another request.

---

### Q8. What is DAO?

A Data Access Object encapsulates persistence/data-access operations.

---

### Q9. What is DTO?

A Data Transfer Object carries data across application boundaries without necessarily exposing the underlying persistence model.

---

### Q10. Why use a Service layer?

To centralize application/business operations and keep controllers thin.

---

### Q11. What is Front Controller?

A centralized entry point that receives requests and dispatches them to appropriate handlers.

---

### Q12. What is the role of a Filter?

Filters intercept requests/responses around servlet processing and are useful for cross-cutting concerns such as authentication, logging, encoding, and tracing.

---

### Q13. Why use `WEB-INF/views`?

It prevents JSP views from being directly accessed through normal browser URL requests and encourages controller-mediated rendering.

---

### Q14. Where should transaction boundaries usually be defined?

At the service/application-operation level, where a complete business operation can be treated as one unit of work.

---

# 42. Day 8 Hands-On Exercises

## Level 1

1. Create `Employee` model.
2. Create `EmployeeDAO`.
3. Create `EmployeeService`.
4. Create `EmployeeListServlet`.
5. Create `employees.jsp`.
6. Pass employees from Servlet to JSP.
7. Render employees with JSTL.
8. Add employee details page.
9. Add create employee page.
10. Add delete employee operation.

---

## Level 2

11. Implement PRG.
12. Implement validation.
13. Implement error JSP.
14. Add authentication filter.
15. Add authorization checks.
16. Add request logging filter.
17. Add JSP fragments.
18. Add DTOs.
19. Add custom exceptions.
20. Add centralized error handling.

---

## Level 3

21. Implement Front Controller.
22. Implement Command pattern.
23. Implement employee CRUD.
24. Add transaction handling.
25. Add JDBC DAO.
26. Add connection pooling.
27. Add pagination.
28. Add search.
29. Add sorting.
30. Add role-based authorization.

---

# 43. Day 8 Mini Project

## Employee Management System — Version 4

Build:

```text
                    Login
                      |
                      v
                 Dashboard
                      |
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
    Employees      Profile       Logout
        |
   ┌────┼────┬────┐
   ↓    ↓    ↓    ↓
 List Create Edit Delete
```

### Architecture

```text
Browser
  ↓
AuthenticationFilter
  ↓
AuthorizationFilter
  ↓
FrontController / Servlet
  ↓
Command
  ↓
EmployeeService
  ↓
EmployeeDAO
  ↓
Database
  ↓
DTO
  ↓
JSP
  ↓
EL + JSTL
```

---

# 44. Production Checklist

Before considering your MVC application complete:

### Web layer

```text
☐ Controllers are thin
☐ Filters handle cross-cutting concerns
☐ Proper HTTP methods
☐ Correct status codes
☐ PRG for successful form submissions
```

### Service layer

```text
☐ Business rules centralized
☐ Transaction boundaries defined appropriately
☐ Exceptions handled consistently
```

### DAO

```text
☐ Parameterized SQL
☐ No SQL injection
☐ Connection management
☐ Proper resource cleanup
☐ Persistence errors handled
```

### View

```text
☐ JSP under WEB-INF/views
☐ EL instead of scriptlets
☐ JSTL for common presentation logic
☐ No SQL
☐ No business logic
☐ Output escaping
```

### Security

```text
☐ Authentication
☐ Authorization
☐ Session management
☐ CSRF protection where applicable
☐ Secure cookies
☐ No sensitive information in JSP
```

---

# 45. Day 8 Mastery Test

Without looking at your notes, explain:

```text
1. What is MVC?
2. Why do we need MVC?
3. What is Model?
4. What is View?
5. What is Controller?
6. Why is Servlet commonly the Controller?
7. Why is JSP commonly the View?
8. What is Service Layer?
9. What is DAO?
10. What is DTO?
11. Entity vs DTO?
12. What is Front Controller?
13. What is Command pattern?
14. What is PRG?
15. Forward vs Redirect?
16. Request scope vs Session scope?
17. Where should business logic live?
18. Where should database logic live?
19. Where should authentication be enforced?
20. Where should authorization be enforced?
21. Where should transaction boundaries normally live?
22. Why use WEB-INF/views?
23. Why use JSP fragments?
24. Why avoid scriptlets?
25. How does a POST request travel through the application?
26. How does a GET request travel through the application?
27. How should exceptions flow through layers?
28. What responsibilities belong to Filters?
29. What makes a Controller a "God class"?
30. Design a production Employee MVC application.
```

---

# 46. Day 8 Golden Mental Model

Memorize this:

```text
                  CLIENT
                     |
                     ↓
                HTTP Request
                     |
                     ↓
                  FILTER
                     |
                     ↓
                CONTROLLER
                  Servlet
                     |
                     ↓
                  SERVICE
              Business Logic
                     |
                     ↓
                    DAO
               Data Access
                     |
                     ↓
                  DATABASE
                     |
                     ↓
                   MODEL
                     |
                     ↓
                CONTROLLER
                     |
                     ↓
               REQUEST MODEL
                     |
                     ↓
                    JSP
                 EL + JSTL
                     |
                     ↓
                   HTML
                     |
                     ↓
                  CLIENT
```

The architectural principle to remember is:

> **Controller coordinates. Service decides. DAO persists. Model carries data. View renders. Filters handle cross-cutting concerns.**

### Tomorrow — Day 9

**JDBC Deep Dive:** JDBC architecture, `DriverManager`, `DataSource`, `Connection`, `PreparedStatement`, `CallableStatement`, `ResultSet`, transactions, batch processing, connection pooling, DAO implementation, SQL injection prevention, exception handling, JDBC metadata, isolation levels, and a production-grade Employee DAO.
