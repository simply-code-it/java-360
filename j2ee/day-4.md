# J2EE Mastery — Day 4

## Sessions, Cookies & Authentication Basics

**Date:** October 6, 2026
**Goal:** Understand how enterprise web applications maintain user state and implement basic authentication using Servlets.

> **Important terminology:** Modern J2EE is **Jakarta EE**. Older applications commonly use `javax.servlet.*`; modern Jakarta EE applications use `jakarta.servlet.*`.

---

# 1. What You Will Master Today

By the end of Day 4, you should understand:

1. Why HTTP is stateless
2. What state means in a web application
3. Cookies
4. Sessions
5. `HttpSession`
6. `JSESSIONID`
7. Session creation and invalidation
8. Session timeout
9. URL rewriting
10. Cookies vs sessions
11. Authentication vs authorization
12. Login/logout flow
13. Session-based authentication
14. Session fixation
15. CSRF basics
16. Secure cookie attributes
17. Remember-me concepts
18. Concurrent sessions
19. Authentication filters
20. Production security practices

---

# 2. Why Do We Need Sessions?

HTTP is fundamentally **stateless**.

Suppose a user makes:

```text
GET /products
```

Then:

```text
GET /cart
```

Then:

```text
GET /checkout
```

The server doesn't inherently know that all three requests came from the same logical user.

Each HTTP request is independent.

```text
Request 1 → Server
Request 2 → Server
Request 3 → Server
```

We need a mechanism to maintain state.

Common approaches:

```text
1. Cookies
2. Sessions
3. URL rewriting
4. Hidden form fields
5. Tokens
6. Server-side storage
```

For traditional Servlet applications, **HTTP sessions + cookies** are fundamental concepts.

---

# 3. Stateful vs Stateless

## Stateless

The server doesn't maintain conversational state between requests.

Example:

```text
GET /products
Authorization: Bearer <token>
```

The request contains the information needed to identify/authenticate the user.

Common with:

```text
REST APIs
JWT
OAuth 2.0
microservices
```

---

## Stateful

The server maintains session state.

```text
Browser
   |
   | JSESSIONID=ABC123
   ↓
Server
   |
   └── Session ABC123
        ├── userId = 101
        ├── username = pradeep
        └── role = ADMIN
```

---

# 4. What Is a Cookie?

A cookie is a small piece of data associated with a browser and a domain.

Server sends:

```http
Set-Cookie: username=pradeep
```

Browser stores it.

Future request:

```http
Cookie: username=pradeep
```

Conceptually:

```text
Server
  |
  | Set-Cookie
  ↓
Browser
  |
  | Cookie
  ↓
Server
```

---

# 5. Creating Cookies in Servlet

```java
@WebServlet("/create-cookie")
public class CookieServlet extends HttpServlet {

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        Cookie cookie = new Cookie("username", "pradeep");

        response.addCookie(cookie);

        response.getWriter()
                .println("Cookie created");
    }
}
```

---

# 6. Reading Cookies

```java
Cookie[] cookies = request.getCookies();

if (cookies != null) {

    for (Cookie cookie : cookies) {

        if ("username".equals(cookie.getName())) {

            String username = cookie.getValue();

            response.getWriter()
                    .println(username);
        }
    }
}
```

Always handle:

```java
request.getCookies() == null
```

because the request may contain no cookies.

---

# 7. Cookie Properties

A production cookie can have several important attributes.

```text
Name
Value
Domain
Path
Max-Age
Expires
Secure
HttpOnly
SameSite
```

Example:

```http
Set-Cookie: SESSION=abc123;
Path=/;
Secure;
HttpOnly;
SameSite=Lax
```

---

# 8. HttpOnly

`HttpOnly` prevents JavaScript from accessing the cookie through APIs such as:

```javascript
document.cookie
```

Useful for session cookies.

Conceptually:

```text
Browser JavaScript
       X
       |
       | cannot directly access
       ↓
HttpOnly Cookie
```

This reduces exposure of session cookies to certain XSS attacks.

It does **not** make an application immune to XSS.

---

# 9. Secure Cookie

`Secure` tells the browser to send the cookie only over HTTPS.

```text
HTTPS → Cookie sent
HTTP  → Cookie not sent
```

Production authentication cookies should generally use:

```text
Secure
HttpOnly
appropriate SameSite policy
```

---

# 10. SameSite

`SameSite` controls when browsers send cookies in cross-site requests.

Common values:

```text
Strict
Lax
None
```

### Strict

Most restrictive.

### Lax

Allows cookies in certain top-level navigations.

### None

Allows cross-site cookie sending, but requires:

```text
Secure
```

This attribute is particularly relevant to **CSRF protection** and modern browser behavior.

---

# 11. Cookie Size

Cookies are intended for relatively small pieces of data.

Don't put this inside cookies:

```text
Large JSON
Database records
Large shopping carts
Passwords
Sensitive application state
```

Instead:

```text
Cookie → session identifier
Server → actual session state
```

---

# 12. What Is a Session?

A session represents a user's interaction with an application over multiple HTTP requests.

Example:

```text
Session ID = ABC123

Session:
    userId = 101
    username = pradeep
    role = USER
    cartId = 5001
```

The browser generally carries the session identifier, while the server maintains the associated state.

---

# 13. HttpSession

Servlet provides:

```java
HttpSession
```

Package in modern Jakarta Servlet:

```java
jakarta.servlet.http.HttpSession
```

Legacy:

```java
javax.servlet.http.HttpSession
```

---

# 14. Creating a Session

```java
HttpSession session =
        request.getSession();
```

If a session doesn't exist, one is created.

---

# 15. Getting Existing Session Only

Sometimes you don't want to create a new session.

Use:

```java
HttpSession session =
        request.getSession(false);
```

If no session exists:

```java
session == null
```

This is extremely useful for authentication checks.

Example:

```java
HttpSession session =
        request.getSession(false);

if (session == null) {
    response.sendRedirect("/login");
    return;
}
```

---

# 16. Store Data in Session

```java
HttpSession session =
        request.getSession();

session.setAttribute(
        "username",
        "pradeep"
);
```

Another example:

```java
session.setAttribute("userId", 101L);
session.setAttribute("role", "USER");
```

---

# 17. Retrieve Session Data

```java
String username =
        (String) session.getAttribute("username");
```

For role:

```java
String role =
        (String) session.getAttribute("role");
```

---

# 18. Remove Session Attribute

```java
session.removeAttribute("username");
```

---

# 19. Invalidate Session

Logout should generally invalidate the session:

```java
session.invalidate();
```

Example:

```java
HttpSession session =
        request.getSession(false);

if (session != null) {
    session.invalidate();
}

response.sendRedirect("/login");
```

---

# 20. Session Timeout

A session shouldn't live forever.

You can configure timeout.

In `web.xml`:

```xml
<session-config>
    <session-timeout>30</session-timeout>
</session-config>
```

This means:

```text
30 minutes of inactivity
```

You can also configure programmatically:

```java
session.setMaxInactiveInterval(1800);
```

`1800` seconds = 30 minutes.

---

# 21. Session ID

A session typically has an identifier:

```java
String sessionId =
        session.getId();
```

Example:

```text
ABC123XYZ
```

The browser needs to send this ID with subsequent requests.

---

# 22. JSESSIONID

Servlet containers commonly use:

```text
JSESSIONID
```

Example:

```http
Cookie: JSESSIONID=ABC123XYZ
```

Flow:

```text
1. Browser → Login request

2. Server creates session

3. Server → Set-Cookie: JSESSIONID=ABC123XYZ

4. Browser stores cookie

5. Browser → Cookie: JSESSIONID=ABC123XYZ

6. Server finds session ABC123XYZ
```

---

# 23. Session Architecture

Think about it like this:

```text
                Browser
                   |
                   | JSESSIONID=ABC123
                   ↓
             Servlet Container
                   |
                   ↓
             Session Store
                   |
          ┌────────┴────────┐
          ↓                 ↓
      userId=101        role=USER
```

---

# 24. Cookies vs Sessions

| Feature            | Cookie                      | Session                 |
| ------------------ | --------------------------- | ----------------------- |
| Stored             | Client/browser              | Server/container        |
| Size               | Small                       | Larger                  |
| Lifetime           | Configurable                | Timeout/configurable    |
| Server memory      | No                          | Usually yes             |
| Identifier         | Cookie itself               | Session ID              |
| Security           | Needs careful configuration | Server-side state       |
| Typical use        | Preferences/session ID      | User state              |
| Sent with requests | Yes                         | Session ID usually sent |

Important:

**Sessions often use cookies.**

They are not mutually exclusive concepts.

---

# 25. URL Rewriting

What happens if cookies are disabled?

Servlet containers can use URL rewriting.

Example:

```text
/products;jsessionid=ABC123
```

Servlet API:

```java
String url =
        response.encodeURL("/products");
```

The container can add the session identifier when required.

For redirect:

```java
String url =
        response.encodeRedirectURL("/dashboard");
```

Modern applications generally rely on cookies, but you should understand URL rewriting for legacy J2EE applications and interviews.

---

# 26. Authentication vs Authorization

These two concepts are frequently confused.

## Authentication

**Who are you?**

Example:

```text
username = pradeep
password = ********
```

Server verifies identity.

---

## Authorization

**What are you allowed to do?**

Example:

```text
USER → view orders
ADMIN → view + delete users
```

Flow:

```text
Authentication
      ↓
Who are you?
      ↓
Authorization
      ↓
What can you access?
```

---

# 27. Basic Login Flow

Traditional Servlet application:

```text
Browser
   |
   | GET /login
   ↓
LoginServlet
   |
   ↓
Login Page
   |
   | POST username/password
   ↓
AuthenticationService
   |
   ↓
UserRepository
   |
   ↓
Database
```

If successful:

```text
Create Session
      ↓
Store user identity
      ↓
Redirect /dashboard
```

---

# 28. Login Form

Example JSP:

```jsp
<form method="post"
      action="${pageContext.request.contextPath}/login">

    <label>Username</label>

    <input type="text"
           name="username"
           required>

    <label>Password</label>

    <input type="password"
           name="password"
           required>

    <button type="submit">
        Login
    </button>

</form>
```

---

# 29. Login Servlet

```java
@WebServlet("/login")
public class LoginServlet extends HttpServlet {

    private final AuthenticationService
            authenticationService =
            new AuthenticationService();

    @Override
    protected void doPost(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        String username =
                request.getParameter("username");

        String password =
                request.getParameter("password");

        User user =
                authenticationService
                        .authenticate(
                                username,
                                password
                        );

        if (user == null) {

            response.sendRedirect(
                    "login?error=true"
            );

            return;
        }

        HttpSession session =
                request.getSession();

        session.setAttribute(
                "userId",
                user.getId()
        );

        session.setAttribute(
                "username",
                user.getUsername()
        );

        session.setAttribute(
                "role",
                user.getRole()
        );

        response.sendRedirect("dashboard");
    }
}
```

This demonstrates the basic concept, but production applications should use stronger security architecture.

---

# 30. Never Store Plain Passwords

Never do:

```java
session.setAttribute(
    "password",
    password
);
```

Never store database passwords as:

```text
password123
admin123
```

Instead, passwords should be stored using an appropriate password hashing algorithm such as:

```text
Argon2
bcrypt
scrypt
PBKDF2
```

Conceptually:

```text
Password
   ↓
Password Hashing
   ↓
Database
```

During login:

```text
Submitted Password
       ↓
Password Verification
       ↓
Stored Hash
```

---

# 31. Logout

A simple logout servlet:

```java
@WebServlet("/logout")
public class LogoutServlet extends HttpServlet {

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        HttpSession session =
                request.getSession(false);

        if (session != null) {
            session.invalidate();
        }

        response.sendRedirect("login");
    }
}
```

---

# 32. Protecting a Page

Suppose:

```text
/dashboard
```

requires authentication.

You could check:

```java
HttpSession session =
        request.getSession(false);

if (session == null ||
    session.getAttribute("userId") == null) {

    response.sendRedirect("/login");

    return;
}
```

But doing this in every servlet creates duplication.

Bad:

```text
DashboardServlet → authentication code
OrderServlet     → authentication code
ProfileServlet   → authentication code
AdminServlet     → authentication code
```

Better:

```text
             AuthenticationFilter
                    |
        ┌───────────┼───────────┐
        ↓           ↓           ↓
   Dashboard     Orders      Profile
```

---

# 33. Authentication Filter

```java
@WebFilter("/*")
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

        String path =
                httpRequest.getRequestURI();

        HttpSession session =
                httpRequest.getSession(false);

        boolean loggedIn =
                session != null &&
                session.getAttribute("userId")
                        != null;

        boolean publicPage =
                path.endsWith("/login") ||
                path.endsWith("/register") ||
                path.contains("/css/") ||
                path.contains("/js/");

        if (!loggedIn && !publicPage) {

            httpResponse.sendRedirect(
                    httpRequest.getContextPath()
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

This introduces an important enterprise architecture concept:

> **Cross-cutting concerns belong in filters/interceptors/security infrastructure rather than being duplicated across business servlets.**

---

# 34. Authorization Filter

Authentication:

```text
Is the user logged in?
```

Authorization:

```text
Does the user have permission?
```

Example:

```java
String role =
        (String) session.getAttribute("role");

if (!"ADMIN".equals(role)) {

    response.sendError(
            HttpServletResponse.SC_FORBIDDEN
    );

    return;
}
```

HTTP status:

```text
401 Unauthorized
```

generally means authentication is required/failed.

```text
403 Forbidden
```

generally means the server understood the request but access is not permitted.

---

# 35. Session Fixation

This is an important authentication security issue.

Conceptually:

```text
Attacker knows session ID
          ↓
Victim logs in
          ↓
Same session remains authenticated
          ↓
Attacker may reuse session ID
```

A defensive strategy is to change the session identifier when authentication succeeds.

Modern Servlet APIs support:

```java
request.changeSessionId();
```

Example:

```java
HttpSession session =
        request.getSession();

request.changeSessionId();

session.setAttribute(
        "userId",
        user.getId()
);
```

The exact approach can depend on Servlet/container version and authentication architecture.

---

# 36. Why Change Session ID?

Before login:

```text
JSESSIONID=ABC
```

After successful authentication:

```text
JSESSIONID=XYZ
```

The authenticated session identifier is different.

Conceptually:

```text
Anonymous Session
       |
       | Login
       ↓
Change Session ID
       |
       ↓
Authenticated Session
```

---

# 37. CSRF

CSRF = Cross-Site Request Forgery.

Imagine a user is authenticated to:

```text
bank.example
```

A malicious website attempts to cause the browser to submit:

```text
POST /transfer
```

Because browsers may automatically attach authentication cookies, the server needs CSRF defenses for cookie-based authentication.

Common defenses include:

```text
CSRF tokens
SameSite cookies
Origin/Referer validation
Framework security mechanisms
```

---

# 38. CSRF Token Concept

Server generates:

```text
CSRF_TOKEN=ABC123
```

Form contains:

```html
<input type="hidden"
       name="csrfToken"
       value="ABC123">
```

Request:

```text
POST /transfer

csrfToken=ABC123
```

Server validates:

```text
Submitted token
       ↓
Expected token
       ↓
Match?
   /       \
 Yes       No
 ↓          ↓
Allow      Reject
```

---

# 39. Authentication Architecture

A clean architecture:

```text
                 Browser
                    |
                    ↓
             AuthenticationFilter
                    |
                    ↓
              LoginServlet
                    |
                    ↓
          AuthenticationService
                    |
                    ↓
               UserDAO
                    |
                    ↓
                Database
```

After successful login:

```text
Session
├── userId
├── username
└── roles
```

---

# 40. Production Session Architecture

For a single server:

```text
Browser
   |
   ↓
Load Balancer
   |
   ↓
Application Server
   |
   ↓
Memory Session
```

But with multiple servers:

```text
                  Load Balancer
                 /      |      \
                /       |       \
              App1    App2     App3
```

Problem:

```text
Request 1 → App1 → Session exists
Request 2 → App2 → Session missing
```

This leads to distributed session-management problems.

---

# 41. Sticky Sessions

One solution:

```text
User A
  |
  ↓
Load Balancer
  |
  ↓
Always App1
```

This is called:

```text
Session Affinity
Sticky Sessions
```

Advantages:

* Simple
* Easy to implement

Disadvantages:

* Uneven traffic
* Server failure can lose session state
* Scaling becomes less flexible

---

# 42. Distributed Session Storage

Another approach:

```text
             Load Balancer
              /    |    \
             ↓     ↓     ↓
           App1   App2   App3
             \      |     /
              \     |    /
               ↓    ↓   ↓
              Shared Session Store
```

Possible technologies:

```text
Redis
Database
Distributed cache
```

Then:

```text
App1 → Redis
App2 → Redis
App3 → Redis
```

All instances can access the same session state.

---

# 43. Session vs JWT

You should understand this distinction for backend interviews.

### Session

```text
Client → Session ID
Server → Session state
```

### JWT

```text
Client → JWT
Server → validates token
```

Session:

```text
State primarily server-side
```

JWT:

```text
Claims primarily inside token
```

Neither approach is universally better; architecture, security requirements, scaling model, revocation requirements, and client types determine the appropriate design.

---

# 44. Remember Me

A "Remember Me" feature means the user remains authenticated beyond the normal session lifetime.

A naive approach would be:

```text
Cookie:
username=pradeep
password=...
```

**Never do this.**

Production systems instead use a secure, revocable credential/token mechanism.

Conceptually:

```text
Remember-me token
        ↓
Secure persistent cookie
        ↓
Server validates token
        ↓
New authenticated session
```

---

# 45. Don't Put Sensitive Information in Cookies

Avoid:

```text
password
database credentials
private keys
sensitive personal data
```

Even if a cookie is:

```text
HttpOnly
Secure
```

its contents may still be exposed through other mechanisms such as browser tooling or network/server-side handling depending on context.

---

# 46. Session Security Checklist

For authentication sessions:

```text
✓ HTTPS
✓ Secure cookie
✓ HttpOnly cookie
✓ Appropriate SameSite
✓ Strong session IDs
✓ Session expiration
✓ Logout invalidation
✓ Session ID rotation after authentication
✓ CSRF protection where applicable
✓ Don't store passwords in sessions
✓ Don't expose session IDs in URLs unnecessarily
✓ Validate authorization on the server
```

---

# 47. Complete Mini Application

Today we'll extend:

```text
EmployeeWebApp
```

Architecture:

```text
employee-webapp/
│
├── src/main/java/
│   │
│   └── com/example/
│       │
│       ├── servlet/
│       │   ├── LoginServlet.java
│       │   ├── LogoutServlet.java
│       │   ├── DashboardServlet.java
│       │   └── ProfileServlet.java
│       │
│       ├── filter/
│       │   └── AuthenticationFilter.java
│       │
│       ├── service/
│       │   └── AuthenticationService.java
│       │
│       ├── dao/
│       │   └── UserDAO.java
│       │
│       └── model/
│           └── User.java
│
└── src/main/webapp/
    │
    ├── login.jsp
    ├── dashboard.jsp
    └── profile.jsp
```

---

# 48. User Model

```java
public class User {

    private Long id;
    private String username;
    private String passwordHash;
    private String role;

    public User(
            Long id,
            String username,
            String passwordHash,
            String role) {

        this.id = id;
        this.username = username;
        this.passwordHash = passwordHash;
        this.role = role;
    }

    public Long getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }

    public String getPasswordHash() {
        return passwordHash;
    }

    public String getRole() {
        return role;
    }
}
```

---

# 49. Authentication Service

Conceptually:

```java
public class AuthenticationService {

    private final UserDAO userDAO =
            new UserDAO();

    public User authenticate(
            String username,
            String password) {

        User user =
                userDAO.findByUsername(username);

        if (user == null) {
            return null;
        }

        // Production:
        // use a password-hashing library
        // to verify the supplied password.

        return verifyPassword(
                password,
                user
        ) ? user : null;
    }

    private boolean verifyPassword(
            String password,
            User user) {

        // Password verification implementation
        return true;
    }
}
```

The important architecture is:

```text
Servlet
   ↓
Service
   ↓
DAO
   ↓
Database
```

Not:

```text
Servlet
   ↓
SQL + business logic + authentication
```

---

# 50. Login Flow — Full Picture

```text
                   Browser
                      |
                      | POST /login
                      ↓
                LoginServlet
                      |
                      ↓
          AuthenticationService
                      |
                      ↓
                   UserDAO
                      |
                      ↓
                  Database
                      |
              credentials valid?
                 /          \
               No            Yes
               |              |
               ↓              ↓
          Login error    Change Session ID
                              |
                              ↓
                       Create/Update Session
                              |
                       ┌──────┼──────┐
                       ↓      ↓      ↓
                    userId username role
                              |
                              ↓
                         Redirect
                              |
                              ↓
                         Dashboard
```

---

# 51. GET vs POST for Login

Login credentials should normally be submitted using:

```http
POST /login
```

rather than:

```http
GET /login?username=...&password=...
```

Why?

GET parameters can appear in:

```text
URL
browser history
server logs
proxy logs
analytics systems
```

HTTPS is still required because POST does not itself encrypt traffic.

---

# 52. PRG Pattern

After successful login:

Don't normally do:

```text
POST /login
    ↓
render dashboard directly
```

Prefer:

```text
POST /login
    ↓
302 Redirect
    ↓
GET /dashboard
```

This is:

```text
Post/Redirect/Get
```

It helps prevent accidental form resubmission when refreshing the dashboard.

---

# 53. Session Attribute Naming

Avoid random names:

```java
session.setAttribute("x", ...);
session.setAttribute("abc", ...);
```

Prefer clearly defined names:

```java
"userId"
"username"
"role"
"authenticated"
```

For larger applications, centralize constants:

```java
public final class SessionAttributes {

    public static final String USER_ID =
            "userId";

    public static final String ROLE =
            "role";

    private SessionAttributes() {
    }
}
```

Then:

```java
session.setAttribute(
        SessionAttributes.USER_ID,
        user.getId()
);
```

---

# 54. Session Lifecycle

Understand this lifecycle extremely well:

```text
No Session
    |
    | getSession()
    ↓
Session Created
    |
    ↓
Requests
    |
    ↓
Inactive
    |
    ↓
Timeout
    |
    ↓
Session Destroyed
```

Explicit destruction:

```java
session.invalidate();
```

---

# 55. Session Lifecycle Events

Servlet listeners can observe session events.

Example:

```java
@WebListener
public class SessionListener
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

Useful for:

```text
monitoring
auditing
active-user counting
resource cleanup
```

---

# 56. Important Interview Question

### What happens when a user logs in?

A good answer:

> The client sends credentials to the authentication endpoint over HTTPS. The server validates the credentials against the stored password hash. After successful authentication, the application establishes an authenticated session, rotates the session identifier where appropriate to prevent session fixation, stores minimal identity/authorization information in the session, and redirects the user to the protected resource. Subsequent requests carry the session identifier, typically through a secure, HttpOnly cookie.

---

# 57. Important Interview Questions

### Q1. Why is HTTP stateless?

Each request is independently processed and HTTP itself does not inherently maintain conversational state.

---

### Q2. What is `HttpSession`?

It is the Servlet API abstraction for maintaining server-side state associated with a client across multiple requests.

---

### Q3. What is JSESSIONID?

A commonly used cookie name containing the identifier used to associate HTTP requests with a Servlet session.

---

### Q4. Difference between:

```java
request.getSession()
```

and:

```java
request.getSession(false)
```

`getSession()` creates a session if necessary.

`getSession(false)` returns an existing session or `null`.

---

### Q5. How do you logout?

Typically:

```java
HttpSession session =
        request.getSession(false);

if (session != null) {
    session.invalidate();
}
```

---

### Q6. What is session timeout?

The period of inactivity after which the container can invalidate a session.

---

### Q7. What is authentication?

Verifying the identity of a user.

---

### Q8. What is authorization?

Determining what an authenticated user is allowed to access.

---

### Q9. What is session fixation?

An attack in which an attacker attempts to cause a victim to authenticate using a session identifier known to the attacker.

A key mitigation is session-ID rotation after authentication.

---

### Q10. Why use filters for authentication?

Authentication is a cross-cutting concern. A filter can enforce it consistently before requests reach protected servlets.

---

### Q11. What is CSRF?

An attack where a victim's browser is induced to make an unwanted state-changing request to an application where the victim is already authenticated.

---

### Q12. Why is `HttpOnly` useful?

It prevents client-side JavaScript from directly reading the cookie.

---

### Q13. Why is `Secure` useful?

It instructs the browser to send the cookie only over HTTPS.

---

### Q14. Can sessions work without cookies?

Yes. URL rewriting can be used to carry the session identifier, although cookies are generally preferred.

---

### Q15. Where should session state be stored in a distributed application?

Possible approaches include:

```text
distributed session store
database
Redis/distributed cache
```

depending on architecture and requirements.

---

# 58. Day 4 Coding Exercises

Complete these in Java Servlets.

### Beginner

**1. Cookie creation**

Create:

```text
username=pradeep
```

---

**2. Cookie reader**

Display all cookies.

---

**3. Session creation**

Create a session and store:

```text
username
userId
```

---

**4. Session reader**

Read and display session information.

---

**5. Logout**

Invalidate the session.

---

### Intermediate

**6. Login system**

Implement:

```text
/login
/dashboard
/logout
```

---

**7. Authentication Filter**

Protect:

```text
/dashboard/*
```

---

**8. Role authorization**

Allow:

```text
ADMIN → /admin/*
USER  → /user/*
```

---

**9. Session timeout**

Configure:

```text
30 minutes
```

---

**10. CSRF protection**

Implement a simple CSRF token for a state-changing form.

---

# 59. Day 4 Mini Project

## Employee Portal Authentication System

Build:

```text
Employee Portal
```

with:

```text
/register
/login
/logout
/dashboard
/profile
/employees
/admin
```

Architecture:

```text
                    Browser
                       |
                       ↓
                 Filter Chain
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Logging      Auth        Security
        Filter      Filter        Filter
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
```

Features:

```text
✓ Registration
✓ Login
✓ Logout
✓ Session management
✓ Role-based authorization
✓ Session timeout
✓ Authentication filter
✓ Authorization filter
✓ CSRF protection
✓ Secure cookies
✓ Password hashing
✓ Error handling
✓ Audit logging
```

---

# 60. Production Architecture

For a real enterprise application, don't stop at:

```text
Servlet + HttpSession
```

You should eventually understand:

```text
                    Client
                       |
                    HTTPS
                       |
                 Load Balancer
                       |
                Authentication
                       |
                 Filter Chain
                       |
              ┌────────┴────────┐
              ↓                 ↓
         Authorization      CSRF
              |
              ↓
          Controller
              |
              ↓
           Service
              |
              ↓
             DAO
              |
              ↓
          Database
```

And for multiple instances:

```text
              Load Balancer
              /     |      \
             ↓      ↓       ↓
           App1    App2    App3
             \      |       /
              \     |      /
               ↓    ↓     ↓
              Redis / DB
            Session Store
```

---

# 61. Day 4 Mental Model

Memorize this:

```text
HTTP is stateless
       ↓
We need state
       ↓
Cookies identify the client
       ↓
Session stores server-side state
       ↓
JSESSIONID identifies the session
       ↓
Authentication establishes identity
       ↓
Authorization determines permissions
       ↓
Filters enforce cross-cutting security
       ↓
Logout invalidates the session
```

And remember:

```text
Authentication ≠ Authorization

Cookie ≠ Session

Session ID ≠ User Identity

401 ≠ 403
```

---

# 62. Day 4 Mastery Checklist

Before moving to Day 5, you should be able to explain and implement:

* [ ] HTTP statelessness
* [ ] Cookies
* [ ] Cookie lifecycle
* [ ] `HttpOnly`
* [ ] `Secure`
* [ ] `SameSite`
* [ ] `HttpSession`
* [ ] `JSESSIONID`
* [ ] Session attributes
* [ ] Session timeout
* [ ] Session invalidation
* [ ] URL rewriting
* [ ] Authentication
* [ ] Authorization
* [ ] Login/logout
* [ ] Authentication filter
* [ ] Role-based authorization
* [ ] Session fixation
* [ ] CSRF
* [ ] Password hashing concepts
* [ ] Sticky sessions
* [ ] Distributed sessions
* [ ] Session vs JWT
* [ ] PRG pattern

### The key enterprise mental model

```text
              ┌───────────────────────┐
              │        Browser        │
              └───────────┬───────────┘
                          │
                       HTTPS
                          │
                          ↓
              ┌───────────────────────┐
              │    Servlet Container  │
              └───────────┬───────────┘
                          │
                    Filter Chain
                          │
              ┌───────────┴───────────┐
              │                       │
        Authentication          Authorization
              │                       │
              └───────────┬───────────┘
                          ↓
                       Servlet
                          ↓
                       Service
                          ↓
                         DAO
                          ↓
                       Database
```

**Day 4 is the foundation for the security portions of the rest of your J2EE journey.** Day 5 can build on this with **Filters, Listeners, security infrastructure, request lifecycle hooks, advanced filter chains, and enterprise cross-cutting concerns**, while avoiding repetition of the basics already covered on Day 3.
