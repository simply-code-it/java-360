# Day 15 — REST APIs with J2EE / Jakarta EE

**Date:** October 17, 2026
**Phase:** J2EE/Jakarta EE Mastery — REST & Web Services
**Goal:** Learn how to design, implement, secure, validate, document, test, and productionize REST APIs using Jakarta EE.

Today is important because you're transitioning from:

```text
Browser → Servlet → JSP → HTML
```

to:

```text
Client → HTTP → REST API → Service → DAO → Database → JSON
```

---

# 1. What Is a REST API?

**REST** stands for **Representational State Transfer**.

REST is an architectural style for designing networked applications around **resources** and standard HTTP semantics.

Example employee resource:

```text
/api/employees
```

Individual employee:

```text
/api/employees/101
```

Instead of creating RPC-style URLs such as:

```text
/api/createEmployee
/api/updateEmployee
/api/deleteEmployee
```

REST commonly models the resource:

```text
/api/employees
```

and uses HTTP methods to describe the operation.

---

# 2. REST Mental Model

Think:

```text
Resource + HTTP Method + Representation
```

Example:

```text
GET /api/employees/101
```

means:

> Retrieve the representation of employee 101.

Response:

```json
{
  "id": 101,
  "firstName": "Rahul",
  "lastName": "Sharma",
  "email": "rahul@example.com"
}
```

---

# 3. Why REST?

Traditional web application:

```text
Browser
   ↓
Servlet
   ↓
JSP
   ↓
HTML
```

REST application:

```text
Browser / Mobile / React / Another Service
                 ↓
              HTTP
                 ↓
              REST API
                 ↓
              Service
                 ↓
              DAO
                 ↓
             Database
```

The API can serve many different clients:

```text
Web UI
Mobile App
React
Angular
Another Backend
Partner System
IoT Device
```

---

# 4. REST vs Servlet

Servlets provide a low-level HTTP programming model.

You manually work with:

```java
HttpServletRequest
HttpServletResponse
```

REST frameworks such as **Jakarta REST** provide higher-level abstractions.

Conceptually:

```text
HTTP
 ↓
Jakarta REST
 ↓
Resource class
 ↓
Service
 ↓
DAO
```

You can therefore think of:

```text
Servlet
```

as the underlying web mechanism and:

```text
Jakarta REST resource
```

as a REST-oriented programming model built on top of the web runtime.

---

# 5. Jakarta REST

Modern Jakarta EE uses:

```text
Jakarta RESTful Web Services
```

Historically, this API was known as:

```text
JAX-RS
```

Namespace transition:

```text
javax.ws.rs.*
        ↓
jakarta.ws.rs.*
```

For modern Jakarta EE applications, you'll typically see:

```java
import jakarta.ws.rs.*;
```

---

# 6. First REST Endpoint

A basic resource:

```java
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/hello")
public class HelloResource {

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "Hello Jakarta REST";
    }
}
```

Request:

```text
GET /api/hello
```

Response:

```text
Hello Jakarta REST
```

---

# 7. REST Application Path

You can define a base API path.

```java
import jakarta.ws.rs.ApplicationPath;
import jakarta.ws.rs.core.Application;

@ApplicationPath("/api")
public class RestApplication
        extends Application {
}
```

Then:

```java
@Path("/hello")
```

becomes:

```text
/api/hello
```

Final URL:

```text
GET /api/hello
```

---

# 8. `@Path`

`@Path` maps HTTP paths to Java resources.

```java
@Path("/employees")
public class EmployeeResource {
}
```

This creates the base resource:

```text
/api/employees
```

---

# 9. `@GET`

```java
@GET
public List<EmployeeResponse> getEmployees() {
    ...
}
```

Handles:

```text
GET /api/employees
```

---

# 10. `@POST`

```java
@POST
public Response createEmployee(...) {
    ...
}
```

Handles:

```text
POST /api/employees
```

Typically used to create a new resource.

---

# 11. `@PUT`

```java
@PUT
@Path("/{id}")
public Response updateEmployee(...) {
    ...
}
```

Handles:

```text
PUT /api/employees/101
```

Usually represents replacement/update semantics.

---

# 12. `@PATCH`

PATCH is commonly used for partial modification.

Depending on your Jakarta REST runtime, PATCH may require explicit support or custom handling rather than assuming a built-in annotation exists.

Conceptually:

```text
PATCH /api/employees/101
```

Request:

```json
{
  "phone": "9876543210"
}
```

Only selected fields are modified.

---

# 13. `@DELETE`

```java
@DELETE
@Path("/{id}")
public Response deleteEmployee(
        @PathParam("id") Long id) {

    ...
}
```

Request:

```text
DELETE /api/employees/101
```

---

# 14. CRUD API

Your Employee API can become:

```text
GET     /api/employees
GET     /api/employees/{id}

POST    /api/employees

PUT     /api/employees/{id}

DELETE  /api/employees/{id}
```

Mental model:

```text
GET    → Read
POST   → Create
PUT    → Replace/update
DELETE → Delete
```

---

# 15. Path Parameters

URL:

```text
GET /api/employees/101
```

Resource:

```java
@GET
@Path("/{id}")
public EmployeeResponse getEmployee(
        @PathParam("id") Long id) {

    return employeeService.findById(id);
}
```

The value:

```text
101
```

is extracted from the URL.

---

# 16. Query Parameters

Request:

```text
GET /api/employees?departmentId=10
```

Code:

```java
@GET
public List<EmployeeResponse> search(
        @QueryParam("departmentId")
        Long departmentId) {

    return employeeService.search(
        departmentId
    );
}
```

---

# 17. Path Parameter vs Query Parameter

### Path parameter

Identifies a resource:

```text
/api/employees/101
```

### Query parameter

Usually modifies/filter/searches a collection:

```text
/api/employees?departmentId=10
```

Think:

```text
/employees/{id}
```

versus:

```text
/employees?filter=value
```

---

# 18. Multiple Query Parameters

```text
GET /api/employees
    ?name=Rahul
    &departmentId=10
    &status=ACTIVE
```

Code:

```java
@GET
public List<EmployeeResponse> search(
        @QueryParam("name") String name,
        @QueryParam("departmentId") Long departmentId,
        @QueryParam("status") String status) {

    return employeeService.search(
        name,
        departmentId,
        status
    );
}
```

---

# 19. Headers

HTTP request:

```text
Authorization: Bearer <token>
Accept: application/json
X-Request-ID: abc123
```

Read headers:

```java
@GET
public Response getEmployees(
        @HeaderParam("X-Request-ID")
        String requestId) {

    ...
}
```

Headers are useful for:

```text
authentication
content negotiation
tracing
caching
client metadata
```

---

# 20. Request Body

POST:

```text
POST /api/employees
Content-Type: application/json
```

Body:

```json
{
  "firstName": "Rahul",
  "lastName": "Sharma",
  "email": "rahul@example.com",
  "salary": 75000,
  "departmentId": 10
}
```

Create a DTO:

```java
public class EmployeeCreateRequest {

    private String firstName;
    private String lastName;
    private String email;
    private BigDecimal salary;
    private Long departmentId;

    // getters/setters
}
```

---

# 21. JSON Mapping

Jakarta REST implementations commonly use a JSON provider to convert:

```text
JSON → Java object
```

and:

```text
Java object → JSON
```

So your resource can conceptually receive:

```java
public Response create(
        EmployeeCreateRequest request) {
    ...
}
```

instead of manually parsing JSON strings.

---

# 22. `@Consumes`

Specify what the endpoint accepts.

```java
@POST
@Consumes(MediaType.APPLICATION_JSON)
public Response create(
        EmployeeCreateRequest request) {

    ...
}
```

This means:

```text
Content-Type: application/json
```

is expected.

---

# 23. `@Produces`

Specify the response representation.

```java
@GET
@Produces(MediaType.APPLICATION_JSON)
public List<EmployeeResponse> getEmployees() {
    ...
}
```

Response:

```text
Content-Type: application/json
```

---

# 24. Content Negotiation

Client:

```text
Accept: application/json
```

Server:

```text
Content-Type: application/json
```

Conceptually:

```text
Client
  ↓
Accept: application/json
  ↓
REST framework
  ↓
JSON representation
```

A client might request another supported representation if your API provides one.

---

# 25. `Response`

Instead of returning only an object:

```java
return employee;
```

you can explicitly construct an HTTP response.

```java
return Response
        .status(Response.Status.OK)
        .entity(employee)
        .build();
```

---

# 26. HTTP Status Codes

Your API should use meaningful HTTP status codes.

### Success

```text
200 OK
201 Created
202 Accepted
204 No Content
```

### Client errors

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Content
429 Too Many Requests
```

### Server errors

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

---

# 27. Create Employee

Correct response:

```text
HTTP/1.1 201 Created
```

Example:

```java
return Response
        .status(Response.Status.CREATED)
        .entity(employee)
        .build();
```

Ideally, the response can also provide the location of the newly created resource.

Conceptually:

```text
Location: /api/employees/101
```

---

# 28. Get Employee

Successful:

```text
HTTP 200 OK
```

Not found:

```text
HTTP 404 Not Found
```

Example:

```java
EmployeeResponse employee =
        service.findById(id);

if (employee == null) {
    return Response
            .status(Response.Status.NOT_FOUND)
            .build();
}

return Response.ok(employee).build();
```

---

# 29. Delete Employee

A successful delete often returns:

```text
204 No Content
```

Example:

```java
return Response
        .noContent()
        .build();
```

---

# 30. Conflict

Suppose:

```text
email = rahul@example.com
```

already exists.

Returning:

```text
500 Internal Server Error
```

would be misleading.

A more appropriate API design may use:

```text
409 Conflict
```

depending on the application's contract.

---

# 31. REST Resource Design

Bad:

```text
POST /createEmployee
POST /updateEmployee
POST /deleteEmployee
```

More REST-oriented:

```text
POST   /employees
GET    /employees
GET    /employees/101
PUT    /employees/101
DELETE /employees/101
```

The HTTP method communicates the operation.

---

# 32. Collection vs Individual Resource

Collection:

```text
/api/employees
```

Individual resource:

```text
/api/employees/101
```

Department:

```text
/api/departments/10
```

Possible relationship:

```text
/api/departments/10/employees
```

But don't over-nest URLs unnecessarily.

---

# 33. REST API Versioning

A common approach:

```text
/api/v1/employees
/api/v2/employees
```

Another approach uses headers/media types.

For example, an API might negotiate versions through a media type.

The important architectural question is:

> How will you evolve an API without unexpectedly breaking existing clients?

---

# 34. Backward Compatibility

Suppose v1 returns:

```json
{
  "id": 101,
  "name": "Rahul"
}
```

You add:

```json
{
  "id": 101,
  "name": "Rahul",
  "department": "Engineering"
}
```

Adding an optional field is often easier for clients to tolerate than changing/removing existing fields.

Avoid casually changing:

```text
field names
types
meaning
required/optional semantics
HTTP status behavior
```

---

# 35. API DTOs

Don't expose:

```java
@Entity
public class Employee {
    ...
}
```

directly unless you have deliberately accepted the coupling.

Prefer:

```text
Entity
 ↓
Mapper
 ↓
EmployeeResponse
 ↓
JSON
```

Architecture:

```text
Database
 ↓
JPA Entity
 ↓
Service
 ↓
DTO
 ↓
REST Resource
 ↓
JSON
```

---

# 36. Employee REST Resource

A basic structure:

```java
@Path("/employees")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class EmployeeResource {

    private EmployeeService employeeService;

    @GET
    public List<EmployeeResponse> findAll() {
        return employeeService.findAll();
    }

    @GET
    @Path("/{id}")
    public Response findById(
            @PathParam("id") Long id) {

        EmployeeResponse employee =
                employeeService.findById(id);

        if (employee == null) {
            return Response
                    .status(
                        Response.Status.NOT_FOUND
                    )
                    .build();
        }

        return Response.ok(employee).build();
    }
}
```

---

# 37. POST Implementation

```java
@POST
public Response create(
        EmployeeCreateRequest request) {

    EmployeeResponse employee =
            employeeService.create(request);

    return Response
            .status(Response.Status.CREATED)
            .entity(employee)
            .build();
}
```

---

# 38. PUT Implementation

```java
@PUT
@Path("/{id}")
public Response update(
        @PathParam("id") Long id,
        EmployeeUpdateRequest request) {

    EmployeeResponse employee =
            employeeService.update(
                id,
                request
            );

    return Response
            .ok(employee)
            .build();
}
```

---

# 39. DELETE Implementation

```java
@DELETE
@Path("/{id}")
public Response delete(
        @PathParam("id") Long id) {

    employeeService.delete(id);

    return Response
            .noContent()
            .build();
}
```

---

# 40. Full REST Architecture

Your application now becomes:

```text
                     Client
                       │
                       ▼
                    HTTP
                       │
                       ▼
               Jakarta REST
                       │
                       ▼
               EmployeeResource
                       │
                       ▼
                EmployeeService
                       │
                Transaction
                       │
                       ▼
                  EmployeeDAO
                       │
                ┌──────┴──────┐
                ▼             ▼
              JPA           JDBC
                │             │
                └──────┬──────┘
                       ▼
                    Database
```

---

# 41. REST vs JSP Architecture

You now have two possible interfaces.

### Web UI

```text
Browser
 ↓
Servlet
 ↓
Service
 ↓
DAO
 ↓
DB
 ↓
JSP
 ↓
HTML
```

### REST

```text
Client
 ↓
REST Resource
 ↓
Service
 ↓
DAO
 ↓
DB
 ↓
JSON
```

Notice:

```text
             Service
             /     \
        Servlet    REST
          ↓          ↓
        JSP         JSON
```

This is powerful because your business logic is reusable.

---

# 42. Exception Handling

Don't repeat:

```java
try {
    ...
} catch (...) {
    return Response
        .status(500)
        .build();
}
```

in every resource method.

Instead, use centralized exception mapping.

Jakarta REST provides:

```text
ExceptionMapper<T>
```

---

# 43. `ExceptionMapper`

Example:

```java
@Provider
public class EmployeeNotFoundMapper
        implements ExceptionMapper<
            EmployeeNotFoundException> {

    @Override
    public Response toResponse(
            EmployeeNotFoundException ex) {

        return Response
                .status(
                    Response.Status.NOT_FOUND
                )
                .build();
    }
}
```

Now:

```java
throw new EmployeeNotFoundException();
```

can be converted into:

```text
404 Not Found
```

---

# 44. Error Response Design

Don't return:

```text
Internal Server Error
```

with no useful machine-readable structure.

Prefer a consistent structure:

```json
{
  "timestamp": "2026-10-17T10:30:00Z",
  "status": 404,
  "code": "EMPLOYEE_NOT_FOUND",
  "message": "Employee was not found",
  "path": "/api/employees/101",
  "requestId": "abc123"
}
```

Be careful not to expose internal exception details.

---

# 45. Validation

Validate request DTOs.

Conceptually:

```java
public class EmployeeCreateRequest {

    @NotBlank
    private String firstName;

    @NotBlank
    private String lastName;

    @Email
    @NotBlank
    private String email;

    @Positive
    private BigDecimal salary;
}
```

Validation APIs can automatically reject invalid input when integrated into the REST runtime.

---

# 46. Validation Flow

```text
HTTP Request
     ↓
JSON deserialization
     ↓
Validation
     ↓
Controller
     ↓
Service
     ↓
DAO
```

Invalid request:

```text
400 Bad Request
```

Don't let obviously invalid input travel deep into your system.

---

# 47. Pagination API

Don't expose millions of records:

```text
GET /api/employees
```

with an unlimited result.

Use:

```text
GET /api/employees?page=0&size=20
```

Response:

```json
{
  "content": [
    {}
  ],
  "page": 0,
  "size": 20,
  "totalElements": 10000,
  "totalPages": 500
}
```

---

# 48. Sorting

Example:

```text
GET /api/employees
    ?page=0
    &size=20
    &sort=lastName
```

Multiple sort fields may be supported according to your API contract.

Always whitelist allowed sort fields rather than blindly injecting client-provided column names into SQL.

---

# 49. Filtering

Example:

```text
GET /api/employees
    ?departmentId=10
    &status=ACTIVE
```

Service:

```text
EmployeeSearchRequest
        ↓
DAO
        ↓
Database
```

Don't implement complex filtering logic directly in the REST resource.

---

# 50. Search API

Possible design:

```text
GET /api/employees?search=rahul
```

or a dedicated search endpoint if the query semantics are complex.

For simple resource filtering, query parameters are usually natural.

---

# 51. Idempotency

Important REST concept.

### GET

Should be safe and idempotent under normal HTTP semantics.

### PUT

Designed to be idempotent.

Example:

```text
PUT /employees/101
```

with the same representation repeatedly should result in the same intended resource state.

### DELETE

Also intended to be idempotent.

### POST

Usually **not inherently idempotent**.

Therefore payment/order APIs often need:

```text
Idempotency-Key
```

to safely handle retries.

---

# 52. Safe vs Idempotent

Don't confuse these.

### Safe

Doesn't intentionally change server state.

Examples:

```text
GET
HEAD
OPTIONS
```

### Idempotent

Repeating the same request has the same intended effect on server state.

Common HTTP methods considered idempotent:

```text
GET
HEAD
PUT
DELETE
OPTIONS
```

`POST` is generally not inherently idempotent.

---

# 53. Authentication

Your REST API may use:

```text
Session authentication
Basic authentication
API keys
OAuth 2.0
OpenID Connect
JWT-based bearer tokens
Mutual TLS
```

The appropriate choice depends on the client and architecture.

Example bearer token:

```text
Authorization:
Bearer eyJ...
```

---

# 54. Authorization

Authentication:

```text
Who is calling?
```

Authorization:

```text
What can they do?
```

Example:

```text
GET /api/employees
```

may be accessible to:

```text
HR
ADMIN
MANAGER
```

while:

```text
DELETE /api/employees/101
```

may require an appropriate administrative permission.

---

# 55. CORS

Suppose:

```text
Frontend:
https://app.example.com

API:
https://api.example.com
```

Different origins can require appropriate CORS configuration.

Conceptually:

```text
Browser
   ↓
Cross-Origin Request
   ↓
API
   ↓
CORS Policy
   ↓
Allow / Reject
```

Don't configure:

```text
Access-Control-Allow-Origin: *
```

blindly for authenticated/private APIs.

---

# 56. CSRF and REST

CSRF risk depends on the authentication mechanism and how credentials are automatically attached.

For example:

```text
Cookie-based authentication
```

has different CSRF considerations from:

```text
Authorization: Bearer ...
```

where the browser doesn't automatically attach the token in the same way.

Security should therefore be designed around the actual authentication architecture rather than applying one generic rule.

---

# 57. Rate Limiting

Your API may need:

```text
100 requests/minute/client
```

to protect infrastructure.

Possible dimensions:

```text
IP
user
API key
client application
endpoint
tenant
```

Architecture:

```text
Client
 ↓
Rate Limiter
 ↓
REST API
 ↓
Service
```

---

# 58. Request Correlation

From Day 14:

```text
X-Request-ID: abc123
```

Filter:

```text
Request
 ↓
Generate/validate ID
 ↓
REST Resource
 ↓
Service
 ↓
DAO
```

Logs:

```text
requestId=abc123
```

This becomes extremely useful once you have multiple services.

---

# 59. REST API Logging

Log useful metadata:

```text
timestamp
requestId
method
path
status
duration
user/client
```

Avoid logging:

```text
password
access tokens
session IDs
sensitive payloads
```

unless there is a carefully controlled security/compliance reason.

---

# 60. API Documentation

Production APIs should be documented.

A common industry approach is:

```text
OpenAPI
```

Documentation should describe:

```text
endpoints
parameters
request bodies
responses
status codes
authentication
error formats
examples
```

You should learn Swagger/OpenAPI because you'll encounter it frequently in Java enterprise development.

---

# 61. REST API Testing

Use:

```text
curl
Postman
HTTP client
automated integration tests
```

Example:

```bash
curl http://localhost:8080/employee-app/api/employees
```

Create:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Rahul","lastName":"Sharma","email":"rahul@example.com"}' \
  http://localhost:8080/employee-app/api/employees
```

---

# 62. API Test Matrix

Test:

```text
GET existing employee
GET nonexistent employee
POST valid employee
POST invalid employee
POST duplicate email
PUT existing employee
PUT nonexistent employee
DELETE existing employee
DELETE nonexistent employee
Unauthorized request
Forbidden request
Malformed JSON
Missing fields
Invalid types
Large pagination request
```

---

# 63. Transaction Integration

Your REST endpoint:

```text
POST /api/employees
```

should call:

```text
EmployeeService.create()
```

where the business transaction occurs.

Flow:

```text
POST
 ↓
REST Resource
 ↓
DTO validation
 ↓
Service
 ↓
BEGIN
 ↓
Employee DAO
 ↓
Audit DAO
 ↓
COMMIT
```

Failure:

```text
ROLLBACK
```

---

# 64. Don't Put Transactions in the REST Resource

Avoid:

```java
@POST
public Response create(...) {

    transaction.begin();

    ...

    transaction.commit();
}
```

The REST resource should focus on HTTP/API concerns.

Better:

```text
REST Resource
     ↓
Service
     ↓
Transaction
```

This allows the same business operation to be reused by:

```text
REST
Servlet
Batch job
Messaging consumer
CLI
```

---

# 65. Complete API Architecture

Your midpoint application now evolves into:

```text
                         Clients
                   /        |        \
                  /         |         \
              Browser     Mobile    Another API
                  \         |         /
                   \        |        /
                         HTTP
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
            Web Layer             REST Layer
            Servlet              Jakarta REST
                │                     │
                └──────────┬──────────┘
                           ▼
                     Service Layer
                           │
                     Transaction
                           │
                           ▼
                       DAO Layer
                      /         \
                     ▼           ▼
                   JPA         JDBC
                     \           /
                      \         /
                       ▼       ▼
                        Database
```

This is a major architectural milestone.

---

# 66. REST API Folder Structure

Extend your project:

```text
src/main/java/
└── com.example.ems
    │
    ├── rest
    │   ├── RestApplication.java
    │   │
    │   ├── EmployeeResource.java
    │   ├── DepartmentResource.java
    │   └── AuthResource.java
    │
    ├── dto
    │   ├── EmployeeCreateRequest.java
    │   ├── EmployeeUpdateRequest.java
    │   ├── EmployeeResponse.java
    │   └── ErrorResponse.java
    │
    ├── exception
    │   ├── EmployeeNotFoundException.java
    │   ├── ValidationException.java
    │   └── DuplicateEmailException.java
    │
    ├── exceptionmapper
    │   ├── EmployeeNotFoundMapper.java
    │   └── GenericExceptionMapper.java
    │
    ├── service
    ├── dao
    ├── entity
    ├── mapper
    └── filter
```

---

# 67. API Endpoints for Your Project

Design these:

### Authentication

```text
POST /api/auth/login
POST /api/auth/logout
GET  /api/auth/me
```

### Employees

```text
GET    /api/employees
GET    /api/employees/{id}
POST   /api/employees
PUT    /api/employees/{id}
DELETE /api/employees/{id}
```

### Departments

```text
GET    /api/departments
GET    /api/departments/{id}
POST   /api/departments
PUT    /api/departments/{id}
DELETE /api/departments/{id}
```

---

# 68. API Contract Example

## Create Employee

```text
POST /api/employees
```

Request:

```json
{
  "firstName": "Rahul",
  "lastName": "Sharma",
  "email": "rahul@example.com",
  "phone": "9876543210",
  "salary": 75000,
  "departmentId": 10
}
```

Response:

```text
201 Created
```

```json
{
  "id": 101,
  "firstName": "Rahul",
  "lastName": "Sharma",
  "email": "rahul@example.com",
  "salary": 75000,
  "departmentId": 10,
  "status": "ACTIVE"
}
```

---

# 69. Get Employee

```text
GET /api/employees/101
```

Response:

```text
200 OK
```

```json
{
  "id": 101,
  "firstName": "Rahul",
  "lastName": "Sharma",
  "email": "rahul@example.com",
  "departmentId": 10,
  "status": "ACTIVE"
}
```

---

# 70. Not Found

```text
GET /api/employees/99999
```

Response:

```text
404 Not Found
```

```json
{
  "status": 404,
  "code": "EMPLOYEE_NOT_FOUND",
  "message": "Employee was not found"
}
```

---

# 71. Validation Error

Request:

```json
{
  "firstName": "",
  "email": "invalid"
}
```

Response:

```text
400 Bad Request
```

Possible structure:

```json
{
  "status": 400,
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "errors": [
    {
      "field": "firstName",
      "message": "must not be blank"
    },
    {
      "field": "email",
      "message": "must be a valid email"
    }
  ]
}
```

---

# 72. Day 15 Mini Project

## Convert Employee Management System into an API Platform

Keep your existing JSP application:

```text
/employee-app/employees
```

and add:

```text
/employee-app/api/employees
```

Now you have two interfaces:

```text
             EmployeeService
              /           \
             /             \
        JSP Web UI       REST API
            │                │
         HTML             JSON
```

The business layer remains shared.

---

# 73. Implementation Tasks

### Task 1

Create `RestApplication`.

### Task 2

Create `EmployeeResource`.

### Task 3

Implement:

```text
GET /api/employees
```

### Task 4

Implement:

```text
GET /api/employees/{id}
```

### Task 5

Implement:

```text
POST /api/employees
```

### Task 6

Implement:

```text
PUT /api/employees/{id}
```

### Task 7

Implement:

```text
DELETE /api/employees/{id}
```

### Task 8

Add DTOs.

### Task 9

Add validation.

### Task 10

Add exception mappers.

### Task 11

Add pagination.

### Task 12

Add filtering.

### Task 13

Add authentication.

### Task 14

Add authorization.

### Task 15

Add request correlation ID.

### Task 16

Add API documentation.

### Task 17

Write integration tests.

---

# 74. 15 Hands-On Exercises

### Easy

1. Create `/api/hello`.
2. Create `/api/employees`.
3. Implement GET by ID.
4. Implement POST.
5. Implement DELETE.

### Medium

6. Add query filtering.
7. Add pagination.
8. Add sorting.
9. Add DTO mapping.
10. Add validation.

### Advanced

11. Add centralized exception handling.
12. Add authentication/authorization.
13. Add optimistic locking.
14. Add idempotency to an order/payment-style endpoint.
15. Implement transactional outbox for an `EmployeeCreated` event.

---

# 75. Interview Questions

### Q1. What is REST?

An architectural style for designing networked systems around resources, representations, and standard HTTP semantics.

### Q2. What is JAX-RS?

The former name for the Java/Jakarta API used to build RESTful web services. Modern Jakarta EE uses Jakarta REST.

### Q3. `@Path`?

Maps a URI path to a REST resource or method.

### Q4. `@GET`?

Maps an HTTP GET request.

### Q5. `@POST`?

Typically used to create/process a resource or subordinate operation.

### Q6. `@PUT`?

Typically used for idempotent replacement/update semantics.

### Q7. `@DELETE`?

Used to delete a resource.

### Q8. `@Produces`?

Defines supported response media types.

### Q9. `@Consumes`?

Defines supported request media types.

### Q10. `@PathParam` vs `@QueryParam`?

```text
@PathParam
→ value embedded in path

@QueryParam
→ value in query string
```

---

# 76. Advanced Interview Questions

### Q11. Why use DTOs instead of entities?

To prevent API contracts from being tightly coupled to persistence models and to control exactly what clients can send/receive.

### Q12. Why use `ExceptionMapper`?

To centralize conversion of application exceptions into consistent HTTP responses.

### Q13. Where should transactions live?

Generally around meaningful business operations in the service/application layer rather than inside REST resource methods.

### Q14. What is idempotency?

Repeated execution of the same operation produces the same intended state/effect.

### Q15. Why is POST generally not idempotent?

Repeated POST requests can create multiple resources or trigger multiple effects unless the application adds idempotency semantics.

---

# 77. Architect-Level Questions

### Q16

> Your REST API and JSP application need the same employee business logic. How do you avoid duplication?

```text
REST ──────┐
           ↓
      Service Layer
           ↑
Servlet ───┘
```

Both interfaces call the same service layer.

---

### Q17

> A client sends the same payment request three times because of network timeouts. How do you prevent duplicate payment?

Use an idempotency mechanism, such as a client-provided idempotency key persisted with the operation and enforced with suitable uniqueness/transaction semantics.

---

### Q18

> Your API returns database entities directly. What problems could occur?

Potential:

```text
API/database coupling
overexposure of fields
lazy-loading problems
circular serialization
uncontrolled contract changes
```

---

### Q19

> Your API has 50 million employees. What changes?

Consider:

```text
pagination
keyset pagination
indexes
filtering
DTO projections
caching where appropriate
database query optimization
connection pool sizing
rate limiting
horizontal scaling
observability
```

---

### Q20

> POST creates an employee and publishes an event. What happens if DB commit succeeds but event publishing fails?

Consider:

```text
Transactional Outbox
```

rather than assuming the HTTP request and message broker form one atomic transaction.

---

# 78. Production REST Checklist

Before calling your API production-ready:

```text
[ ] Resource-oriented URLs
[ ] Correct HTTP methods
[ ] Correct HTTP status codes
[ ] DTOs
[ ] Validation
[ ] Consistent error responses
[ ] Exception mapping
[ ] Authentication
[ ] Authorization
[ ] CSRF considerations
[ ] CORS configuration
[ ] Rate limiting
[ ] Pagination
[ ] Filtering
[ ] Sorting
[ ] Idempotency
[ ] Transaction boundaries
[ ] Optimistic locking where needed
[ ] Request correlation
[ ] Structured logging
[ ] No sensitive information in logs
[ ] API documentation
[ ] Unit tests
[ ] Integration tests
[ ] Backward-compatible API evolution
```

---

# 79. Day 15 Mastery Test

Without looking at your notes, design this API:

```text
Employee Management API
```

Requirements:

```text
Create employee
Get employee
List employees
Search employees
Update employee
Delete employee
Authentication
Authorization
Pagination
Audit
Optimistic locking
```

You should be able to define:

```text
URLs
HTTP methods
request DTOs
response DTOs
status codes
validation
exception handling
authentication
authorization
transaction boundary
database interaction
```

---

# 80. Your New J2EE Mental Model

After Days 1–15, you should now see the entire architecture:

```text
                         CLIENTS
                  /        |        \
                 /         |         \
             Browser     Mobile    Services
                 \         |         /
                  \        |        /
                       HTTP
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
    Servlet/JSP                  Jakarta REST
       Web UI                       JSON API
          │                             │
          └──────────────┬──────────────┘
                         ▼
                  Service Layer
                         │
                 Business Rules
                         │
                  Transaction
                         │
                         ▼
                     DAO Layer
                    /         \
                   ▼           ▼
                 JDBC         JPA
                   \           /
                    \         /
                     ▼       ▼
                      DATABASE
```

### The key architectural lesson

**Servlets and REST resources are interfaces. They should not become your business layer.**

Your core business flow should remain:

```text
Client
  ↓
Controller / REST Resource
  ↓
DTO
  ↓
Service
  ↓
Transaction
  ↓
DAO
  ↓
Database
```

That separation is exactly what will allow you later to build **microservices, Spring Boot applications, event-driven systems, and larger distributed architectures** without throwing away the fundamental design principles you're learning here.

**Next: Day 16 — Jakarta REST/JAX-RS Deep Dive**, where you'll go beyond basic REST endpoints into resource design, sub-resources, providers, filters/interceptors, content negotiation, JSON serialization, exception handling, URI building, asynchronous REST processing, REST client APIs, and production-grade API architecture.
