# Day 24 — Validation & Exception Handling

**J2EE/Jakarta EE 30-Day Mastery**
**Date:** October 26, 2026
**Goal:** Build enterprise applications that **reject invalid data early, handle failures predictably, return safe API errors, and remain observable and maintainable in production**.

Today connects:

```text
HTTP Request
     ↓
Validation
     ↓
Controller / REST Resource
     ↓
Service
     ↓
Transaction
     ↓
Database
     ↓
Exception Handling
     ↓
Safe Error Response
```

---

# 1. Why Validation Matters

Never assume incoming data is valid.

A client might send:

```json
{
  "name": "",
  "email": "abc",
  "age": -50
}
```

Without validation, bad data can reach:

```text
REST
 ↓
Service
 ↓
Database
```

and cause:

* invalid records
* database errors
* security vulnerabilities
* inconsistent business state
* confusing user experience

Therefore:

> **Validate data at appropriate boundaries, but don't rely only on client-side validation.**

---

# 2. Types of Validation

Think in layers.

### 1. Syntactic validation

Is the format valid?

```text
email = valid format?
age = integer?
date = valid format?
```

### 2. Structural validation

Are required fields present?

```text
name != null
email != null
```

### 3. Business validation

Does the operation make business sense?

```text
age >= 18
balance >= withdrawal
```

### 4. Security validation

Is the input safe?

```text
authorization
input constraints
payload size
file type
```

---

# 3. Jakarta Bean Validation

Jakarta Validation provides standard validation annotations and APIs.

Modern applications use:

```java
jakarta.validation.*
```

Typical annotations include:

```java
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@Positive
@PositiveOrZero
@Email
@Pattern
@Past
@Future
```

---

# 4. `@NotNull`

```java
public class EmployeeRequest {

    @NotNull
    private Long departmentId;
}
```

It means:

```text
departmentId cannot be null
```

But:

```text
""
```

is not `null`.

That's an important distinction.

---

# 5. `@NotEmpty`

```java
@NotEmpty
private List<String> skills;
```

Requires the collection/string not to be empty.

Conceptually:

```text
null → invalid
""   → invalid
[]   → invalid
```

---

# 6. `@NotBlank`

Primarily for strings.

```java
@NotBlank
private String name;
```

Rejects:

```text
null
""
"   "
```

For user-entered text, `@NotBlank` is often more appropriate than `@NotNull`.

---

# 7. `@Size`

```java
@Size(min = 2, max = 100)
private String name;
```

Useful for:

* names
* descriptions
* usernames
* lists
* collections

Example:

```java
@Size(min = 1, max = 10)
private List<String> skills;
```

---

# 8. Numeric Validation

Example:

```java
@Min(18)
private int age;
```

Other useful constraints:

```java
@Max(100)

@Positive
@PositiveOrZero

@Negative
@NegativeOrZero
```

Example:

```java
@Positive
private BigDecimal salary;
```

---

# 9. `@Email`

```java
@Email
private String email;
```

Remember:

> Format validation is not proof that the email address actually exists.

You may still need:

```text
email verification
```

for workflows that require it.

---

# 10. `@Pattern`

Use regular expressions for constrained formats.

```java
@Pattern(
    regexp = "^[A-Z]{2}[0-9]{6}$"
)
private String employeeCode;
```

Don't turn complicated business logic into giant regexes.

Prefer a custom validator or service-level validation when the rule is complex.

---

# 11. Date Validation

Examples:

```java
@Past
private LocalDate dateOfBirth;
```

and:

```java
@Future
private LocalDate joiningDate;
```

Other constraints:

```java
@PastOrPresent
@FutureOrPresent
```

---

# 12. Example DTO

Create a request DTO:

```java
public class CreateEmployeeRequest {

    @NotBlank
    @Size(max = 100)
    private String name;

    @NotBlank
    @Email
    private String email;

    @Positive
    private BigDecimal salary;

    @NotNull
    private Long departmentId;

    // getters/setters
}
```

This is preferable to exposing your persistence entity directly as the API input model.

---

# 13. DTO vs Entity

Avoid:

```text
HTTP Request
    ↓
JPA Entity
    ↓
Database
```

Prefer:

```text
HTTP Request
    ↓
Request DTO
    ↓
Validation
    ↓
Service
    ↓
Entity
    ↓
Database
```

Why?

Because API requirements and database structure evolve independently.

---

# 14. Nested Validation

Suppose:

```java
public class CreateOrderRequest {

    @NotNull
    private CustomerRequest customer;

    @Valid
    private List<OrderItemRequest> items;
}
```

`@Valid` tells the validation engine to validate nested objects.

Example:

```java
public class OrderItemRequest {

    @NotBlank
    private String productId;

    @Positive
    private int quantity;
}
```

Now invalid items are detected.

---

# 15. `@Valid` vs `@Validated`

You will encounter both.

### `@Valid`

Standard Jakarta Validation cascading mechanism.

### `@Validated`

Often used by frameworks for validation groups and method-level validation.

The exact behavior depends on the framework/integration layer you're using.

For Jakarta EE, understand the standard validation API first.

---

# 16. Validation Groups

Sometimes the same DTO requires different rules.

Example:

```text
Create Employee
Update Employee
```

Create:

```text
email required
password required
```

Update:

```text
email optional
password optional
```

Validation groups allow different constraint sets.

Conceptually:

```java
public interface CreateGroup {}
public interface UpdateGroup {}
```

Then:

```java
@NotBlank(groups = CreateGroup.class)
private String password;
```

---

# 17. Cross-Field Validation

Some rules involve multiple fields.

Example:

```text
startDate < endDate
```

No single-field annotation can fully express this rule.

You can create a class-level constraint.

Conceptually:

```java
@ValidDateRange
public class BookingRequest {

    private LocalDate startDate;
    private LocalDate endDate;
}
```

---

# 18. Custom Validator

Define:

```java
@Constraint(validatedBy = EmployeeCodeValidator.class)
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidEmployeeCode {

    String message() default "Invalid employee code";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

Validator:

```java
public class EmployeeCodeValidator
        implements ConstraintValidator<ValidEmployeeCode, String> {

    @Override
    public boolean isValid(
            String value,
            ConstraintValidatorContext context) {

        if (value == null) {
            return true;
        }

        return value.matches("EMP-[0-9]{6}");
    }
}
```

Usage:

```java
@ValidEmployeeCode
private String employeeCode;
```

---

# 19. Important Validation Principle

Not every rule belongs in Bean Validation.

Good Bean Validation:

```text
email format
required field
minimum length
positive amount
date format
```

Business validation:

```text
Can this employee approve this transaction?
```

or:

```text
Can this customer place another order?
```

belongs in the service/domain layer.

Think:

```text
Validation
   ↓
"Is this input structurally valid?"

Business Rule
   ↓
"Is this operation allowed?"
```

---

# 20. Validation in Jakarta REST

A REST endpoint might receive:

```java
@POST
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public Response createEmployee(
        @Valid CreateEmployeeRequest request) {

    Employee employee =
        employeeService.create(request);

    return Response
        .status(Response.Status.CREATED)
        .entity(employee)
        .build();
}
```

The validation layer can reject invalid input before business processing proceeds, depending on the Jakarta REST implementation/integration.

---

# 21. Validation Error

Suppose:

```json
{
  "name": "",
  "email": "wrong"
}
```

A production API shouldn't return:

```text
500 Internal Server Error
```

This is a client input problem.

Typically:

```text
400 Bad Request
```

is appropriate.

---

# 22. Good Error Response

Create a consistent error contract.

Example:

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields are invalid",
  "instance": "/api/employees",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email"
    },
    {
      "field": "name",
      "message": "must not be blank"
    }
  ]
}
```

A standardized error representation makes client development easier.

---

# 23. What Is an Exception?

An exception represents an abnormal condition that interrupts normal program flow.

Example:

```java
throw new EmployeeNotFoundException(id);
```

instead of:

```java
return null;
```

when absence is an exceptional condition for that operation.

---

# 24. Exception Hierarchy

Basic Java hierarchy:

```text
Throwable
├── Error
└── Exception
    ├── RuntimeException
    └── checked exceptions
```

Examples:

```text
RuntimeException
 ├── IllegalArgumentException
 ├── NullPointerException
 └── IllegalStateException
```

Checked exceptions include many I/O/database-related exceptions.

---

# 25. Checked vs Unchecked Exceptions

### Checked

Compiler forces handling/declaration.

Example:

```java
IOException
SQLException
```

### Unchecked

Subclass of `RuntimeException`.

Example:

```java
IllegalArgumentException
IllegalStateException
```

For modern enterprise applications, don't mechanically make every failure a checked exception. Choose exception types based on whether callers can meaningfully recover.

---

# 26. Custom Exceptions

Create domain-specific exceptions.

```java
public class EmployeeNotFoundException
        extends RuntimeException {

    public EmployeeNotFoundException(Long id) {
        super("Employee not found: " + id);
    }
}
```

Another:

```java
public class DuplicateEmployeeException
        extends RuntimeException {

    public DuplicateEmployeeException(String email) {
        super("Employee already exists: " + email);
    }
}
```

---

# 27. Why Domain Exceptions?

Compare:

```java
throw new RuntimeException("something went wrong");
```

with:

```java
throw new EmployeeNotFoundException(id);
```

The second provides semantic information.

Now the API layer can map:

```text
EmployeeNotFoundException
        ↓
404
```

---

# 28. Exception Layering

A clean architecture looks like:

```text
REST Resource
      ↓
Service
      ↓
Repository
      ↓
Database
```

Exceptions may originate at different layers.

Example:

```text
Database
   ↓
SQLException
   ↓
Repository
   ↓
DataAccessException
   ↓
Service
   ↓
EmployeeNotFoundException
   ↓
REST
   ↓
404
```

Don't expose low-level database details to API clients.

---

# 29. Exception Translation

Suppose JDBC produces:

```java
SQLException
```

Don't return:

```json
{
  "error": "SQLException: password authentication failed..."
}
```

Instead:

```text
SQLException
    ↓
Repository translates/logs appropriately
    ↓
Application exception
    ↓
REST error mapper
    ↓
Safe response
```

---

# 30. Jakarta REST ExceptionMapper

Jakarta REST provides:

```java
ExceptionMapper<E>
```

Example:

```java
@Provider
public class EmployeeNotFoundMapper
        implements ExceptionMapper<EmployeeNotFoundException> {

    @Override
    public Response toResponse(
            EmployeeNotFoundException exception) {

        ErrorResponse error =
            new ErrorResponse(
                404,
                exception.getMessage()
            );

        return Response
            .status(Response.Status.NOT_FOUND)
            .entity(error)
            .build();
    }
}
```

Now:

```text
EmployeeNotFoundException
             ↓
ExceptionMapper
             ↓
HTTP 404
```

---

# 31. Global Exception Handling

A production API should have centralized exception mapping.

Conceptually:

```text
                 REST API
                    │
                    ▼
             Exception Layer
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
 Validation      NotFound     Conflict
       │            │            │
      400          404          409
```

And unexpected failures:

```text
Unexpected Exception
       ↓
500 Internal Server Error
```

---

# 32. HTTP Status Mapping

A useful baseline:

| Situation                      |                     Typical Status |
| ------------------------------ | ---------------------------------: |
| Successful GET                 |                                200 |
| Successful POST                |                                201 |
| Successful DELETE              |                                204 |
| Invalid request                |                                400 |
| Unauthenticated                |                                401 |
| Authenticated but unauthorized |                                403 |
| Resource not found             |                                404 |
| Conflict                       |                                409 |
| Unsupported media type         |                                415 |
| Rate limited                   |                                429 |
| Unexpected server failure      |                                500 |
| Temporary upstream failure     | 502/503/504 depending on situation |

Status selection should reflect the actual semantics of the failure.

---

# 33. 401 vs 403

Very common interview question.

### 401 Unauthorized

Authentication is missing/invalid.

```text
"Who are you?"
```

### 403 Forbidden

The request is authenticated, but access is not permitted.

```text
"I know who you are,
but you cannot perform this operation."
```

---

# 34. 404 vs 400

### 400

The request itself is invalid.

```text
GET /employees/abc
```

when an integer ID is required.

### 404

The requested resource cannot be found.

```text
GET /employees/999999
```

when employee 999999 doesn't exist.

---

# 35. 409 Conflict

Useful when the request conflicts with current resource state.

Example:

```text
POST /employees
email = john@example.com
```

but the email must be unique and already exists.

Response:

```text
409 Conflict
```

---

# 36. Don't Leak Internal Details

Never return:

```json
{
  "error": "org.postgresql.util.PSQLException",
  "sql": "SELECT password FROM users...",
  "stackTrace": "..."
}
```

This can expose:

* database technology
* SQL
* table names
* filesystem paths
* internal class names
* stack traces
* secrets

Instead:

```json
{
  "status": 500,
  "title": "Internal Server Error",
  "detail": "An unexpected error occurred.",
  "traceId": "abc-123"
}
```

Log the detailed information internally.

---

# 37. Logging Exceptions

Bad:

```java
catch (Exception e) {
    System.out.println(e);
}
```

Better enterprise practice:

```java
logger.error(
    "Failed to create employee. correlationId={}",
    correlationId,
    e
);
```

Log:

```text
timestamp
service
operation
correlationId
traceId
exception type
safe business identifiers
```

Avoid logging sensitive information.

---

# 38. Exception Handling Anti-Pattern

Avoid:

```java
try {
    process();
} catch (Exception e) {
    return null;
}
```

This hides failures.

Also avoid:

```java
catch (Exception e) {
    e.printStackTrace();
}
```

without proper logging/handling.

---

# 39. Don't Catch Everything

Bad:

```java
try {
    processOrder();
} catch (Exception e) {
    // ignore
}
```

Why?

Because:

```text
failure → hidden
```

and the application may continue in an invalid state.

Catch an exception when you can:

* recover
* translate it
* add meaningful context
* perform required cleanup
* map it to an appropriate response

Otherwise let the appropriate higher-level handler process it.

---

# 40. Try-With-Resources

For resources:

```java
try (Connection connection =
         dataSource.getConnection()) {

    // database operation

}
```

Resources are automatically closed.

This reduces resource leaks.

---

# 41. Exception Handling + Transactions

This is extremely important.

Suppose:

```text
BEGIN
 ↓
Create Order
 ↓
Update Inventory
 ↓
Exception
```

Depending on the transaction configuration and exception type:

```text
Exception
   ↓
Rollback?
```

You need to understand what causes the transaction to roll back in your transaction technology/container rather than assuming every exception automatically rolls everything back.

For Jakarta Transactions, rollback behavior can be explicitly controlled, including via transaction APIs and `@Transactional` semantics.

---

# 42. Business Exception vs Technical Exception

### Business exception

Expected domain condition.

```text
InsufficientFundsException
EmployeeNotFoundException
DuplicateOrderException
```

### Technical exception

Infrastructure failure.

```text
DatabaseUnavailable
Timeout
ConnectionFailure
SerializationFailure
```

These should not necessarily be presented to users in the same way.

---

# 43. Error Handling Architecture

Use:

```text
Client
  ↓
REST
  ↓
Validation
  ↓
Service
  ↓
Repository
  ↓
Exception
  ↓
Exception Mapper
  ↓
Error DTO
  ↓
HTTP Response
```

Example:

```text
Employee ID doesn't exist
        ↓
EmployeeNotFoundException
        ↓
ExceptionMapper
        ↓
404
        ↓
JSON Error
```

---

# 44. Correlation ID

Remember Day 22.

Client:

```http
X-Correlation-ID: abc-123
```

Application:

```text
REST
 ↓
Service
 ↓
Database
```

Log:

```text
correlationId=abc-123
```

If a failure occurs:

```json
{
  "status": 500,
  "traceId": "abc-123"
}
```

Now support/operations teams can locate the corresponding logs.

For modern distributed tracing, a trace/span context may be preferable to inventing a separate correlation mechanism; understand how your platform propagates it.

---

# 45. Validation + Exception Handling Flow

Full request:

```text
POST /employees
       │
       ▼
JSON Parsing
       │
       ▼
Bean Validation
       │
   ┌───┴────┐
 Invalid   Valid
   │          │
   ▼          ▼
  400       Service
              │
              ▼
          Business
          Validation
              │
         ┌────┴────┐
        Fail      Success
         │           │
         ▼           ▼
       4xx          DB
                     │
                ┌────┴────┐
               Fail      Success
                │           │
                ▼           ▼
              5xx          201
```

---

# 46. Production Error Contract

Consider standardizing on a structure such as:

```java
public class ErrorResponse {

    private String type;
    private String title;
    private int status;
    private String detail;
    private String instance;
    private String traceId;
    private List<FieldError> errors;
}
```

Example:

```json
{
  "type": "validation-error",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Invalid employee request",
  "instance": "/api/employees",
  "traceId": "7fa21c",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email"
    }
  ]
}
```

---

# 47. File Upload Validation

From your earlier file-upload topic, apply today's principles.

Don't trust:

```text
filename
Content-Type
file extension
```

alone.

Validate:

```text
maximum file size
allowed media types
actual file content where appropriate
filename/path handling
virus/malware scanning where required
storage permissions
authorization
```

Never construct filesystem paths directly from an untrusted filename.

---

# 48. Validation and Security

Validation is not a complete security mechanism.

For example:

```text
@Size(max = 100)
```

doesn't determine whether the user is authorized to modify an employee.

You need:

```text
Validation
+
Authentication
+
Authorization
+
Business Rules
```

---

# 49. Complete Employee API Example

DTO:

```java
public class CreateEmployeeRequest {

    @NotBlank
    @Size(max = 100)
    private String name;

    @NotBlank
    @Email
    private String email;

    @NotNull
    @Positive
    private BigDecimal salary;
}
```

Resource:

```java
@Path("/employees")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class EmployeeResource {

    @Inject
    EmployeeService service;

    @POST
    public Response create(
            @Valid CreateEmployeeRequest request) {

        EmployeeResponse employee =
            service.create(request);

        return Response
            .status(Response.Status.CREATED)
            .entity(employee)
            .build();
    }
}
```

Service:

```java
public EmployeeResponse create(
        CreateEmployeeRequest request) {

    if (repository.existsByEmail(request.getEmail())) {
        throw new DuplicateEmployeeException(
            request.getEmail()
        );
    }

    Employee employee = mapper.toEntity(request);

    repository.save(employee);

    return mapper.toResponse(employee);
}
```

Exception mapper:

```java
@Provider
public class DuplicateEmployeeMapper
        implements ExceptionMapper<DuplicateEmployeeException> {

    @Override
    public Response toResponse(
            DuplicateEmployeeException ex) {

        ErrorResponse error =
            new ErrorResponse(
                409,
                "Employee already exists"
            );

        return Response
            .status(Response.Status.CONFLICT)
            .entity(error)
            .build();
    }
}
```

Architecture:

```text
Request
 ↓
DTO Validation
 ↓
Resource
 ↓
Service
 ↓
Business Validation
 ↓
Repository
 ↓
Database
```

---

# 50. Production Exception Taxonomy

A useful application-level hierarchy:

```text
ApplicationException
│
├── BusinessException
│   ├── EmployeeNotFoundException
│   ├── DuplicateEmployeeException
│   └── InsufficientBalanceException
│
└── TechnicalException
    ├── DatabaseException
    ├── ExternalServiceException
    └── MessagingException
```

Don't over-engineer this. The hierarchy should make handling clearer, not create dozens of meaningless exception classes.

---

# 51. Exception Handling for External Services

Suppose:

```text
Order Service
      ↓
Payment Service
```

Payment returns timeout.

Don't simply:

```text
catch Exception
```

and return 500.

Classify the failure:

```text
Timeout
Connection failure
4xx business response
5xx server response
```

Then apply appropriate behavior:

```text
Transient failure
      ↓
Retry / fallback / asynchronous recovery

Permanent business failure
      ↓
Business error

Unknown failure
      ↓
Safe 500
```

This connects today's topic with Day 22's retry and messaging patterns.

---

# 52. Exception Handling + Retry

Be careful:

```text
Validation failure
→ don't retry
```

Usually:

```text
Malformed message
→ don't repeatedly retry
```

Potential transient failure:

```text
Temporary network failure
→ retry may be appropriate
```

Therefore:

```text
Exception Classification
       ↓
Retryable?
       │
   ┌───┴───┐
  Yes      No
   │        │
 Retry     DLQ/error
```

---

# 53. Exception Handling + DLQ

For asynchronous processing:

```text
Message
  ↓
Consumer
  ↓
Validation
  ↓
Business Logic
  ↓
Failure
  ↓
Retry
  ↓
Retry Exhausted
  ↓
DLQ
```

But don't send every failure to DLQ.

Classify:

```text
Malformed payload → DLQ
Permanent business failure → appropriate failure handling
Temporary DB outage → retry
```

---

# 54. Day 24 Mini Project

## Build: Production-Grade Employee API

Implement:

```text
POST   /api/employees
GET    /api/employees/{id}
PUT    /api/employees/{id}
DELETE /api/employees/{id}
```

### Validation

Implement:

```text
name
email
salary
department
employeeCode
```

with:

* `@NotBlank`
* `@Email`
* `@Size`
* `@Positive`
* custom employee-code validator

### Business rules

Implement:

```text
duplicate email
employee not found
invalid salary change
department not found
```

### Exception handling

Create:

```text
EmployeeNotFoundException
DuplicateEmployeeException
DepartmentNotFoundException
InvalidSalaryException
```

### REST mapping

```text
Validation → 400
Not Found → 404
Duplicate → 409
Unauthorized → 401
Forbidden → 403
Unexpected → 500
```

### Error contract

Every error should include:

```text
type
title
status
detail
instance
traceId
field errors where applicable
```

---

# 55. Day 24 Testing Exercises

Write tests for:

### Validation

1. Empty employee name.
2. Invalid email.
3. Negative salary.
4. Missing department.
5. Oversized name.
6. Invalid employee code.
7. Invalid nested order item.
8. Invalid date range.

### Exceptions

9. Employee not found.
10. Duplicate email.
11. Database failure.
12. External service timeout.
13. Unexpected runtime exception.

### REST

14. Verify `400`.
15. Verify `404`.
16. Verify `409`.
17. Verify `500`.
18. Verify consistent error JSON.

---

# 56. Day 24 Interview Questions

### Validation

1. What is Jakarta Bean Validation?
2. Why should validation happen server-side?
3. Difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?
4. What does `@Size` do?
5. `@Min` vs `@Positive`?
6. What is `@Pattern`?
7. What is `@Valid`?
8. What are validation groups?
9. How do you create a custom validator?
10. How do you validate multiple fields together?

### Exceptions

11. Checked vs unchecked exceptions?
12. When should you create a custom exception?
13. What is exception translation?
14. Why shouldn't database exceptions reach REST clients?
15. What is an ExceptionMapper?
16. How does centralized exception handling work?
17. What should a production error response contain?
18. How should unexpected exceptions be handled?
19. Why shouldn't stack traces be returned to clients?
20. How should exceptions interact with transactions?

### REST

21. 400 vs 404?
22. 401 vs 403?
23. 409 vs 422?
24. When should an API return 500?
25. How should validation errors be represented?
26. How would you implement a global exception handler?
27. How would you attach a correlation ID?
28. How should you handle external-service timeouts?
29. Which failures should be retried?
30. How would you handle invalid asynchronous messages?

---

# 57. Architect-Level Scenario

Imagine your API receives:

```json
{
  "employeeId": 10,
  "salary": -100
}
```

You should be able to reason:

```text
Malformed/invalid input
        ↓
Bean Validation
        ↓
400
```

Now:

```json
{
  "employeeId": 999999,
  "salary": 100000
}
```

If the employee doesn't exist:

```text
Service
  ↓
EmployeeNotFoundException
  ↓
ExceptionMapper
  ↓
404
```

Now:

```json
{
  "employeeId": 10,
  "salary": 100000
}
```

but the user isn't allowed to modify that employee:

```text
Authentication
      ↓
Authorization
      ↓
403
```

Now database is unavailable:

```text
Database
   ↓
Connection failure
   ↓
Technical exception
   ↓
Log detailed failure
   ↓
Safe 500/appropriate transient response
```

The key is **classification**.

---

# 58. The Enterprise Error Pipeline

Memorize:

```text
                   HTTP Request
                        │
                        ▼
                   Parse Input
                        │
                        ▼
                    Validate
                   /        \
               Invalid       Valid
                 │             │
                 ▼             ▼
                400         Controller
                               │
                               ▼
                            Service
                               │
                         Business Rules
                          /          \
                       Fail          Pass
                        │              │
                        ▼              ▼
                      4xx           Repository
                                      │
                                  /       \
                               Fail       Success
                                │            │
                                ▼            ▼
                              Error         2xx
```

And all errors flow through:

```text
Exception
   ↓
Classification
   ↓
Logging
   ↓
Mapping
   ↓
Safe Response
```

---

# 59. Day 24 Production Checklist

Before considering an enterprise API production-ready:

* [ ] Validate all external input.
* [ ] Use DTOs.
* [ ] Don't expose persistence entities blindly.
* [ ] Separate structural validation from business validation.
* [ ] Use domain-specific exceptions where useful.
* [ ] Centralize REST exception mapping.
* [ ] Use correct HTTP status codes.
* [ ] Return consistent error responses.
* [ ] Never expose stack traces.
* [ ] Never expose secrets.
* [ ] Log detailed internal errors.
* [ ] Include correlation/trace identifiers.
* [ ] Classify retryable vs non-retryable failures.
* [ ] Don't retry validation failures.
* [ ] Handle asynchronous poison messages.
* [ ] Test validation failures.
* [ ] Test exception mappings.
* [ ] Test unexpected failures.
* [ ] Monitor error rates and latency.

---

# 60. Day 24 Golden Mental Model

```text
                INPUT
                  │
                  ▼
             VALIDATION
                  │
           ┌──────┴──────┐
        Invalid          Valid
           │               │
           ▼               ▼
          400            SERVICE
                           │
                     BUSINESS RULES
                           │
                  ┌────────┴────────┐
                Failure           Success
                   │                 │
                   ▼                 ▼
              Exception          Database
                   │                 │
                   ▼                 │
             Exception Mapper        │
                   │                 │
                   ▼                 ▼
              Safe 4xx/5xx          2xx
```

### The most important lesson

> **Validation protects your system from invalid input; exception handling protects your system from unexpected conditions; neither should replace authorization, business rules, transactions, or observability.**

By the end of Day 24, you should be able to build an API where a failure is not simply **“something broke”**, but a deliberately classified condition with the correct behavior, transaction semantics, logs, HTTP response, and recovery strategy.

**Next: Day 25 — Application Servers**, where you'll go deep into **WildFly/Payara/GlassFish/Tomcat roles, deployment architecture, WAR/EAR packaging, classloading, server configuration, JNDI, datasources, connection pools, security domains, clustering, administration, and production deployment.**
