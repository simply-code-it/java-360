# Day 14 — Midpoint Project: Production-Style Employee Management System

**Date:** October 16, 2026
**Phase:** J2EE/Jakarta EE Mastery — Midpoint Project
**Goal:** Combine everything you've learned from **Days 1–13** into one complete enterprise application.

Today is not primarily about learning another API. It is about learning how to **assemble enterprise Java components into a maintainable system**.

---

# 1. What You're Building

Build a complete:

> **Employee Management System (EMS)**

It will support:

```text
Authentication
    ↓
Employee Management
    ↓
Departments
    ↓
Search
    ↓
Pagination
    ↓
Employee Creation
    ↓
Employee Update
    ↓
Employee Deletion
    ↓
Transactions
    ↓
Validation
    ↓
Exception Handling
    ↓
Audit Logging
```

You'll implement it using traditional enterprise Java/J2EE concepts.

---

# 2. Technologies

Use:

```text
Java 11+
Jakarta EE / Servlet API
Servlets
JSP
EL
JSTL
JDBC
JPA
JTA concepts
HTML/CSS
Maven
Git
Tomcat / WildFly / Payara
MySQL or PostgreSQL
```

For the first version, you can use:

```text
Servlets
    +
JSP
    +
JDBC
```

Then progressively integrate JPA.

---

# 3. Architecture

Your application should follow:

```text
                    Browser
                       │
                       ▼
                 HTTP Request
                       │
                       ▼
                 Filter Chain
                       │
                       ▼
               Controller Servlet
                       │
                       ▼
                  Service Layer
                       │
                 Transaction
                       │
                       ▼
                   DAO Layer
                       │
              ┌────────┴────────┐
              ▼                 ▼
            JDBC               JPA
              │                 │
              └────────┬────────┘
                       ▼
                    Database
                       │
                       ▼
                 Service Layer
                       │
                       ▼
                 Controller
                       │
                       ▼
                     JSP
                       │
                 EL + JSTL
                       │
                       ▼
                    Browser
```

---

# 4. Layer Responsibilities

This is extremely important.

## Controller

Responsible for:

```text
HTTP
Request parameters
Request validation
Calling service
Choosing view
Redirect/forward
```

It should NOT contain:

```text
SQL
business rules
transaction logic
database logic
```

---

# 5. Service Layer

Responsible for:

```text
Business rules
Use cases
Transaction boundary
Coordination between DAOs
```

Example:

```java
public void createEmployee(EmployeeCreateRequest request) {
    validate(request);
    employeeDao.save(...);
}
```

---

# 6. DAO Layer

Responsible for:

```text
Database interaction
SQL
JPA queries
Result mapping
Persistence
```

Example:

```java
public Employee findById(Long id) {
    // database operation
}
```

---

# 7. View Layer

JSP should primarily handle:

```text
HTML
EL
JSTL
presentation
```

Avoid:

```jsp
<%
    // business logic
%>
```

Use:

```jsp
${employee.name}
```

and:

```jsp
<c:forEach ...>
```

---

# 8. Filter Layer

Filters handle cross-cutting concerns:

```text
Authentication
Logging
Encoding
Security headers
Request tracing
Authorization
```

Example:

```text
Request
   ↓
EncodingFilter
   ↓
LoggingFilter
   ↓
AuthenticationFilter
   ↓
AuthorizationFilter
   ↓
Servlet
```

---

# 9. Database Design

Create:

```sql
CREATE DATABASE employee_management;
```

---

# 10. Department Table

```sql
CREATE TABLE department (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE,
    description VARCHAR(255)
);
```

---

# 11. Employee Table

```sql
CREATE TABLE employee (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    first_name VARCHAR(100) NOT NULL,

    last_name VARCHAR(100) NOT NULL,

    email VARCHAR(150) NOT NULL UNIQUE,

    phone VARCHAR(30),

    salary DECIMAL(15,2),

    department_id BIGINT,

    status VARCHAR(30) NOT NULL,

    created_at TIMESTAMP NOT NULL,

    updated_at TIMESTAMP NOT NULL,

    version BIGINT NOT NULL DEFAULT 0,

    CONSTRAINT fk_employee_department
        FOREIGN KEY (department_id)
        REFERENCES department(id)
);
```

---

# 12. User Table

For authentication:

```sql
CREATE TABLE app_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    username VARCHAR(100) NOT NULL UNIQUE,

    password_hash VARCHAR(255) NOT NULL,

    role VARCHAR(50) NOT NULL,

    active BOOLEAN NOT NULL DEFAULT TRUE,

    created_at TIMESTAMP NOT NULL
);
```

Never store plain-text passwords.

---

# 13. Audit Table

Create:

```sql
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,

    username VARCHAR(100),

    action VARCHAR(100) NOT NULL,

    entity_type VARCHAR(100),

    entity_id BIGINT,

    created_at TIMESTAMP NOT NULL,

    details VARCHAR(1000)
);
```

Example:

```text
username: admin
action: CREATE_EMPLOYEE
entity_type: Employee
entity_id: 100
```

---

# 14. Entity Model

Create:

```java
@Entity
@Table(name = "employee")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;

    private String lastName;

    private String email;

    private String phone;

    private BigDecimal salary;

    @Enumerated(EnumType.STRING)
    private EmployeeStatus status;

    @Version
    private Long version;

    // getters/setters
}
```

---

# 15. Employee Status

```java
public enum EmployeeStatus {

    ACTIVE,
    INACTIVE,
    TERMINATED,
    ON_LEAVE
}
```

Using an enum prevents random strings from spreading throughout the application.

---

# 16. Department Relationship

Employee:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "department_id")
private Department department;
```

Department:

```java
@OneToMany(mappedBy = "department")
private List<Employee> employees;
```

Prefer deliberate fetch strategies rather than blindly making relationships `EAGER`.

---

# 17. DTOs

Don't expose your JPA entities everywhere.

Create:

```text
EmployeeCreateRequest
EmployeeUpdateRequest
EmployeeResponse
EmployeeSearchRequest
```

For example:

```java
public class EmployeeCreateRequest {

    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private BigDecimal salary;
    private Long departmentId;
}
```

---

# 18. Why DTOs?

Consider your entity:

```text
Employee
 ├── Department
 │     └── Employees
 │           └── Department
 │                 └── ...
```

Directly exposing entities can cause:

```text
circular references
lazy-loading problems
over-fetching
accidental updates
API coupling
```

DTOs provide a controlled boundary.

---

# 19. Repository / DAO Interface

Create:

```java
public interface EmployeeDAO {

    Employee findById(Long id);

    List<Employee> findAll();

    List<Employee> search(
            String name,
            String email,
            Long departmentId
    );

    void save(Employee employee);

    void update(Employee employee);

    void delete(Employee employee);
}
```

---

# 20. Service Interface

```java
public interface EmployeeService {

    EmployeeResponse create(
            EmployeeCreateRequest request
    );

    EmployeeResponse findById(Long id);

    List<EmployeeResponse> search(
            EmployeeSearchRequest request
    );

    EmployeeResponse update(
            Long id,
            EmployeeUpdateRequest request
    );

    void delete(Long id);
}
```

---

# 21. Employee Create Flow

When user submits:

```text
POST /employees/create
```

Flow:

```text
Browser
   ↓
CreateEmployeeServlet
   ↓
Validate request
   ↓
EmployeeService
   ↓
Transaction BEGIN
   ↓
Find Department
   ↓
Create Employee
   ↓
EmployeeDAO
   ↓
Database
   ↓
Create Audit Log
   ↓
COMMIT
   ↓
Redirect
   ↓
GET /employees
```

---

# 22. Transaction Boundary

The service operation should conceptually own the business transaction:

```text
createEmployee()
      │
      ├── validate
      │
      ├── create employee
      │
      └── audit
```

One logical operation:

```text
BEGIN
 ↓
Create employee
 ↓
Audit
 ↓
COMMIT
```

Failure:

```text
ROLLBACK
```

---

# 23. PRG Pattern

After successful POST:

```text
POST /employees/create
```

don't render the employee list directly.

Instead:

```text
POST
 ↓
302 Redirect
 ↓
GET /employees
```

This is:

> **Post/Redirect/Get**

It prevents accidental duplicate form submission when the user refreshes the page.

---

# 24. Employee List

URL:

```text
GET /employees
```

Flow:

```text
Browser
   ↓
EmployeeListServlet
   ↓
EmployeeService
   ↓
EmployeeDAO
   ↓
Database
   ↓
EmployeeResponse[]
   ↓
request.setAttribute()
   ↓
JSP
```

JSP:

```jsp
<c:forEach var="employee"
           items="${employees}">

    <tr>
        <td>${employee.id}</td>
        <td>${employee.firstName}</td>
        <td>${employee.lastName}</td>
        <td>${employee.email}</td>
    </tr>

</c:forEach>
```

---

# 25. Search

Support:

```text
GET /employees?name=John
```

or:

```text
GET /employees?departmentId=5
```

or:

```text
GET /employees?name=John&departmentId=5
```

Don't build SQL like:

```java
"SELECT * FROM employee WHERE name = '" + name + "'"
```

Use parameter binding.

---

# 26. Pagination

Instead of:

```sql
SELECT *
FROM employee;
```

use:

```sql
SELECT *
FROM employee
ORDER BY id
LIMIT ? OFFSET ?;
```

Conceptually:

```text
page = 0
size = 20

offset = page * size
```

Then:

```text
Page 0 → employees 1–20
Page 1 → employees 21–40
Page 2 → employees 41–60
```

---

# 27. Authentication

Login:

```text
POST /login
```

Flow:

```text
Browser
   ↓
LoginServlet
   ↓
UserDAO
   ↓
Database
   ↓
Verify password
   ↓
Create session
   ↓
Redirect dashboard
```

Session:

```java
HttpSession session =
        request.getSession();

session.setAttribute(
        "authenticatedUser",
        user
);
```

---

# 28. Authentication Filter

Protected resources:

```text
/employees/*
/departments/*
/dashboard/*
```

Filter:

```java
HttpSession session =
        request.getSession(false);

if (session == null ||
    session.getAttribute("authenticatedUser") == null) {

    response.sendRedirect(
        request.getContextPath() + "/login"
    );

    return;
}

chain.doFilter(request, response);
```

---

# 29. Authorization

Authentication:

> Who are you?

Authorization:

> What are you allowed to do?

Example:

```text
ADMIN
 ├── Create employee
 ├── Update employee
 ├── Delete employee
 └── View employees

HR
 ├── Create employee
 ├── Update employee
 └── View employees

EMPLOYEE
 └── View own profile
```

---

# 30. Authorization Filter

Conceptually:

```text
Request
   ↓
AuthenticationFilter
   ↓
AuthorizationFilter
   ↓
Servlet
```

Example:

```java
if (!"ADMIN".equals(user.getRole())) {
    response.sendError(
        HttpServletResponse.SC_FORBIDDEN
    );
    return;
}
```

---

# 31. Validation

Validate at multiple boundaries.

Example:

```java
if (request.getEmail() == null ||
    request.getEmail().isBlank()) {

    throw new ValidationException(
        "Email is required"
    );
}
```

Also validate:

```text
salary > 0
email format
name length
department exists
status allowed
```

---

# 32. Exception Handling

Create application exceptions:

```text
EmployeeNotFoundException
DepartmentNotFoundException
DuplicateEmailException
ValidationException
UnauthorizedException
ForbiddenException
```

Avoid:

```java
catch (Exception e) {
    e.printStackTrace();
}
```

in production application code.

---

# 33. Error Flow

```text
Servlet
   ↓
Service
   ↓
Exception
   ↓
Global/error handling mechanism
   ↓
Error JSP
```

Example:

```text
/WEB-INF/views/error/404.jsp
/WEB-INF/views/error/403.jsp
/WEB-INF/views/error/500.jsp
```

---

# 34. Logging

Every request should have useful logs.

Example:

```text
2026-10-16 10:30:15
requestId=abc123
user=admin
method=POST
path=/employees/create
```

Use structured logging where possible.

Don't log:

```text
passwords
session secrets
access tokens
sensitive personal information
```

---

# 35. Request Correlation ID

A useful enterprise pattern:

```text
Client
 ↓
X-Request-ID: abc123
 ↓
Filter
 ↓
Servlet
 ↓
Service
 ↓
DAO
```

Every log entry can include:

```text
requestId=abc123
```

This makes debugging distributed requests much easier.

---

# 36. Audit Logging

When admin creates an employee:

```text
CREATE_EMPLOYEE
```

When employee is updated:

```text
UPDATE_EMPLOYEE
```

When deleted:

```text
DELETE_EMPLOYEE
```

Example:

```text
admin
UPDATE_EMPLOYEE
Employee
102
2026-10-16 10:45
```

Audit logs should be designed separately from ordinary debug logs.

---

# 37. JSP Structure

```text
src/main/webapp/
│
├── css/
│   └── application.css
│
├── js/
│   └── application.js
│
└── WEB-INF/
    └── views/
        ├── login.jsp
        ├── dashboard.jsp
        │
        ├── employees/
        │   ├── list.jsp
        │   ├── create.jsp
        │   ├── edit.jsp
        │   └── details.jsp
        │
        ├── departments/
        │   ├── list.jsp
        │   └── create.jsp
        │
        └── error/
            ├── 403.jsp
            ├── 404.jsp
            └── 500.jsp
```

---

# 38. Complete Java Package Structure

```text
src/main/java/
└── com.example.ems
    │
    ├── controller
    │   ├── LoginServlet.java
    │   ├── LogoutServlet.java
    │   ├── DashboardServlet.java
    │   ├── EmployeeListServlet.java
    │   ├── EmployeeCreateServlet.java
    │   ├── EmployeeEditServlet.java
    │   ├── EmployeeDeleteServlet.java
    │   └── EmployeeDetailsServlet.java
    │
    ├── service
    │   ├── EmployeeService.java
    │   ├── EmployeeServiceImpl.java
    │   ├── UserService.java
    │   └── AuditService.java
    │
    ├── dao
    │   ├── EmployeeDAO.java
    │   ├── EmployeeDAOImpl.java
    │   ├── UserDAO.java
    │   └── AuditDAO.java
    │
    ├── entity
    │   ├── Employee.java
    │   ├── Department.java
    │   ├── User.java
    │   └── AuditLog.java
    │
    ├── dto
    │   ├── EmployeeCreateRequest.java
    │   ├── EmployeeUpdateRequest.java
    │   ├── EmployeeResponse.java
    │   └── EmployeeSearchRequest.java
    │
    ├── filter
    │   ├── AuthenticationFilter.java
    │   ├── AuthorizationFilter.java
    │   ├── LoggingFilter.java
    │   └── EncodingFilter.java
    │
    ├── exception
    │   ├── EmployeeNotFoundException.java
    │   ├── ValidationException.java
    │   └── DuplicateEmailException.java
    │
    ├── mapper
    │   └── EmployeeMapper.java
    │
    └── util
        ├── PasswordUtil.java
        └── DatabaseUtil.java
```

---

# 39. Maven Structure

```text
employee-management/
│
├── pom.xml
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   ├── resources/
│   │   │   └── META-INF/
│   │   │       └── persistence.xml
│   │   │
│   │   └── webapp/
│   │       ├── css/
│   │       ├── js/
│   │       └── WEB-INF/
│   │           ├── views/
│   │           └── web.xml
│   │
│   └── test/
│       └── java/
│
└── README.md
```

---

# 40. `web.xml`

Even though annotations are preferred for many mappings, understand `web.xml`.

Example:

```xml
<web-app>

    <display-name>
        Employee Management System
    </display-name>

    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>

</web-app>
```

---

# 41. Persistence Configuration

For JPA, understand:

```text
META-INF/persistence.xml
```

Conceptually:

```xml
<persistence>

    <persistence-unit
        name="employeePU">

        <class>
            com.example.ems.entity.Employee
        </class>

        <class>
            com.example.ems.entity.Department
        </class>

    </persistence-unit>

</persistence>
```

Exact configuration differs between application servers and whether you're using resource-local or JTA persistence.

---

# 42. Employee Service Transaction

Conceptually:

```java
public void createEmployee(
        EmployeeCreateRequest request) {

    validate(request);

    Department department =
            departmentDao.findById(
                request.getDepartmentId()
            );

    if (department == null) {
        throw new DepartmentNotFoundException();
    }

    Employee employee =
            employeeMapper.toEntity(request);

    employee.setDepartment(department);

    employeeDao.save(employee);

    auditService.record(
        "CREATE_EMPLOYEE",
        employee.getId()
    );
}
```

The transaction should encompass the operations that must succeed or fail together.

---

# 43. Update Employee

Flow:

```text
GET /employees/edit?id=10
```

loads the form.

Then:

```text
POST /employees/edit
```

performs:

```text
Validate
 ↓
Find employee
 ↓
Check version
 ↓
Update fields
 ↓
Save
 ↓
Audit
 ↓
Commit
 ↓
Redirect
```

---

# 44. Optimistic Locking

Your Employee entity has:

```java
@Version
private Long version;
```

Suppose:

```text
Admin A → version 5
Admin B → version 5
```

Admin A updates:

```text
5 → 6
```

Admin B attempts to update using stale version 5.

The update should fail rather than silently overwrite A's changes.

Your application should convert that persistence conflict into a meaningful user-facing response.

---

# 45. Delete Employee

Don't immediately assume physical deletion is correct.

Consider:

```text
DELETE employee
```

versus:

```text
status = TERMINATED
```

For many enterprise systems, historical records need to remain available.

This is a **business requirement**, not merely a database decision.

---

# 46. Soft Delete

Example:

```text
active = false
```

or:

```text
status = TERMINATED
```

Then:

```sql
SELECT *
FROM employee
WHERE status <> 'TERMINATED';
```

But soft deletion introduces its own complexity:

```text
unique constraints
queries
restoration
reporting
data retention
```

Don't use it automatically; decide based on requirements.

---

# 47. Security Requirements

Your application should protect against:

```text
SQL Injection
XSS
CSRF
Session Fixation
Broken Access Control
Credential Leakage
Sensitive Data Exposure
```

Important:

```text
PreparedStatement
+
output escaping
+
CSRF protection
+
secure session management
+
authorization checks
```

---

# 48. Session Security

After successful login, ensure the application doesn't continue using an attacker-controlled pre-authentication session identifier.

Modern authentication systems commonly mitigate session fixation by changing or renewing the session identifier at authentication time.

On logout:

```java
session.invalidate();
```

Then:

```text
redirect → /login
```

---

# 49. CSRF

Suppose:

```text
POST /employees/delete
```

An attacker shouldn't be able to trick an authenticated browser into submitting arbitrary state-changing requests.

Use a CSRF token:

```html
<input
    type="hidden"
    name="csrfToken"
    value="${csrfToken}">
```

Then validate it server-side.

---

# 50. XSS

Never blindly output untrusted data.

Bad conceptual pattern:

```text
raw user input → HTML
```

Prefer proper HTML escaping through appropriate view mechanisms.

For JSTL:

```jsp
<c:out value="${employee.firstName}" />
```

is preferable to raw output where escaping is required.

---

# 51. Connection Pool

Don't create a new physical database connection for every request if your production architecture provides a connection pool.

Conceptually:

```text
Application
     ↓
DataSource
     ↓
Connection Pool
 ┌───┼───┬───┐
 C1  C2  C3  C4
```

Request:

```text
borrow connection
      ↓
execute
      ↓
commit/rollback
      ↓
return connection
```

---

# 52. Error Handling Architecture

```text
                    Exception
                       ↓
              ┌────────┴────────┐
              ↓                 ↓
        Business Error     Technical Error
              ↓                 ↓
         4xx response       500 response
              ↓                 ↓
        User-friendly       Generic message
```

Never expose:

```text
SQL stack traces
database credentials
internal paths
implementation details
```

to users.

---

# 53. Testing Strategy

Test each layer.

## DAO tests

Test:

```text
save
find
update
delete
search
pagination
```

## Service tests

Test:

```text
business rules
validation
transactions
exception scenarios
```

## Servlet tests

Test:

```text
request parameters
response status
redirect
forward
session behavior
```

---

# 54. Test Scenarios

You should implement at least:

```text
Create employee successfully
Duplicate email
Invalid email
Missing department
Employee not found
Update employee
Delete employee
Unauthorized request
Unauthenticated request
Pagination
Search
Concurrent update
Transaction rollback
```

---

# 55. Transaction Rollback Test

Create a deliberate failure:

```text
BEGIN
 ↓
Create employee
 ↓
Create audit record
 ↓
Throw exception
 ↓
ROLLBACK
```

Verify:

```text
employee → NOT persisted
audit → NOT persisted
```

This is one of the most important exercises today.

---

# 56. Integration Test

Test:

```text
HTTP
 ↓
Servlet
 ↓
Service
 ↓
DAO
 ↓
Database
```

instead of testing only individual classes.

This teaches you the difference between:

```text
Unit Test
```

and:

```text
Integration Test
```

---

# 57. Git Strategy

Create:

```text
main
develop
feature/authentication
feature/employee-crud
feature/search
feature/security
feature/testing
```

Example:

```bash
git checkout -b feature/employee-crud
```

Commit small logical changes:

```text
feat: add employee entity
feat: add employee dao
feat: add employee service
feat: add employee create servlet
feat: add employee list jsp
test: add employee service tests
```

---

# 58. Day 14 Implementation Sequence

Don't try to build everything simultaneously.

Follow this sequence:

### Step 1

Create Maven project.

### Step 2

Configure application server.

### Step 3

Configure database.

### Step 4

Create schema.

### Step 5

Create entities.

### Step 6

Create DAO layer.

### Step 7

Create Service layer.

### Step 8

Create Employee List.

### Step 9

Create Employee Create.

### Step 10

Create Employee Update.

### Step 11

Create Employee Delete.

### Step 12

Add JSP + EL + JSTL.

### Step 13

Add authentication.

### Step 14

Add authorization.

### Step 15

Add filters.

### Step 16

Add transactions.

### Step 17

Add optimistic locking.

### Step 18

Add validation.

### Step 19

Add exception handling.

### Step 20

Add logging/auditing.

### Step 21

Add tests.

### Step 22

Package as WAR.

---

# 59. Final Request Flow

You should be able to draw this from memory:

```text
                  Browser
                     │
                     ▼
                HTTP Request
                     │
                     ▼
              ┌──────────────┐
              │ Filter Chain  │
              └──────┬───────┘
                     │
                     ▼
             Controller Servlet
                     │
                     ▼
                DTO / Input
                     │
                     ▼
               Service Layer
                     │
               Transaction
                     │
                     ▼
                 DAO Layer
                     │
              ┌──────┴───────┐
              ▼              ▼
            JDBC            JPA
              │              │
              └──────┬───────┘
                     ▼
                  Database
                     │
                     ▼
                 Commit
                     │
                     ▼
               Service Layer
                     │
                     ▼
                Controller
                     │
                     ▼
                  JSP
                     │
                 EL/JSTL
                     │
                     ▼
                  Browser
```

---

# 60. What You Should Be Able to Explain After Day 14

### Architecture

* What is MVC?
* What is layered architecture?
* Why Service layer?
* Why DAO layer?
* Why DTO?
* What is Front Controller?
* Where should transactions live?

### Servlets

* Servlet lifecycle
* Filters
* Listeners
* RequestDispatcher
* Forward vs redirect
* Session management

### JSP

* JSP lifecycle
* EL
* JSTL
* scopes
* `WEB-INF`

### Database

* JDBC
* JPA
* EntityManager
* Persistence Context
* Lazy loading
* Transactions
* Optimistic locking
* Pagination

### Security

* Authentication
* Authorization
* Sessions
* CSRF
* XSS
* SQL injection
* Session fixation

### Enterprise architecture

* Transaction boundary
* Connection pooling
* Audit logging
* Exception handling
* DTO boundaries
* Concurrency
* Idempotency
* External side effects

---

# 61. Day 14 Interview Questions

### Q1. Why shouldn't a Servlet directly access the database?

Because it couples HTTP handling with persistence and makes testing, reuse, transaction management, and maintenance harder.

### Q2. Why use a Service layer?

To encapsulate business operations and coordinate multiple persistence operations within meaningful transaction boundaries.

### Q3. Why use DTOs?

To control data crossing application boundaries and avoid exposing persistence models directly.

### Q4. Where should a transaction normally begin?

Around a meaningful business operation, commonly at the service/application layer.

### Q5. Why use PRG?

To prevent duplicate form submission after a successful POST.

### Q6. Why use filters?

To implement cross-cutting concerns without duplicating them in every Servlet.

### Q7. Authentication vs authorization?

```text
Authentication = Who are you?
Authorization = What can you do?
```

### Q8. Why use `@Version`?

To detect conflicting concurrent updates using optimistic locking.

### Q9. Why use pagination?

To prevent unnecessarily loading huge result sets into application memory and reducing database/application workload.

### Q10. Why shouldn't JSP contain business logic?

Because it violates separation of concerns and makes presentation difficult to test and maintain.

---

# 62. Architect-Level Questions

### Scenario 1

> Employee creation updates the employee table and audit table. The audit insert fails. What should happen?

If both operations are required to be atomic, the transaction should roll back the employee creation as well.

---

### Scenario 2

> Two HR users edit the same employee simultaneously. How do you prevent silent overwrites?

Use optimistic locking with a version field or another concurrency-control strategy appropriate to the business requirement.

---

### Scenario 3

> Employee creation needs to call an external notification API.

Don't automatically put the external API call inside the database transaction.

Consider:

```text
DB transaction
      ↓
Commit
      ↓
Outbox/event
      ↓
Notification service
```

---

### Scenario 4

> The employee table has 10 million rows.

Consider:

```text
indexes
pagination
keyset pagination
query optimization
projection
filtering
database execution plans
connection pool sizing
```

---

### Scenario 5

> Your application has 20 Servlets with duplicate authentication code.

Introduce an:

```text
AuthenticationFilter
```

instead of duplicating authentication logic.

---

# 63. Your Day 14 Mastery Test

Without looking at your notes, draw:

```text
Browser
 ↓
Filters
 ↓
Servlet
 ↓
DTO
 ↓
Service
 ↓
Transaction
 ↓
DAO
 ↓
JPA/JDBC
 ↓
Database
```

Then explain:

1. Where authentication happens.
2. Where authorization happens.
3. Where validation happens.
4. Where the transaction starts.
5. Where SQL/JPA belongs.
6. How rollback happens.
7. How exceptions reach the user.
8. How JSP receives data.
9. How CSRF is prevented.
10. How concurrent employee updates are handled.
11. How audit logging works.
12. How the system would scale to millions of employees.

If you can explain all 12 clearly, you've moved from **learning individual J2EE APIs to thinking in enterprise application architecture**.

---

## Day 14 Golden Mental Model

```text
             ENTERPRISE APPLICATION
                      │
      ┌───────────────┼────────────────┐
      ↓               ↓                ↓
   Security       Business          Persistence
      │               │                │
   Filters         Service           DAO
   Sessions        Rules             JPA/JDBC
   Auth            DTOs              Database
      │               │                │
      └───────────────┼────────────────┘
                      ↓
                  Transaction
                      ↓
                 Consistent
                   State
                      ↓
                    View
                      ↓
                  JSP/EL/JSTL
```

**The key lesson of Day 14:** don't think of J2EE as a collection of technologies. Think of it as a way to structure **HTTP handling + security + business logic + transactions + persistence + presentation** into a coherent enterprise system.

**Day 15** moves into **REST APIs**, where you'll take the same enterprise concepts and expose business capabilities as HTTP APIs rather than JSP-based web pages.
