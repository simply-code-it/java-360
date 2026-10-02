# Day 17 — REST API Security with Jakarta EE

**Date:** October 19, 2026
**Goal:** Master how to secure enterprise REST APIs—from basic authentication concepts to OAuth 2.0/JWT, authorization, CORS/CSRF, service-to-service security, and production architecture.

> **Terminology:** Modern J2EE is Jakarta EE. You'll commonly see `jakarta.*` in current applications and `javax.*` in legacy applications.

---

# 1. What You Must Master Today

By the end of Day 17, you should be able to design:

```text
Client
   ↓
HTTPS
   ↓
API Gateway / Load Balancer
   ↓
Authentication
   ↓
Authorization
   ↓
Rate Limiting
   ↓
Jakarta REST Filters/Security
   ↓
Resource
   ↓
Service
   ↓
Database
```

You should understand:

* Authentication
* Authorization
* Identity
* Principal
* Roles
* Permissions
* Sessions
* Cookies
* Basic Authentication
* API keys
* Bearer tokens
* JWT
* OAuth 2.0
* OpenID Connect
* Access tokens
* Refresh tokens
* CORS
* CSRF
* TLS
* Password hashing
* Session fixation
* Token expiration
* Token revocation
* Service-to-service authentication
* RBAC
* ABAC
* Security filters
* Security headers
* Rate limiting
* Audit logging
* Secure REST architecture

---

# 2. Authentication vs Authorization

This is the first concept you must know perfectly.

### Authentication

> **Who are you?**

Example:

```text
username = pradeep
password = ********
```

The system verifies the identity.

### Authorization

> **What are you allowed to do?**

Example:

```text
Pradeep
   ↓
Authenticated
   ↓
ROLE_EMPLOYEE
   ↓
Can view employee profile
   ↓
Cannot delete employees
```

Mental model:

```text
Authentication
     ↓
Identity
     ↓
Authorization
     ↓
Permissions
```

---

# 3. Identity

An authenticated user might have:

```json
{
  "userId": 101,
  "username": "pradeep",
  "roles": [
    "EMPLOYEE"
  ]
}
```

The security system establishes this identity.

In Jakarta Security terminology, an authenticated identity is commonly represented using a `Principal`.

---

# 4. Principal

Conceptually:

```java
Principal principal =
        securityContext.getUserPrincipal();
```

You can obtain the authenticated identity:

```java
@Context
SecurityContext securityContext;

@GET
public Response profile() {

    Principal principal =
        securityContext.getUserPrincipal();

    return Response.ok(
        principal.getName()
    ).build();
}
```

---

# 5. SecurityContext

Jakarta REST provides:

```java
@Context
SecurityContext securityContext;
```

It can provide information such as:

```text
authenticated principal
authentication scheme
secure connection
roles
```

For example:

```java
if (securityContext.isUserInRole("ADMIN")) {
    ...
}
```

---

# 6. Authentication Methods

Enterprise REST APIs can use several mechanisms.

```text
Basic Authentication
API Keys
Session Cookies
Bearer Tokens
JWT
OAuth 2.0
OpenID Connect
mTLS
```

The appropriate mechanism depends on the architecture and threat model.

---

# 7. Basic Authentication

HTTP:

```text
Authorization: Basic <credentials>
```

Conceptually:

```text
username + password
        ↓
Base64 encoding
        ↓
Authorization header
```

Important:

> Base64 is **encoding**, not encryption.

Therefore Basic Authentication should be used only over HTTPS.

---

# 8. Basic Authentication Flow

```text
Client
  ↓
Authorization: Basic credentials
  ↓
HTTPS
  ↓
Server
  ↓
Verify credentials
  ↓
Authenticated
```

Potential disadvantages:

* credentials accompany requests
* password-based authentication
* requires secure transport
* less suitable for many modern distributed architectures

---

# 9. API Keys

Example:

```text
X-API-Key: abc123...
```

or sometimes:

```text
Authorization: ApiKey abc123...
```

Common use cases:

```text
server-to-server integrations
external API consumers
internal tooling
developer APIs
```

An API key generally identifies the client/application, not necessarily a human user.

---

# 10. API Key Security

Don't:

```text
commit API keys to Git
```

Don't:

```text
print API keys in logs
```

Don't:

```text
put secrets into frontend JavaScript
```

Prefer:

```text
Secret Manager
Environment/configuration management
Key rotation
Expiration
Access restrictions
Audit logging
```

---

# 11. Session Authentication

Traditional web applications often use:

```text
JSESSIONID
```

Flow:

```text
Login
  ↓
Server authenticates user
  ↓
Creates HTTP session
  ↓
Set-Cookie: JSESSIONID=...
  ↓
Browser
```

Subsequent requests:

```text
Cookie: JSESSIONID=...
```

The server retrieves the session.

---

# 12. Secure Session Cookies

Important cookie attributes:

```text
Secure
HttpOnly
SameSite
```

### Secure

Cookie is sent over HTTPS.

### HttpOnly

JavaScript cannot directly access it through `document.cookie`.

### SameSite

Controls cross-site cookie sending behavior.

These properties help reduce specific attack classes but aren't substitutes for complete application security.

---

# 13. Session Fixation

A session fixation attack can occur when an attacker gets a victim to use a session identifier known to the attacker and the application doesn't properly change the session identifier after authentication.

Important mitigation:

```text
Before authentication
        ↓
session
        ↓
Successful authentication
        ↓
New session identifier
```

Modern security frameworks/container security should be configured to protect against session fixation.

---

# 14. Logout

Logout should invalidate the authenticated session.

Conceptually:

```java
request.getSession(false);
```

then invalidate it when appropriate.

For example:

```java
HttpSession session =
        request.getSession(false);

if (session != null) {
    session.invalidate();
}
```

For token-based systems, logout semantics are different because a self-contained access token may remain cryptographically valid until expiry unless you introduce revocation/denylisting or use short-lived tokens.

---

# 15. Bearer Tokens

Modern APIs commonly use:

```text
Authorization: Bearer <access-token>
```

Flow:

```text
Client
  ↓
Access Token
  ↓
REST API
  ↓
Validate token
  ↓
Identity
  ↓
Authorization
```

---

# 16. What Is a JWT?

JWT = JSON Web Token.

A JWT commonly consists of:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example conceptually:

```text
xxxxx.yyyyy.zzzzz
```

---

# 17. JWT Header

Example:

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

The header describes metadata such as the signing algorithm.

---

# 18. JWT Payload

Example:

```json
{
  "iss": "https://identity.example.com",
  "sub": "user-101",
  "aud": "employee-api",
  "exp": 1795000000,
  "iat": 1794996400,
  "scope": "employee:read"
}
```

These are called **claims**.

---

# 19. Important JWT Claims

### `iss`

Issuer.

```text
Who issued the token?
```

### `sub`

Subject.

```text
Who does this token represent?
```

### `aud`

Audience.

```text
Which service is this token intended for?
```

### `exp`

Expiration time.

```text
When does it expire?
```

### `iat`

Issued-at time.

```text
When was it issued?
```

Other claims can contain:

```text
scope
roles
tenant
permissions
```

depending on the design.

---

# 20. JWT Is Not Encryption

This is extremely important.

A normal signed JWT payload is typically **encoded**, not encrypted.

Therefore:

```text
Don't put secrets inside JWT claims.
```

For example, don't put:

```json
{
  "password": "secret123"
}
```

A signed JWT provides integrity/authenticity when correctly validated; confidentiality requires encryption or another protected channel.

---

# 21. JWT Signature

Conceptually:

```text
Header + Payload
       ↓
Signing algorithm
       ↓
Private key / secret
       ↓
Signature
```

The server validates:

```text
Received JWT
     ↓
Signature verification
     ↓
Claims validation
     ↓
Authenticated identity
```

---

# 22. Symmetric vs Asymmetric Signing

### Symmetric

Example:

```text
HS256
```

Same secret is used to sign and verify.

```text
Secret
 ↙    ↘
Sign  Verify
```

### Asymmetric

Example:

```text
RS256
ES256
```

Private key signs:

```text
Private Key
     ↓
Sign
```

Public key verifies:

```text
Public Key
     ↓
Verify
```

This can be advantageous in distributed systems because APIs can verify tokens using public keys without possessing the private signing key.

---

# 23. JWT Validation

Never merely decode a JWT and trust its contents.

Proper validation should include relevant checks such as:

```text
signature
issuer
audience
expiration
not-before where applicable
algorithm policy
required claims
```

For example:

```text
JWT
 ↓
Verify signature
 ↓
Verify issuer
 ↓
Verify audience
 ↓
Verify expiration
 ↓
Build authenticated identity
```

---

# 24. OAuth 2.0

OAuth 2.0 is primarily an **authorization framework**.

It allows a client to obtain access to protected resources without giving the resource server the user's password.

Core actors:

```text
Resource Owner
       ↓
Authorization Server
       ↓
Client
       ↓
Access Token
       ↓
Resource Server
```

---

# 25. OAuth 2.0 Components

### Resource Owner

Usually the user.

### Client

Application requesting access.

### Authorization Server

Issues tokens.

### Resource Server

Hosts the protected API.

Example:

```text
User
 ↓
Frontend
 ↓
Identity/Authorization Server
 ↓
Access Token
 ↓
Employee API
```

---

# 26. Authorization Code Flow

A common modern flow for user-facing applications is:

```text
Browser
   ↓
Client Application
   ↓
Authorization Server
   ↓
User Authentication
   ↓
Authorization
   ↓
Authorization Code
   ↓
Client
   ↓
Token Endpoint
   ↓
Access Token
   ↓
API
```

For public clients such as SPAs/mobile apps, **PKCE** is an important protection.

---

# 27. PKCE

PKCE = Proof Key for Code Exchange.

Conceptually:

```text
Client creates
code_verifier
      ↓
code_challenge
      ↓
Authorization Server
```

Later:

```text
Authorization Code
+
code_verifier
      ↓
Token Endpoint
```

This helps protect authorization-code flows from interception attacks.

---

# 28. Access Token

Access token answers:

```text
What API access has been granted?
```

Example:

```text
scope = employee:read employee:update
```

The resource server validates the token before allowing protected operations.

---

# 29. Refresh Token

A refresh token can be used to obtain a new access token without requiring the user to authenticate again.

Conceptually:

```text
Access Token
     ↓
expires
     ↓
Refresh Token
     ↓
Authorization Server
     ↓
New Access Token
```

Refresh tokens require strong protection and appropriate rotation/revocation policies.

---

# 30. Access Token vs Refresh Token

| Access Token               | Refresh Token                    |
| -------------------------- | -------------------------------- |
| Used to call APIs          | Used to obtain new access tokens |
| Usually short-lived        | Usually longer-lived             |
| Sent to resource server    | Sent to authorization server     |
| Limits API access lifetime | Extends authenticated session    |
| Should be tightly scoped   | Requires stronger protection     |

---

# 31. OAuth Scopes

Example:

```text
employee:read
employee:create
employee:update
employee:delete
```

Token:

```text
scope =
employee:read employee:update
```

The API can enforce:

```text
GET    → employee:read
POST   → employee:create
PUT    → employee:update
DELETE → employee:delete
```

---

# 32. Roles vs Permissions

Role:

```text
ADMIN
```

Permission:

```text
employee:delete
```

You might map:

```text
ADMIN
 ├── employee:read
 ├── employee:create
 ├── employee:update
 └── employee:delete
```

Roles simplify administration.

Permissions provide finer-grained control.

---

# 33. RBAC

RBAC = Role-Based Access Control.

Example:

```text
USER
 ↓
EMPLOYEE

MANAGER
 ↓
EMPLOYEE + REPORTING

ADMIN
 ↓
ALL EMPLOYEE OPERATIONS
```

Application checks:

```java
securityContext.isUserInRole("ADMIN");
```

---

# 34. ABAC

ABAC = Attribute-Based Access Control.

Instead of only:

```text
role = MANAGER
```

you could consider:

```text
user.department
resource.department
operation
location
time
tenant
```

Example:

```text
Manager can modify employees
only within their department.
```

Conceptually:

```text
User.department == Employee.department
```

This is more expressive than simple RBAC.

---

# 35. Object-Level Authorization

This is extremely important.

Suppose:

```text
GET /employees/101
```

A user is authenticated.

But does that mean they can access employee `101`?

**No.**

You must distinguish:

```text
Authentication
    ↓
"Who is the user?"

Authorization
    ↓
"Can this user access this resource?"
```

This is especially important for APIs because simply checking that a user has a valid token is not enough.

---

# 36. Example Authorization

Bad:

```java
if (authenticated) {
    return employeeService.findById(id);
}
```

Potentially better domain-level logic:

```java
Employee employee =
    employeeService.findById(id);

authorizationService.checkCanView(
    currentUser,
    employee
);
```

The exact architecture depends on your authorization model.

---

# 37. Multi-Tenant Security

Imagine:

```text
Tenant A
 ├── User A1
 └── User A2

Tenant B
 ├── User B1
 └── User B2
```

User A1 must not access:

```text
Tenant B data
```

A robust design propagates tenant context and enforces tenant isolation at appropriate layers.

Never rely solely on:

```text
tenantId supplied by frontend
```

as proof of authorization.

---

# 38. CORS

CORS = Cross-Origin Resource Sharing.

Suppose:

```text
Frontend:
https://app.example.com

API:
https://api.example.com
```

These are different origins.

The browser applies same-origin security rules.

CORS allows the server to explicitly declare which cross-origin browser requests are permitted.

---

# 39. CORS Headers

Common response headers:

```text
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Access-Control-Allow-Credentials
```

Example:

```text
Access-Control-Allow-Origin:
https://app.example.com
```

Avoid blindly using:

```text
*
```

especially when credentials are involved.

---

# 40. Preflight Request

Some browser requests trigger:

```text
OPTIONS
```

before the actual request.

Example:

```text
Browser
   ↓
OPTIONS /api/employees
   ↓
Server
   ↓
CORS response
   ↓
Browser
   ↓
POST /api/employees
```

This is called a **preflight request**.

---

# 41. CORS vs CSRF

Don't confuse them.

### CORS

Primarily a browser mechanism controlling cross-origin resource access.

### CSRF

An attack where a victim's browser is induced to make an unwanted authenticated request.

They solve different problems.

---

# 42. CSRF

Suppose authentication uses cookies:

```text
Browser
 ↓
Cookie: JSESSIONID=...
 ↓
POST /transfer
```

A malicious website may attempt to cause the browser to send an unwanted request.

Potential defenses include:

```text
CSRF tokens
SameSite cookies
Origin/Referer validation
appropriate authentication design
```

CSRF protection is especially relevant for cookie-based authentication.

---

# 43. JWT and CSRF

A common misconception:

> "JWT automatically solves CSRF."

Not necessarily.

The important question is:

```text
How is the credential transmitted?
```

If a credential is automatically attached by the browser, CSRF considerations remain.

If a token is explicitly placed in an `Authorization: Bearer ...` header by application code, the browser's cookie-based CSRF mechanism is different.

You still need to evaluate the entire architecture.

---

# 44. XSS

XSS = Cross-Site Scripting.

Example:

```text
User input
 ↓
Stored in database
 ↓
Rendered without escaping
 ↓
Browser executes malicious script
```

REST APIs should validate input and safely encode output where output reaches an HTML context.

Also avoid returning unsafe HTML unless explicitly required.

---

# 45. SQL Injection

Never:

```java
String sql =
    "SELECT * FROM employees WHERE name='"
    + name
    + "'";
```

Use:

```java
PreparedStatement
```

or parameterized JPA queries.

Security must exist across every layer.

---

# 46. Password Storage

Never store:

```text
password = "mypassword123"
```

Use a password hashing algorithm designed for passwords, such as:

```text
Argon2
bcrypt
scrypt
PBKDF2
```

with appropriate parameters.

Do not use ordinary fast hashes such as plain SHA-256 as a password-storage scheme.

---

# 47. Password Hashing Flow

```text
Password
   ↓
Password Hash Function
   ↓
Salt + Derived Hash
   ↓
Database
```

During login:

```text
Password
   ↓
Password verification
   ↓
Stored hash
   ↓
Match?
```

The plaintext password should not be stored.

---

# 48. HTTPS/TLS

Production REST APIs should use HTTPS.

```text
HTTP
 ↓
TLS
 ↓
HTTPS
```

TLS protects:

```text
confidentiality
integrity
server authentication
```

It protects data while traveling between endpoints.

---

# 49. Don't Put Secrets in URLs

Avoid:

```text
GET /api/login?username=x&password=y
```

and:

```text
GET /api/orders?token=secret
```

URLs can appear in:

```text
logs
browser history
proxies
monitoring systems
referrer information
```

Credentials and access tokens should not be placed in URLs.

---

# 50. Security Headers

Depending on your deployment architecture, consider appropriate security headers such as:

```text
Strict-Transport-Security
Content-Security-Policy
X-Content-Type-Options
Referrer-Policy
```

The exact set should match the application and whether responses are consumed by browsers.

---

# 51. Rate Limiting

Suppose an attacker sends:

```text
10,000 requests/sec
```

Your API needs protection.

Architecture:

```text
Client
 ↓
Rate Limiter
 ↓
REST API
```

Example policy:

```text
100 requests/minute/client
```

But real limits should be based on:

```text
API operation
user
client
IP
tenant
business requirements
infrastructure capacity
```

---

# 52. Brute-Force Protection

Login endpoints are especially sensitive.

Possible controls:

```text
rate limiting
progressive delays
account protection
MFA
monitoring
anomaly detection
```

Avoid designing authentication so that attackers can make unlimited credential attempts.

---

# 53. Security Logging

Log security events such as:

```text
login success
login failure
logout
authorization failure
token validation failure
password change
privilege change
```

But never log:

```text
password
access token
refresh token
session ID
private keys
```

in plaintext.

---

# 54. Audit Logging

Security logs and audit logs are related but not identical.

Audit event:

```json
{
  "actor": "user-101",
  "action": "EMPLOYEE_UPDATED",
  "resource": "employee-500",
  "timestamp": "2026-10-19T10:30:00Z",
  "requestId": "abc123"
}
```

For sensitive systems, audit trails can be critical for investigation and compliance.

---

# 55. Authentication Filter Architecture

A simplified Jakarta REST request filter:

```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class AuthenticationFilter
        implements ContainerRequestFilter {

    @Override
    public void filter(
            ContainerRequestContext context) {

        String header =
            context.getHeaderString(
                HttpHeaders.AUTHORIZATION
            );

        if (header == null ||
            !header.startsWith("Bearer ")) {

            context.abortWith(
                Response.status(
                    Response.Status.UNAUTHORIZED
                ).build()
            );
        }
    }
}
```

This is only a conceptual example.

A production implementation should use a well-tested security/token library rather than manually implementing cryptographic token verification.

---

# 56. 401 vs 403

Very common interview question.

### 401 Unauthorized

The request lacks valid authentication credentials.

Conceptually:

```text
Who are you?
→ Not successfully authenticated.
```

### 403 Forbidden

The server understands who you are but you don't have permission.

```text
I know who you are.
→ You're not allowed to perform this operation.
```

---

# 57. Example

```text
GET /api/admin/users
```

No valid token:

```text
401 Unauthorized
```

Valid employee token:

```text
403 Forbidden
```

Admin token:

```text
200 OK
```

---

# 58. Security Architecture

A production architecture could look like:

```text
                    Internet
                       │
                       ▼
                    HTTPS
                       │
                       ▼
              Load Balancer / WAF
                       │
                       ▼
                 API Gateway
                       │
        ┌──────────────┼───────────────┐
        │              │               │
        ▼              ▼               ▼
 Rate Limiting     CORS Policy      Routing
        │
        ▼
 Authentication
        │
        ▼
 Authorization
        │
        ▼
 Jakarta REST
        │
        ▼
    Service
        │
        ▼
   Database
```

---

# 59. API Gateway vs Application Security

Don't assume:

```text
Gateway handles security
→ Application needs nothing
```

The application should still enforce authorization appropriate to the resource.

Defense in depth:

```text
WAF
 ↓
Gateway
 ↓
Application authentication
 ↓
Application authorization
 ↓
Database controls
```

---

# 60. Service-to-Service Authentication

Imagine:

```text
Order Service
     ↓
Payment Service
```

How does Order Service authenticate?

Possible approaches:

```text
OAuth 2.0 access tokens
mTLS
workload identity
signed service credentials
```

Avoid hardcoding:

```text
username = admin
password = admin123
```

inside source code.

---

# 61. mTLS

mTLS = mutual TLS.

Normal TLS:

```text
Client → verifies server
```

mTLS:

```text
Client ↔ verifies ↔ Server
```

Both sides authenticate using certificates.

Useful in some high-security service-to-service environments.

---

# 62. Token Validation at the Resource Server

Suppose:

```text
Frontend
 ↓
Authorization Server
 ↓
Access Token
 ↓
Employee API
```

Employee API should validate that the token:

```text
was issued by trusted issuer
is correctly signed
is intended for this API
has not expired
contains sufficient scopes/claims
```

Do not blindly trust claims simply because they appear in the payload.

---

# 63. JWKS

JWT-based systems often expose public signing keys through a **JWKS** endpoint.

Conceptually:

```text
Authorization Server
       ↓
JWKS
       ↓
Public Keys
       ↓
Resource Server
       ↓
Verify JWT
```

Key rotation means APIs need a strategy for discovering and caching current verification keys.

---

# 64. Key Rotation

Cryptographic keys shouldn't necessarily live forever.

Typical lifecycle:

```text
Generate new key
      ↓
Publish verification key
      ↓
Start signing with new key
      ↓
Continue accepting old key temporarily
      ↓
Expire old tokens
      ↓
Retire old key
```

This is important in production identity infrastructure.

---

# 65. Token Expiration

Imagine:

```text
Access token
expires in 15 minutes
```

Benefits:

```text
stolen token has limited lifetime
```

But short expiration creates a refresh-token/session-management requirement for many applications.

Security is therefore a balance between:

```text
security
usability
operational complexity
```

---

# 66. Token Revocation

JWT access tokens are often designed to be independently verifiable.

That creates a trade-off:

```text
Easy distributed validation
        vs
Immediate revocation
```

Possible strategies include:

```text
short-lived access tokens
refresh token rotation
token introspection
revocation lists
centralized authorization
```

Choose based on business/security requirements.

---

# 67. Zero Trust Mental Model

Don't assume:

```text
Internal network = trusted
```

Instead:

```text
Every request
 ↓
Authenticate
 ↓
Authorize
 ↓
Validate
 ↓
Monitor
```

This becomes particularly important with microservices and cloud environments.

---

# 68. Secure Employee API

Let's design:

```text
GET /api/v1/employees
```

Request:

```text
Authorization: Bearer <token>
```

Flow:

```text
Request
  ↓
HTTPS
  ↓
Gateway
  ↓
Rate Limit
  ↓
Authentication
  ↓
Token Validation
  ↓
Authorization
  ↓
employee:read?
  ↓
EmployeeResource
  ↓
EmployeeService
  ↓
EmployeeDAO
  ↓
Database
```

---

# 69. Authorization Example

```java
@GET
@Path("/{id}")
public Response findById(
        @PathParam("id") Long id,
        @Context SecurityContext securityContext) {

    if (!securityContext
            .isUserInRole("EMPLOYEE_READ")) {

        return Response
            .status(Response.Status.FORBIDDEN)
            .build();
    }

    EmployeeResponse employee =
        service.findById(id);

    return Response.ok(employee).build();
}
```

For a larger application, move authorization decisions into a dedicated policy/service layer rather than filling resources with security conditionals.

---

# 70. Security Decision Flow

Think:

```text
Request
   ↓
Is transport secure?
   ↓
Is credential present?
   ↓
Is credential valid?
   ↓
Who is the caller?
   ↓
What scopes/roles?
   ↓
What resource?
   ↓
Is this operation allowed?
   ↓
Execute business operation
```

---

# 71. Security Threat Model

For every API ask:

```text
What are we protecting?
Who are the attackers?
What happens if authentication is bypassed?
What happens if authorization is bypassed?
What secrets exist?
Where are secrets stored?
What data is sensitive?
What needs auditing?
What happens if a token is stolen?
What happens if a service is compromised?
```

This is how security becomes architecture rather than just annotations.

---

# 72. OWASP API Security Concepts

You should become familiar with common API risks such as:

```text
Broken object-level authorization
Broken authentication
Broken object property-level authorization
Unrestricted resource consumption
Broken function-level authorization
Unrestricted access to sensitive business flows
Server-side request forgery
Security misconfiguration
Improper inventory management
Unsafe consumption of APIs
```

Don't memorize names only.

For every risk understand:

```text
attack
impact
detection
prevention
testing
monitoring
```

---

# 73. Day 17 Project

Upgrade your Employee Management System to:

# Employee Management System — Secure API

Implement:

```text
Authentication
Authorization
RBAC
JWT/OAuth2 integration concepts
CORS
CSRF analysis
Rate limiting
Security filters
Error handling
Audit logging
Request correlation
HTTPS
Secure configuration
```

---

# 74. API Security Matrix

Create this table yourself:

| Endpoint                 | Authentication | Authorization       |
| ------------------------ | -------------- | ------------------- |
| `GET /employees`         | Required       | `employee:read`     |
| `GET /employees/{id}`    | Required       | Object-level access |
| `POST /employees`        | Required       | `employee:create`   |
| `PUT /employees/{id}`    | Required       | `employee:update`   |
| `DELETE /employees/{id}` | Required       | `employee:delete`   |
| `GET /profile`           | Required       | Own profile         |
| `POST /logout`           | Required       | Own session         |

The important lesson is that **authentication requirements and authorization requirements are separate design decisions**.

---

# 75. Day 17 Coding Exercises

### Level 1

1. Read `Authorization` header.
2. Detect Bearer token.
3. Return `401` for missing authentication.
4. Return `403` for insufficient permissions.
5. Read authenticated principal.
6. Check a role using `SecurityContext`.

### Level 2

7. Build an authentication filter.
8. Build an authorization filter.
9. Build a correlation-ID filter.
10. Implement secure session logout.
11. Implement CORS configuration.
12. Build a standardized security error response.
13. Add audit logging.

### Level 3

14. Design JWT validation.
15. Design OAuth authorization-code flow.
16. Add PKCE to the design.
17. Design service-to-service authentication.
18. Design key rotation.
19. Design token revocation.
20. Design multi-tenant authorization.

---

# 76. Day 17 Interview Questions

### Fundamentals

**1. Authentication vs authorization?**

Authentication establishes identity; authorization determines permitted actions/resources.

**2. What is a Principal?**

A representation of the authenticated identity.

**3. What is `SecurityContext`?**

Jakarta REST's contextual API for obtaining security-related information about the current request.

**4. What is RBAC?**

Authorization based on roles assigned to identities.

**5. What is ABAC?**

Authorization based on attributes of the subject, resource, action, and/or environment.

---

# 77. Security Interview Questions

**6. JWT vs session?**

Session authentication stores authentication state server-side; JWT-based systems can carry signed claims in a token that the resource server validates. Each has different scalability, revocation, storage, and security trade-offs.

**7. Is JWT encrypted?**

Not necessarily. A normal signed JWT is generally encoded and signed, not encrypted.

**8. What is OAuth 2.0?**

An authorization framework for delegated access to protected resources.

**9. What is OpenID Connect?**

An identity layer built on top of OAuth 2.0 for authentication and identity information.

**10. What is PKCE?**

A mechanism that protects authorization-code flows using a client-generated verifier/challenge.

---

# 78. Scenario Question

### "A user has a valid JWT. Can they delete any employee?"

No.

The token establishes authentication and may contain authorization information, but the application must still enforce:

```text
required permission
+
resource-level authorization
+
business rules
```

For example:

```text
DELETE /employees/500
```

requires checking whether this particular caller can delete employee `500`.

---

# 79. Scenario Question

### "Why shouldn't we put the JWT in a URL?"

Because URLs can leak through:

```text
browser history
server logs
proxy logs
monitoring systems
referrers
```

Use an appropriate authorization mechanism such as:

```text
Authorization: Bearer <token>
```

over HTTPS.

---

# 80. Scenario Question

### "Why isn't CORS an API authentication mechanism?"

Because CORS controls browser cross-origin behavior. It doesn't establish the identity of an API caller.

```text
CORS
→ Browser origin policy

Authentication
→ Caller identity
```

---

# 81. Scenario Question

### "How would you secure a microservices environment?"

A possible architecture:

```text
                    Identity Provider
                          │
                          ▼
Client ──HTTPS──> API Gateway
                    │
              Authentication
                    │
              Rate Limiting
                    │
                    ▼
               Order Service
                    │
             Service Identity
                    │
                    ▼
              Payment Service
                    │
                    ▼
                 Database
```

Then add:

```text
authorization
mTLS/workload identity where appropriate
secret management
audit logging
distributed tracing
network policies
key rotation
least privilege
```

---

# 82. Production Security Checklist

Before deploying your API:

### Transport

```text
☐ HTTPS
☐ TLS configuration
☐ HSTS where appropriate
```

### Authentication

```text
☐ Strong authentication mechanism
☐ Token/session expiration
☐ Secure credential storage
☐ Secure logout/session handling
```

### Authorization

```text
☐ RBAC/ABAC
☐ Object-level authorization
☐ Least privilege
☐ Tenant isolation
```

### API

```text
☐ Input validation
☐ Output encoding where relevant
☐ SQL injection protection
☐ Rate limiting
☐ Request size limits
☐ Consistent errors
```

### Secrets

```text
☐ No secrets in Git
☐ Secret manager
☐ Key rotation
☐ Token protection
```

### Observability

```text
☐ Audit logs
☐ Security events
☐ Correlation IDs
☐ Monitoring
☐ Alerts
```

---

# 83. Day 17 Mastery Test

You should be able to answer these without notes:

1. Authentication vs authorization?
2. What is a Principal?
3. What is SecurityContext?
4. Basic Auth vs Bearer authentication?
5. What is JWT?
6. Is JWT encrypted?
7. What are `iss`, `sub`, `aud`, `exp`?
8. Symmetric vs asymmetric signing?
9. How is a JWT validated?
10. What is OAuth 2.0?
11. OAuth client vs resource server?
12. Authorization Code flow?
13. Why PKCE?
14. Access token vs refresh token?
15. What is RBAC?
16. What is ABAC?
17. What is object-level authorization?
18. What is CORS?
19. What is a preflight request?
20. CORS vs CSRF?
21. What is session fixation?
22. How should passwords be stored?
23. Why HTTPS?
24. Why not put tokens in URLs?
25. 401 vs 403?
26. How do you secure service-to-service calls?
27. How do you rotate signing keys?
28. How do you handle token revocation?
29. How do you secure a multi-tenant API?
30. How would you design security for 100+ microservices?

---

# 84. Day 17 Final Mental Model

Memorize this:

```text
                         CLIENT
                           │
                         HTTPS
                           │
                           ▼
                    API Gateway / WAF
                           │
                     Rate Limiting
                           │
                           ▼
                    Authentication
                           │
                     ┌─────┴─────┐
                     │           │
                   Valid       Invalid
                     │           │
                     ▼           ▼
              Authorization     401
                     │
              ┌──────┴──────┐
              │             │
            Allowed       Denied
              │             │
              ▼             ▼
           Resource        403
              │
           Service
              │
          Transaction
              │
             DAO
              │
           Database
```

### The most important principle

> **Authentication answers "Who are you?" Authorization answers "What are you allowed to do?"**

And for production APIs:

```text
Secure transport
      +
Strong authentication
      +
Fine-grained authorization
      +
Input validation
      +
Least privilege
      +
Secret management
      +
Rate limiting
      +
Auditability
      +
Monitoring
      =
Secure REST Architecture
```

**Day 18:** CDI / Dependency Injection — CDI container, beans, scopes, `@Inject`, qualifiers, producers, alternatives, interceptors, decorators, lifecycle callbacks, dependency management, and how CDI forms the foundation for loosely coupled Jakarta EE applications.
