# Day 16 — Jakarta REST / JAX-RS Deep Dive

**Date:** October 18, 2026
**Goal:** Move from simply *creating REST endpoints* to understanding how Jakarta REST works internally and how to design production-grade APIs.

> **Terminology:** JAX-RS is the historical name. In modern Jakarta EE, the specification is **Jakarta RESTful Web Services**, using `jakarta.ws.rs.*`.

---

# 1. What You Should Master Today

By the end of Day 16, you should understand:

```text
Jakarta REST architecture
        ↓
Application configuration
        ↓
Resource classes
        ↓
URI matching
        ↓
HTTP methods
        ↓
Parameters
        ↓
Request/Response
        ↓
MessageBodyReader / Writer
        ↓
Providers
        ↓
Filters
        ↓
Interceptors
        ↓
ExceptionMapper
        ↓
Content negotiation
        ↓
REST Client
        ↓
Testing
```

---

# 2. Jakarta REST Architecture

A request:

```text
GET /api/employees/101
```

flows approximately like:

```text
Client
  ↓
HTTP
  ↓
Application Server
  ↓
Jakarta REST Runtime
  ↓
URI matching
  ↓
Resource method
  ↓
Parameter injection
  ↓
Message body conversion
  ↓
Resource method execution
  ↓
Response processing
  ↓
HTTP Response
```

For example:

```text
GET /api/employees/101
```

becomes:

```java
EmployeeResource.getById(101)
```

---

# 3. Jakarta REST Application

You can define your REST application:

```java
import jakarta.ws.rs.ApplicationPath;
import jakarta.ws.rs.core.Application;

@ApplicationPath("/api")
public class RestApplication extends Application {
}
```

Therefore:

```java
@Path("/employees")
```

combined with:

```java
@ApplicationPath("/api")
```

produces:

```text
/api/employees
```

---

# 4. Resource Class

A resource represents an API resource.

```java
@Path("/employees")
public class EmployeeResource {

    @GET
    public List<EmployeeResponse> findAll() {
        return service.findAll();
    }
}
```

Think:

```text
EmployeeResource
       ↓
Employee REST API
```

---

# 5. Resource Method

A resource class can contain multiple methods:

```java
@Path("/employees")
public class EmployeeResource {

    @GET
    public List<EmployeeResponse> findAll() {
        return service.findAll();
    }

    @GET
    @Path("/{id}")
    public EmployeeResponse findById(
            @PathParam("id") Long id) {
        return service.findById(id);
    }

    @POST
    public Response create(
            EmployeeCreateRequest request) {
        ...
    }

    @DELETE
    @Path("/{id}")
    public Response delete(
            @PathParam("id") Long id) {
        ...
    }
}
```

---

# 6. URI Templates

Jakarta REST supports URI templates.

```java
@Path("/employees/{id}")
```

Request:

```text
/employees/101
```

gives:

```text
id = 101
```

---

# 7. Path Expressions

You can constrain path variables.

For example:

```java
@Path("/{id: \\d+}")
```

This indicates that `id` should consist of digits.

So:

```text
/employees/101
```

matches.

While:

```text
/employees/abc
```

doesn't match that template.

This becomes useful when multiple URI patterns overlap.

---

# 8. Multiple Path Parameters

Example:

```java
@Path("/departments/{departmentId}/employees/{employeeId}")
```

Request:

```text
/api/departments/10/employees/101
```

Code:

```java
@GET
@Path("/departments/{departmentId}/employees/{employeeId}")
public EmployeeResponse find(
        @PathParam("departmentId")
        Long departmentId,

        @PathParam("employeeId")
        Long employeeId) {

    return service.find(
        departmentId,
        employeeId
    );
}
```

---

# 9. Path vs Query Parameters

Use path parameters when identifying a resource:

```text
/api/employees/101
```

Use query parameters for filtering/searching:

```text
/api/employees?departmentId=10
```

Don't turn every parameter into a path segment.

---

# 10. Matrix Parameters

Jakarta REST also supports matrix parameters.

Example:

```text
/employees;department=engineering
```

Code:

```java
@GET
public Response find(
        @MatrixParam("department")
        String department) {

    ...
}
```

Matrix parameters are less common in modern APIs than query parameters, but they are part of the Jakarta REST programming model.

---

# 11. Form Parameters

For form submissions:

```java
@POST
@Consumes(MediaType.APPLICATION_FORM_URLENCODED)
public Response login(
        @FormParam("username")
        String username,

        @FormParam("password")
        String password) {

    ...
}
```

Request:

```text
username=rahul&password=secret
```

---

# 12. Header Parameters

```java
@GET
public Response getEmployees(
        @HeaderParam("Authorization")
        String authorization) {

    ...
}
```

Useful headers include:

```text
Authorization
Accept
Content-Type
If-None-Match
If-Match
X-Request-ID
```

---

# 13. Cookie Parameters

You can access cookies:

```java
@GET
public Response profile(
        @CookieParam("JSESSIONID")
        String sessionId) {

    ...
}
```

For authentication systems, avoid treating the raw session ID as ordinary application data. Handle authentication through the security architecture.

---

# 14. `@Context`

Jakarta REST can inject contextual objects.

Examples include:

```java
@Context
HttpHeaders headers;
```

and:

```java
@Context
UriInfo uriInfo;
```

Also commonly used:

```java
SecurityContext
Request
ResourceInfo
```

---

# 15. `HttpHeaders`

Example:

```java
@Context
HttpHeaders headers;

@GET
public Response get() {

    List<String> values =
        headers.getRequestHeader(
            "X-Request-ID"
        );

    ...
}
```

Useful when you need access to HTTP metadata.

---

# 16. `UriInfo`

`UriInfo` provides URI information.

```java
@Context
UriInfo uriInfo;
```

You can use it to inspect:

```text
absolute URI
base URI
path
query parameters
path parameters
```

---

# 17. Building URIs

Suppose employee `101` was created.

You can construct:

```text
/api/employees/101
```

using URI-building APIs rather than concatenating strings manually.

Conceptually:

```java
URI uri = uriInfo
        .getAbsolutePathBuilder()
        .path(String.valueOf(employee.getId()))
        .build();
```

Then:

```java
return Response
        .created(uri)
        .entity(employee)
        .build();
```

This is cleaner for APIs that need a `Location` header.

---

# 18. `Response`

The `Response` class gives explicit HTTP control.

Example:

```java
return Response.ok(employee).build();
```

Or:

```java
return Response
        .status(Response.Status.CREATED)
        .entity(employee)
        .build();
```

Or:

```java
return Response.noContent().build();
```

---

# 19. `Response.Status`

Instead of magic numbers:

```java
return Response
        .status(404)
        .build();
```

prefer:

```java
return Response
        .status(Response.Status.NOT_FOUND)
        .build();
```

This communicates intent better.

---

# 20. Response Headers

You can create custom headers:

```java
return Response
        .ok(employee)
        .header("X-Request-ID", requestId)
        .build();
```

Common response headers include:

```text
Content-Type
Location
Cache-Control
ETag
Last-Modified
Allow
Retry-After
```

---

# 21. Entity Providers

One of the most important concepts today.

Suppose your method returns:

```java
EmployeeResponse
```

but HTTP needs:

```text
JSON
```

Who converts Java into JSON?

The REST runtime uses **MessageBodyWriter** implementations.

Conceptually:

```text
EmployeeResponse
       ↓
MessageBodyWriter
       ↓
JSON
       ↓
HTTP response
```

For incoming requests:

```text
JSON
 ↓
MessageBodyReader
 ↓
EmployeeCreateRequest
```

---

# 22. MessageBodyReader

Conceptual flow:

```text
HTTP Request
Content-Type: application/json

{
   ...
}
```

then:

```text
JSON
 ↓
MessageBodyReader
 ↓
Java DTO
```

This is why you don't normally manually parse JSON in every REST method.

---

# 23. MessageBodyWriter

Response:

```java
return employee;
```

can become:

```text
Employee object
 ↓
MessageBodyWriter
 ↓
JSON
 ↓
HTTP response
```

This abstraction is extremely important when you start building custom providers.

---

# 24. Providers

Jakarta REST providers extend the runtime.

Common provider types include:

```text
MessageBodyReader
MessageBodyWriter
ExceptionMapper
ContextResolver
ParamConverterProvider
```

Think:

```text
Resource
   +
Providers
   =
REST processing pipeline
```

---

# 25. `@Provider`

A provider can be registered using:

```java
@Provider
public class MyExceptionMapper
        implements ExceptionMapper<MyException> {
}
```

Providers allow cross-cutting API behavior to be centralized.

---

# 26. ExceptionMapper

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
            .status(Response.Status.NOT_FOUND)
            .entity(new ErrorResponse(
                "EMPLOYEE_NOT_FOUND",
                "Employee was not found"
            ))
            .build();
    }
}
```

Now your resource can simply:

```java
throw new EmployeeNotFoundException();
```

---

# 27. Why ExceptionMapper Matters

Without centralized handling:

```text
Resource 1 → try/catch
Resource 2 → try/catch
Resource 3 → try/catch
Resource 4 → try/catch
```

With `ExceptionMapper`:

```text
Resource
   ↓
Exception
   ↓
ExceptionMapper
   ↓
Standard Error Response
```

This is much easier to maintain.

---

# 28. REST Filters

Filters operate around request/response processing.

Two major categories:

```text
ContainerRequestFilter
ContainerResponseFilter
```

---

# 29. Request Filter

Example:

```java
@Provider
public class RequestLoggingFilter
        implements ContainerRequestFilter {

    @Override
    public void filter(
            ContainerRequestContext context) {

        System.out.println(
            context.getMethod()
            + " "
            + context.getUriInfo()
                    .getRequestUri()
        );
    }
}
```

Flow:

```text
Request
  ↓
Request Filter
  ↓
Resource
```

---

# 30. Response Filter

```java
@Provider
public class ResponseHeaderFilter
        implements ContainerResponseFilter {

    @Override
    public void filter(
            ContainerRequestContext request,
            ContainerResponseContext response) {

        response.getHeaders().add(
            "X-API-Version",
            "v1"
        );
    }
}
```

Flow:

```text
Resource
  ↓
Response Filter
  ↓
HTTP Response
```

---

# 31. Filters vs Interceptors

This distinction is frequently asked in interviews.

### Filters

Primarily work with:

```text
HTTP request/response
```

Examples:

```text
authentication metadata
logging
headers
CORS
request IDs
```

### Interceptors

Primarily operate around:

```text
entity body processing
```

Examples:

```text
MessageBodyReader
MessageBodyWriter
```

interception.

Mental model:

```text
HTTP-level concern → Filter

Entity serialization/deserialization concern
                  → Interceptor
```

---

# 32. Content Negotiation

Request:

```text
Accept: application/json
```

means:

> I prefer JSON.

Request:

```text
Content-Type: application/json
```

means:

> The request body is JSON.

Remember:

```text
Accept
   ↓
What response representation I want

Content-Type
   ↓
What representation I'm sending
```

This distinction is fundamental.

---

# 33. Media Types

Common:

```java
MediaType.APPLICATION_JSON
MediaType.TEXT_PLAIN
MediaType.APPLICATION_XML
MediaType.APPLICATION_FORM_URLENCODED
```

You can specify:

```java
@Produces(MediaType.APPLICATION_JSON)
```

and:

```java
@Consumes(MediaType.APPLICATION_JSON)
```

---

# 34. Multiple Representations

An endpoint could theoretically support:

```text
application/json
application/xml
```

For example:

```java
@Produces({
    MediaType.APPLICATION_JSON,
    MediaType.APPLICATION_XML
})
```

The REST runtime can use content negotiation to select an appropriate representation based on the request.

---

# 35. REST Sub-Resources

Suppose:

```text
/api/departments/10/employees
```

You could model a sub-resource.

Example:

```java
@Path("/departments/{departmentId}")
public class DepartmentResource {

    @Path("/employees")
    public EmployeeSubResource employees() {
        return new EmployeeSubResource();
    }
}
```

Then:

```text
/api/departments/10/employees
```

can be delegated to the sub-resource.

---

# 36. Sub-Resource Locator

A method without an HTTP method annotation can act as a locator.

Example:

```java
@Path("/employees")
public class EmployeeResource {

    @Path("/{id}")
    public EmployeeSubResource employee(
            @PathParam("id") Long id) {

        return new EmployeeSubResource(id);
    }
}
```

Then:

```java
public class EmployeeSubResource {

    private final Long id;

    public EmployeeSubResource(Long id) {
        this.id = id;
    }

    @GET
    public EmployeeResponse get() {
        ...
    }

    @PUT
    public Response update(...) {
        ...
    }
}
```

This becomes useful for complex resource hierarchies.

---

# 37. Don't Over-Nest Resources

Technically you can create:

```text
/api/company/1/department/10/team/20/employee/101/address/5
```

but this can become difficult to understand and maintain.

Prefer resource URLs that clearly represent the domain.

For example:

```text
/api/employees/101
/api/departments/10
/api/employees/101/address
```

when appropriate.

---

# 38. REST Client API

Jakarta REST isn't only for building APIs.

It also provides a client API.

Conceptually:

```text
Java Application
       ↓
Jakarta REST Client
       ↓
HTTP
       ↓
External REST API
```

---

# 39. Basic REST Client

Example:

```java
Client client = ClientBuilder.newClient();

Response response = client
        .target("https://example.com/api/employees/101")
        .request(MediaType.APPLICATION_JSON)
        .get();

System.out.println(response.getStatus());

response.close();
client.close();
```

---

# 40. Client Target

Instead of repeating the URL:

```java
WebTarget target =
        client.target(
            "https://example.com/api/employees"
        );
```

Then:

```java
Response response =
        target
            .path("101")
            .request(MediaType.APPLICATION_JSON)
            .get();
```

---

# 41. Reading the Response

You can retrieve an entity:

```java
EmployeeResponse employee =
        response.readEntity(
            EmployeeResponse.class
        );
```

Always manage the response/client lifecycle appropriately.

For example:

```java
try (Response response =
         target.request().get()) {

    EmployeeResponse employee =
        response.readEntity(
            EmployeeResponse.class
        );
}
```

The exact resource-management approach should match your Jakarta REST client/runtime version and application architecture.

---

# 42. POST Using REST Client

Conceptually:

```java
Response response =
    target
        .request(MediaType.APPLICATION_JSON)
        .post(
            Entity.json(request)
        );
```

Flow:

```text
Java DTO
 ↓
REST Client
 ↓
JSON
 ↓
HTTP POST
 ↓
External API
```

---

# 43. REST Client in Enterprise Architecture

Imagine your Employee system calls:

```text
Employee Service
       ↓
REST Client
       ↓
Payroll Service
```

Architecture:

```text
                    Employee API
                         │
                    EmployeeService
                         │
                    REST Client
                         │
                         ▼
                  Payroll API
                         │
                   PayrollService
                         │
                      Database
```

Now you are entering **distributed systems**.

---

# 44. Timeouts

Never blindly call an external service without timeout policies.

Suppose:

```text
Employee API
     ↓
Payroll API
     ↓
Payroll API hangs
```

Your threads could become blocked.

You need appropriate:

```text
connection timeout
read timeout
request timeout
```

and failure handling appropriate to your runtime.

---

# 45. External API Failure

Never assume:

```text
External API = always available
```

Consider:

```text
timeout
connection failure
5xx
429
invalid response
partial response
slow response
DNS failure
```

Architecture should define what your service does for each class of failure.

---

# 46. Retry Carefully

A retry may be useful for transient failures.

But:

```text
retry × retry × retry
```

can amplify an outage.

Use controlled:

```text
limited retries
backoff
jitter
timeouts
idempotency
```

For non-idempotent operations, blindly retrying can create duplicate side effects.

This connects directly to the distributed-systems concepts you'll study later.

---

# 47. REST and Caching

HTTP supports caching semantics.

Useful headers:

```text
Cache-Control
ETag
If-None-Match
Last-Modified
If-Modified-Since
```

Example:

```text
GET /api/employees/101
```

Response:

```text
ETag: "abc123"
```

Client later sends:

```text
If-None-Match: "abc123"
```

If unchanged:

```text
304 Not Modified
```

This can reduce payload transfer.

---

# 48. ETags

Conceptually:

```text
Resource
   ↓
Representation version
   ↓
ETag
```

The ETag can also participate in concurrency control.

For example:

```text
GET employee
ETag = "v5"
```

Client updates:

```text
If-Match: "v5"
```

If the server's current version is already `"v6"`:

```text
412 Precondition Failed
```

This is one HTTP-level approach to preventing lost updates.

---

# 49. Conditional Requests

Important headers:

```text
If-Match
If-None-Match
If-Modified-Since
If-Unmodified-Since
```

These become useful for:

```text
caching
optimistic concurrency
avoiding unnecessary updates
```

---

# 50. REST and Transactions

Remember:

```text
HTTP request ≠ automatically a business transaction
```

Example:

```text
POST /orders
```

could execute:

```text
OrderService.createOrder()
```

which performs:

```text
BEGIN
 ↓
Create Order
 ↓
Create Order Items
 ↓
Update Inventory
 ↓
COMMIT
```

The service/application layer should define the business transaction boundary.

---

# 51. REST and Async Processing

Some operations take too long to complete synchronously.

Example:

```text
POST /reports
```

Instead of waiting 5 minutes:

```text
POST /reports
        ↓
202 Accepted
        ↓
job created
```

Client can later query:

```text
GET /reports/jobs/123
```

Possible status:

```json
{
  "jobId": "123",
  "status": "PROCESSING"
}
```

---

# 52. 202 Accepted

Use:

```text
202 Accepted
```

when the server has accepted the request for processing but the final result isn't available yet.

Don't use `202` simply because an endpoint is asynchronous internally; it should match the actual API contract.

---

# 53. REST API Security Architecture

A production API typically needs:

```text
Client
 ↓
TLS
 ↓
Gateway / Load Balancer
 ↓
Authentication
 ↓
Authorization
 ↓
Rate Limiting
 ↓
REST Resource
 ↓
Service
```

Security mechanisms can include:

```text
OAuth 2.0
OpenID Connect
JWT bearer tokens
mTLS
API keys
session authentication
```

You will go deeper into REST security on Day 17.

---

# 54. Request Correlation

Create/request a correlation ID:

```text
X-Request-ID: 7f1d...
```

Filter:

```text
HTTP Request
      ↓
Correlation Filter
      ↓
REST Resource
      ↓
Service
```

Logs:

```text
requestId=7f1d...
method=POST
path=/api/employees
status=201
duration=48ms
```

This is essential for troubleshooting distributed systems.

---

# 55. Production Logging

A useful REST log might contain:

```text
timestamp
requestId
method
URI
status
duration
authenticated principal
service instance
```

Avoid logging:

```text
passwords
access tokens
session secrets
credit-card information
sensitive personal data
```

unless there is a carefully controlled and justified requirement.

---

# 56. API Error Contract

Standardize errors.

Example:

```json
{
  "timestamp": "2026-10-18T10:20:30Z",
  "status": 400,
  "code": "VALIDATION_ERROR",
  "message": "Invalid employee request",
  "path": "/api/employees",
  "requestId": "abc123",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email"
    }
  ]
}
```

Every API should ideally have predictable error semantics.

---

# 57. REST API Design Principles

Remember these:

```text
Resources
   ↓
HTTP semantics
   ↓
Representations
   ↓
Stateless requests
   ↓
Consistent contracts
   ↓
Correct status codes
   ↓
Validation
   ↓
Security
   ↓
Observability
```

---

# 58. Statelessness

A REST interaction should contain the information needed to process that request.

Don't depend on arbitrary server-side conversational state for the API's meaning.

Instead:

```text
Request
 ↓
Authentication/context
 ↓
Resource operation
```

This makes horizontal scaling easier.

---

# 59. Horizontal Scaling

If your API is stateless:

```text
              Load Balancer
              /     |     \
             /      |      \
           API-1   API-2   API-3
             \       |      /
              \      |     /
                   DB
```

Any instance can process the request.

If sessions are involved, you'll need an explicit strategy for session state, such as shared storage or appropriately configured affinity, depending on the architecture.

---

# 60. REST API Versioning

Possible:

```text
/api/v1/employees
/api/v2/employees
```

But versioning isn't merely about changing the URL.

You need a strategy for:

```text
breaking changes
deprecation
migration
client compatibility
documentation
support lifecycle
```

---

# 61. API Contract First

Before coding, define:

```text
Resource
URL
Method
Request
Response
Status codes
Validation
Authentication
Authorization
Errors
Pagination
Filtering
Sorting
```

Example:

```text
POST /api/employees

Request:
EmployeeCreateRequest

Success:
201 Created

Validation failure:
400 Bad Request

Duplicate email:
409 Conflict
```

This is how architects think about APIs.

---

# 62. Day 16 Production Architecture

Your Employee Management System should now look like:

```text
                         Clients
                            │
                            ▼
                         HTTPS
                            │
                            ▼
                    Load Balancer
                            │
                            ▼
                     Jakarta REST
                            │
           ┌────────────────┼────────────────┐
           │                │                │
           ▼                ▼                ▼
       Filters          Providers       Interceptors
           │                │                │
           └────────────────┼────────────────┘
                            ▼
                    EmployeeResource
                            │
                            ▼
                     DTO Validation
                            │
                            ▼
                     EmployeeService
                            │
                      Transaction
                            │
                            ▼
                      EmployeeDAO
                            │
                            ▼
                         JPA/JDBC
                            │
                            ▼
                        Database
```

---

# 63. Day 16 Coding Project

## Employee REST API v2

Upgrade yesterday's API with:

### Resources

```text
EmployeeResource
DepartmentResource
AuthResource
```

### DTOs

```text
EmployeeCreateRequest
EmployeeUpdateRequest
EmployeeResponse
ErrorResponse
```

### Providers

```text
EmployeeNotFoundMapper
ValidationExceptionMapper
GenericExceptionMapper
```

### Filters

```text
CorrelationIdFilter
RequestLoggingFilter
SecurityFilter
```

### Features

```text
pagination
sorting
filtering
validation
error handling
ETag
conditional requests
```

---

# 64. Suggested Package Structure

```text
com.example.ems
│
├── config
│   └── RestApplication.java
│
├── resource
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
├── service
│   ├── EmployeeService.java
│   └── DepartmentService.java
│
├── dao
│   ├── EmployeeDAO.java
│   └── DepartmentDAO.java
│
├── entity
│   ├── Employee.java
│   └── Department.java
│
├── mapper
│   └── EmployeeMapper.java
│
├── exception
│   ├── EmployeeNotFoundException.java
│   └── DuplicateEmailException.java
│
├── provider
│   ├── EmployeeNotFoundMapper.java
│   └── GenericExceptionMapper.java
│
└── filter
    ├── CorrelationIdFilter.java
    └── RequestLoggingFilter.java
```

---

# 65. 20 Hands-On Exercises

### Beginner

1. Create `/api/hello`.
2. Create `/api/employees`.
3. Add `@PathParam`.
4. Add `@QueryParam`.
5. Add `@HeaderParam`.

### Intermediate

6. Add POST JSON.
7. Add PUT.
8. Add DELETE.
9. Add `@Consumes`.
10. Add `@Produces`.
11. Build `Response` objects.
12. Add `Location` header.
13. Add validation.
14. Create an `ExceptionMapper`.
15. Create a request filter.

### Advanced

16. Create a response filter.
17. Implement a custom `MessageBodyWriter`.
18. Implement a REST client.
19. Implement ETag/conditional update semantics.
20. Implement an asynchronous report API using `202 Accepted`.

---

# 66. Interview Questions

### Fundamentals

**1. What is JAX-RS?**

The former name for the Java API used to create RESTful web services; modern Jakarta EE calls it Jakarta RESTful Web Services.

**2. What is `@Path`?**

It maps URI paths to resources or resource methods.

**3. What is `@Produces`?**

It declares response media types.

**4. What is `@Consumes`?**

It declares request media types.

**5. What is `@PathParam`?**

Extracts values from URI path parameters.

**6. What is `@QueryParam`?**

Extracts values from the URI query string.

**7. What is `@HeaderParam`?**

Injects a request-header value.

---

# 67. Advanced Interview Questions

**8. What is MessageBodyReader?**

It converts an HTTP request entity into a Java representation.

**9. What is MessageBodyWriter?**

It converts a Java representation into an HTTP response entity.

**10. What is an ExceptionMapper?**

It converts application exceptions into HTTP responses.

**11. Filter vs interceptor?**

```text
Filter
→ request/response processing

Interceptor
→ entity body processing
```

**12. What is content negotiation?**

The mechanism through which the client and server determine a suitable representation using media types such as JSON or XML.

**13. What is a sub-resource locator?**

A resource method that returns another resource/sub-resource rather than directly handling an HTTP method.

---

# 68. Architect-Level Questions

### 14. Why shouldn't your REST resource contain business logic?

Because it couples business behavior to HTTP and makes reuse, testing, and evolution harder.

Better:

```text
REST
 ↓
Service
 ↓
DAO
```

---

### 15. How would you make a REST API horizontally scalable?

Keep request processing stateless where practical and externalize shared state into appropriate systems.

```text
Load Balancer
   ↓
API-1
API-2
API-3
   ↓
Shared DB/cache/message infrastructure
```

---

### 16. How would you handle an external REST service becoming slow?

Use:

```text
timeouts
bounded retries
backoff
jitter
circuit breaking
bulkheads
fallbacks where appropriate
observability
```

The exact strategy depends on whether the operation is safe/idempotent and on business requirements.

---

### 17. How do you prevent duplicate POST operations?

Possible mechanisms:

```text
Idempotency-Key
+
database uniqueness constraints
+
transactional processing
```

---

### 18. How do you prevent lost updates?

Possible approaches include:

```text
@Version optimistic locking
ETag + If-Match
pessimistic locking
```

depending on the concurrency model.

---

### 19. How do you design a large REST API?

Start with:

```text
Domain resources
      ↓
API contracts
      ↓
HTTP semantics
      ↓
DTOs
      ↓
Validation
      ↓
Security
      ↓
Persistence
      ↓
Observability
      ↓
Scalability
```

---

# 69. Day 16 Mastery Challenge

Design this API without coding first:

## Banking API

Resources:

```text
customers
accounts
transactions
beneficiaries
```

Design:

```text
POST   /api/v1/customers
GET    /api/v1/customers/{id}

POST   /api/v1/accounts
GET    /api/v1/accounts/{id}

GET    /api/v1/accounts/{id}/transactions

POST   /api/v1/transfers

POST   /api/v1/beneficiaries
GET    /api/v1/beneficiaries
DELETE /api/v1/beneficiaries/{id}
```

For every endpoint specify:

```text
HTTP method
URL
request DTO
response DTO
status codes
validation
authentication
authorization
transaction boundary
idempotency
concurrency strategy
error response
logging
rate limit
```

If you can do this correctly, you're beginning to think like an **API/system designer**, not merely an endpoint programmer.

---

# 70. Day 16 Final Cheat Sheet

```text
@Path
→ Resource URI

@GET
→ Retrieve

@POST
→ Create/process

@PUT
→ Replace/update

@DELETE
→ Delete

@PathParam
→ Path variable

@QueryParam
→ Query parameter

@HeaderParam
→ Header

@CookieParam
→ Cookie

@FormParam
→ Form field

@Consumes
→ Request representation

@Produces
→ Response representation

Response
→ Explicit HTTP response

@Provider
→ REST provider

ExceptionMapper
→ Exception → HTTP response

ContainerRequestFilter
→ Request filter

ContainerResponseFilter
→ Response filter

MessageBodyReader
→ HTTP body → Java

MessageBodyWriter
→ Java → HTTP body

UriInfo
→ URI information

HttpHeaders
→ HTTP headers

Client
→ Call external REST APIs
```

## The most important mental model

```text
                    HTTP Request
                         │
                         ▼
                Jakarta REST Runtime
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
      Request Filters             URI Matching
                                      │
                                      ▼
                              Resource Method
                                      │
                              Parameter Binding
                                      │
                                      ▼
                                  Service
                                      │
                                  Database
                                      │
                                      ▼
                                  Response
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                  Response Filter          MessageBodyWriter
                                                   │
                                                   ▼
                                                 JSON
```

### Day 16 golden rule

**A REST API is not just a collection of URLs. It is a contract between clients and your system involving resources, HTTP semantics, representations, validation, errors, security, concurrency, reliability, and observability.**

**Day 17:** REST API Security — authentication vs authorization, Basic Auth, session authentication, API keys, OAuth 2.0, JWT, OpenID Connect, roles/permissions, security filters, CORS, CSRF, TLS, token validation, refresh tokens, service-to-service authentication, and production API security architecture.
