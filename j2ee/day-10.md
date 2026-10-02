# J2EE 30-Day Mastery — Day 10

## JPA Fundamentals — ORM, EntityManager & Persistence Context

**Date:** October 12, 2026
**Goal:** Understand JPA deeply enough to build enterprise applications without treating Hibernate as “magic.”

Today is one of the most important days in the roadmap because **JPA becomes the bridge between Java objects and relational databases**.

---

# 1. What is JPA?

**JPA = Java Persistence API.**

Modern terminology:

> JPA is now part of Jakarta Persistence under Jakarta EE.

JPA is a **specification**, not an implementation.

It defines standard APIs and annotations for:

* Mapping Java classes to database tables
* Persisting objects
* Finding entities
* Updating entities
* Deleting entities
* Relationships
* Queries
* Transactions
* Persistence context
* Entity lifecycle

Popular implementations include:

* Hibernate
* EclipseLink
* OpenJPA

Think:

```text
JPA
 │
 ├── Specification
 │
 ├── EntityManager
 │
 ├── @Entity
 │
 ├── JPQL
 │
 └── Persistence Context
        │
        ▼
   Hibernate
        │
        ▼
     JDBC
        │
        ▼
    Database
```

---

# 2. JDBC vs JPA

Yesterday you learned JDBC.

### JDBC

You manually write:

```java
String sql =
    "SELECT id, name, email FROM employee WHERE id = ?";

PreparedStatement ps = connection.prepareStatement(sql);
ps.setLong(1, 10L);

ResultSet rs = ps.executeQuery();

Employee employee = new Employee();

if (rs.next()) {
    employee.setId(rs.getLong("id"));
    employee.setName(rs.getString("name"));
    employee.setEmail(rs.getString("email"));
}
```

You are responsible for:

* SQL
* Connection
* PreparedStatement
* ResultSet
* Mapping rows to objects

---

### JPA

You can write:

```java
Employee employee =
        entityManager.find(Employee.class, 10L);
```

JPA/Hibernate handles much of the persistence machinery.

Conceptually:

```text
Java Object
     ↓
    JPA
     ↓
 Hibernate
     ↓
   JDBC
     ↓
 Database
```

---

# 3. What is ORM?

**ORM = Object-Relational Mapping.**

Relational databases deal with:

```text
Tables
Rows
Columns
Primary Keys
Foreign Keys
```

Java deals with:

```text
Classes
Objects
Fields
References
Inheritance
```

ORM maps between these two worlds.

Example:

```java
public class Employee {
    private Long id;
    private String name;
    private String email;
}
```

Database:

```text
employee
--------------------------------
id | name | email
--------------------------------
1  | John | john@example.com
```

Mapping:

```text
Employee.id       → employee.id
Employee.name     → employee.name
Employee.email    → employee.email
```

---

# 4. Why ORM?

Without ORM:

```text
SQL
 ↓
ResultSet
 ↓
Manual mapping
 ↓
Java Object
```

With ORM:

```text
Java Entity
 ↓
JPA
 ↓
ORM implementation
 ↓
SQL
 ↓
Database
```

Benefits:

* Less boilerplate
* Object-oriented programming model
* Automatic mapping
* Relationship handling
* Entity lifecycle management
* Dirty checking
* First-level cache
* JPQL
* Transaction integration

But ORM does **not** eliminate SQL knowledge.

A strong enterprise developer still understands:

```text
SQL
+
Indexes
+
Transactions
+
Isolation
+
Execution plans
+
JPA
+
Hibernate
```

---

# 5. Your First JPA Entity

Modern Jakarta Persistence:

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;

@Entity
public class Employee {

    @Id
    private Long id;

    private String name;

    private String email;

    // getters/setters
}
```

The important annotation:

```java
@Entity
```

means:

> This Java class is a persistent entity managed by JPA.

---

# 6. `@Entity`

Example:

```java
@Entity
public class Employee {
}
```

Typically:

```text
Employee class
       ↓
employee table
```

The exact table name can be controlled:

```java
@Entity
@Table(name = "employees")
public class Employee {
}
```

Now:

```text
Employee → employees
```

---

# 7. Entity Requirements

A typical JPA entity should have:

### 1. `@Entity`

```java
@Entity
public class Employee {
}
```

### 2. Primary key

```java
@Id
private Long id;
```

### 3. No-argument constructor

```java
protected Employee() {
}
```

or:

```java
public Employee() {
}
```

### 4. Suitable non-final class/method structure

Avoid designing entities as immutable/final in ways that conflict with the persistence provider's requirements.

---

# 8. Primary Key

Use:

```java
@Id
private Long id;
```

Example:

```java
@Entity
public class Employee {

    @Id
    private Long id;

    private String name;
}
```

---

# 9. Generated ID

Instead of manually assigning IDs:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

Imports:

```java
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
```

Example:

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String email;
}
```

---

# 10. Generation Strategies

Common strategies:

```java
GenerationType.IDENTITY
GenerationType.SEQUENCE
GenerationType.TABLE
GenerationType.AUTO
```

### IDENTITY

Database generates the ID.

Common example:

```sql
AUTO_INCREMENT
```

### SEQUENCE

Database sequence generates IDs.

Example:

```sql
employee_seq
```

### TABLE

A table is used to manage generated IDs.

### AUTO

Provider chooses an appropriate strategy.

---

# 11. Mapping Columns

You can customize a column:

```java
@Column(name = "employee_name")
private String name;
```

More constraints:

```java
@Column(
    name = "email",
    nullable = false,
    unique = true,
    length = 200
)
private String email;
```

---

# 12. Complete Entity

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true, length = 200)
    private String email;

    protected Employee() {
    }

    public Employee(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
```

---

# 13. What is EntityManager?

One of the most important JPA classes:

```java
EntityManager
```

It manages entity persistence.

Example:

```java
Employee employee = new Employee(
        "John",
        "john@example.com"
);

entityManager.persist(employee);
```

Conceptually:

```text
EntityManager
     │
     ├── persist()
     ├── find()
     ├── merge()
     ├── remove()
     ├── flush()
     ├── createQuery()
     └── createNativeQuery()
```

---

# 14. `persist()`

Used to make a new entity persistent.

```java
Employee employee =
        new Employee("John", "john@example.com");

entityManager.persist(employee);
```

Conceptually:

```text
new Employee()
      ↓
   persist()
      ↓
Persistence Context
      ↓
 INSERT
```

Important:

`persist()` does not necessarily mean the SQL `INSERT` executes immediately.

The SQL may execute when the persistence context is flushed.

---

# 15. `find()`

Retrieve an entity by primary key:

```java
Employee employee =
        entityManager.find(Employee.class, 1L);
```

Conceptually:

```text
find(Employee.class, 1L)
             ↓
       Persistence Context
             ↓
        Database if needed
```

---

# 16. `remove()`

Delete an entity:

```java
Employee employee =
        entityManager.find(Employee.class, 1L);

entityManager.remove(employee);
```

Conceptually:

```text
Managed Entity
      ↓
   remove()
      ↓
 DELETE
```

---

# 17. `merge()`

This is one of the most misunderstood JPA operations.

Suppose:

```java
Employee detachedEmployee = ...;
```

You can do:

```java
Employee managedEmployee =
        entityManager.merge(detachedEmployee);
```

Important:

> `merge()` returns a managed instance.

It does **not** simply turn the supplied object into the managed object.

Example:

```java
Employee managed =
        entityManager.merge(detached);
```

You should generally continue working with:

```java
managed
```

rather than assuming:

```java
detached
```

is now managed.

---

# 18. `flush()`

```java
entityManager.flush();
```

Flush synchronizes the persistence context with the database.

Conceptually:

```text
Persistence Context
        ↓
      flush()
        ↓
 SQL statements
        ↓
    Database
```

Flush is not the same thing as commit.

```text
flush ≠ commit
```

A transaction can flush SQL and later roll back.

---

# 19. Persistence Context

This is **the most important concept today**.

A persistence context is essentially a managed set of entity instances associated with an `EntityManager`.

Think:

```text
EntityManager
      │
      ▼
Persistence Context
      │
      ├── Employee #1
      ├── Employee #2
      └── Employee #3
```

JPA tracks managed entities.

This enables:

* Identity guarantee within the persistence context
* Dirty checking
* First-level cache
* Automatic synchronization

---

# 20. First-Level Cache

Suppose:

```java
Employee e1 =
    entityManager.find(Employee.class, 1L);

Employee e2 =
    entityManager.find(Employee.class, 1L);
```

Within the same persistence context, JPA can return the already-managed entity rather than querying the database again.

Conceptually:

```text
find(1)
  ↓
Database
  ↓
Employee #1
  ↓
Persistence Context

find(1)
  ↓
Persistence Context
  ↓
Employee #1
```

This is called the:

> First-level cache.

It is associated with the persistence context/EntityManager.

---

# 21. Identity Guarantee

Within one persistence context:

```java
Employee e1 =
    entityManager.find(Employee.class, 1L);

Employee e2 =
    entityManager.find(Employee.class, 1L);
```

Typically:

```java
e1 == e2
```

is true for the same entity identity in that persistence context.

This is a major reason the persistence context matters.

---

# 22. Dirty Checking

One of JPA's most powerful features.

Suppose:

```java
Employee employee =
    entityManager.find(Employee.class, 1L);

employee.setName("Pradeep");
```

You didn't write:

```java
UPDATE employees
SET name = ?
WHERE id = ?
```

JPA can detect the change.

At flush:

```text
Employee changed
      ↓
Dirty Checking
      ↓
UPDATE
      ↓
Database
```

This is called:

> Dirty checking.

---

# 23. Example of Dirty Checking

```java
@Transactional
public void updateEmployee(Long id) {

    Employee employee =
        entityManager.find(Employee.class, id);

    employee.setName("New Name");
}
```

You don't necessarily need:

```java
entityManager.merge(employee);
```

when `employee` is already managed.

At flush/commit, JPA can detect the modification.

---

# 24. Entity Lifecycle

An entity can move through several states:

```text
        new
         │
         │ persist()
         ▼
     MANAGED
      │   │
      │   │ remove()
      │   ▼
      │ REMOVED
      │
      │ detach()/clear()
      ▼
   DETACHED
      │
      │ merge()
      ▼
   MANAGED
```

Main states:

1. Transient
2. Managed
3. Detached
4. Removed

---

# 25. Transient Entity

Created using:

```java
Employee employee =
        new Employee();
```

but not yet associated with the persistence context.

```text
new Employee()
       ↓
TRANSIENT
```

---

# 26. Managed Entity

After:

```java
entityManager.persist(employee);
```

the entity becomes managed.

```text
persist()
   ↓
MANAGED
```

JPA tracks changes.

---

# 27. Detached Entity

An entity can become detached when:

```java
entityManager.detach(employee);
```

or:

```java
entityManager.clear();
```

or the persistence context ends.

Conceptually:

```text
MANAGED
   ↓
detach
   ↓
DETACHED
```

Changes to a detached entity are not automatically tracked by that persistence context.

---

# 28. Removed Entity

```java
entityManager.remove(employee);
```

The entity becomes marked for removal.

```text
MANAGED
   ↓
remove()
   ↓
REMOVED
```

The actual SQL `DELETE` is generally synchronized during flush.

---

# 29. `detach()`

```java
entityManager.detach(employee);
```

Removes that entity from the persistence context.

---

# 30. `clear()`

```java
entityManager.clear();
```

Detaches all managed entities from that persistence context.

---

# 31. `contains()`

Check whether an entity is managed:

```java
boolean managed =
        entityManager.contains(employee);
```

---

# 32. `refresh()`

Reload state from the database:

```java
entityManager.refresh(employee);
```

Conceptually:

```text
Java entity
    ↓
refresh()
    ↓
Database
    ↓
Current DB state
```

Be careful: local changes can be overwritten.

---

# 33. `getReference()`

```java
Employee employee =
    entityManager.getReference(Employee.class, 1L);
```

It can provide a reference without immediately loading the full entity.

This is useful in certain relationships and performance-sensitive operations.

---

# 34. JPA CRUD Summary

| Operation               | JPA           |
| ----------------------- | ------------- |
| Create                  | `persist()`   |
| Read by ID              | `find()`      |
| Update managed entity   | Modify object |
| Reattach detached state | `merge()`     |
| Delete                  | `remove()`    |
| Flush                   | `flush()`     |
| Detach one              | `detach()`    |
| Detach all              | `clear()`     |
| Reload                  | `refresh()`   |
| Check managed           | `contains()`  |

---

# 35. JPQL

JPA provides:

> **JPQL = Jakarta Persistence Query Language**

It operates on **entities and their fields**, not directly on tables and columns.

SQL:

```sql
SELECT *
FROM employees
WHERE email = ?
```

JPQL:

```java
SELECT e
FROM Employee e
WHERE e.email = :email
```

Notice:

```text
SQL → table
JPQL → entity
```

---

# 36. JPQL Example

```java
TypedQuery<Employee> query =
    entityManager.createQuery(
        "SELECT e FROM Employee e WHERE e.name = :name",
        Employee.class
    );

query.setParameter("name", "John");

List<Employee> employees =
    query.getResultList();
```

---

# 37. JPQL Uses Entity Names

Suppose:

```java
@Entity
@Table(name = "employees")
public class Employee {
}
```

JPQL:

```sql
SELECT e FROM Employee e
```

Not:

```sql
SELECT e FROM employees e
```

because JPQL works with the entity model.

---

# 38. Named Parameters

Prefer:

```java
WHERE e.email = :email
```

then:

```java
query.setParameter("email", email);
```

rather than constructing query strings with user input.

---

# 39. Query Methods

Common:

```java
getResultList()
```

returns multiple results.

```java
getSingleResult()
```

returns one result but requires careful handling when zero or multiple results occur.

```java
getResultStream()
```

can provide a stream-oriented result API where supported.

---

# 40. JPQL Filtering

```java
SELECT e
FROM Employee e
WHERE e.salary > :salary
```

Java:

```java
query.setParameter("salary", 50000);
```

---

# 41. Sorting

```java
SELECT e
FROM Employee e
ORDER BY e.name ASC
```

---

# 42. Pagination

Conceptually:

```java
query.setFirstResult(0);
query.setMaxResults(20);
```

This gives:

```text
page size = 20
offset = 0
```

For the next page:

```java
query.setFirstResult(20);
query.setMaxResults(20);
```

---

# 43. Named Queries

You can define:

```java
@NamedQuery(
    name = "Employee.findByEmail",
    query = "SELECT e FROM Employee e WHERE e.email = :email"
)
```

Then:

```java
TypedQuery<Employee> query =
    entityManager.createNamedQuery(
        "Employee.findByEmail",
        Employee.class
    );
```

---

# 44. Persistence Unit

A JPA application needs configuration describing persistence.

Traditionally this is configured using:

```text
META-INF/persistence.xml
```

Example:

```xml
<persistence
    xmlns="https://jakarta.ee/xml/ns/persistence"
    version="3.1">

    <persistence-unit
        name="employeePU">

        <class>com.example.entity.Employee</class>

    </persistence-unit>

</persistence>
```

In Jakarta EE environments, the application server can manage much of the infrastructure.

---

# 45. EntityManagerFactory

Another important object:

```java
EntityManagerFactory
```

Conceptually:

```text
Application
    ↓
EntityManagerFactory
    ↓
EntityManager
    ↓
Persistence Context
```

### EntityManagerFactory

Usually:

* Expensive to create
* Long-lived
* Application-level resource

### EntityManager

Usually:

* Shorter-lived
* Represents a persistence context
* Used for entity operations

---

# 46. EntityManager vs EntityManagerFactory

| EntityManagerFactory     | EntityManager                    |
| ------------------------ | -------------------------------- |
| Heavyweight              | Lightweight                      |
| Usually application-wide | Usually operation/request scoped |
| Creates EntityManagers   | Performs persistence operations  |
| Thread-safe by contract  | Not thread-safe                  |
| Long-lived               | Short-lived                      |

Never treat one `EntityManager` as a globally shared application singleton.

---

# 47. JPA + Transactions

JPA operations normally participate in transactions for database modifications.

Conceptually:

```text
BEGIN
  ↓
persist()
  ↓
update()
  ↓
remove()
  ↓
flush
  ↓
COMMIT
```

If something fails:

```text
BEGIN
  ↓
Operations
  ↓
ERROR
  ↓
ROLLBACK
```

Transaction management becomes a major topic on **Day 13**.

---

# 48. Complete JPA Service Example

```java
public class EmployeeService {

    private final EntityManager entityManager;

    public EmployeeService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    public void create(Employee employee) {
        entityManager.persist(employee);
    }

    public Employee find(Long id) {
        return entityManager.find(Employee.class, id);
    }

    public void delete(Long id) {

        Employee employee =
                entityManager.find(Employee.class, id);

        if (employee != null) {
            entityManager.remove(employee);
        }
    }
}
```

---

# 49. JPA Architecture

Understand this architecture:

```text
                Application
                     │
                     ▼
              EntityManager
                     │
                     ▼
            Persistence Context
                     │
                     ▼
              JPA Provider
              (Hibernate)
                     │
                     ▼
                   JDBC
                     │
                     ▼
                DataSource
                     │
                     ▼
                 Database
```

This is extremely important for interviews.

---

# 50. JPA vs Hibernate

This is a classic interview question.

### JPA

Specification.

### Hibernate

Implementation/provider.

Think:

```text
JPA = contract
Hibernate = implementation
```

For example:

```java
@Entity
@Id
@OneToMany
```

These are standardized Jakarta Persistence concepts.

Hibernate provides the runtime implementation.

---

# 51. JPA vs JDBC

| JDBC                     | JPA                         |
| ------------------------ | --------------------------- |
| Lower level              | Higher level                |
| SQL-centric              | Entity-centric              |
| Manual mapping           | ORM mapping                 |
| Manual resource handling | Provider manages much of it |
| Manual CRUD              | Entity operations           |
| ResultSet                | Entity objects              |
| More boilerplate         | Less boilerplate            |
| Fine SQL control         | Abstraction over SQL        |

Neither replaces the need to understand databases.

---

# 52. Important JPA Annotations

Start memorizing these:

```java
@Entity
@Table
@Id
@GeneratedValue
@Column
@Transient
@Enumerated
@Temporal // mainly relevant to older date/time mappings
@Version
@Embedded
@Embeddable
@OneToOne
@OneToMany
@ManyToOne
@ManyToMany
@JoinColumn
@JoinTable
@NamedQuery
```

Relationships will be covered deeply on **Day 11**.

---

# 53. `@Transient`

Suppose:

```java
@Entity
public class Employee {

    private String firstName;

    private String lastName;

    @Transient
    private String displayName;
}
```

`displayName` is not persisted as a database column by JPA.

---

# 54. `@Enumerated`

Java:

```java
public enum EmployeeStatus {
    ACTIVE,
    INACTIVE
}
```

Entity:

```java
@Enumerated(EnumType.STRING)
private EmployeeStatus status;
```

Prefer:

```java
EnumType.STRING
```

over ordinal persistence in most production designs because enum ordering can change.

---

# 55. Optimistic Locking Preview

JPA supports optimistic locking with:

```java
@Version
private Long version;
```

Conceptually:

```text
Employee
-------------------------
id
name
email
version
-------------------------
```

This helps detect conflicting concurrent updates.

We will study concurrency/transactions more deeply later.

---

# 56. The Most Important Concept: Managed Entity

Consider:

```java
Employee employee =
    entityManager.find(Employee.class, 1L);

employee.setEmail("new@example.com");
```

If `employee` is managed:

```text
Java object
     ↓
Persistence Context
     ↓
Dirty checking
     ↓
SQL UPDATE
```

This is the fundamental JPA mental model.

---

# 57. What Actually Happens?

Suppose:

```java
@Transactional
public void updateEmployee(Long id) {

    Employee employee =
        entityManager.find(Employee.class, id);

    employee.setName("Alex");
}
```

Conceptually:

### Step 1

Transaction starts.

### Step 2

JPA executes:

```sql
SELECT ...
FROM employees
WHERE id = ?
```

### Step 3

Entity enters persistence context.

### Step 4

Application changes:

```java
employee.setName("Alex");
```

### Step 5

JPA tracks the change.

### Step 6

Flush occurs.

### Step 7

Hibernate generates approximately:

```sql
UPDATE employees
SET name = ?
WHERE id = ?
```

### Step 8

Transaction commits.

This flow is extremely important for enterprise development.

---

# 58. Why You Should Not Call `merge()` Everywhere

Bad habit:

```java
Employee employee =
    entityManager.find(Employee.class, id);

employee.setName("Alex");

entityManager.merge(employee);
```

If `employee` is already managed, `merge()` is unnecessary.

Better:

```java
Employee employee =
    entityManager.find(Employee.class, id);

employee.setName("Alex");
```

Dirty checking handles the update.

---

# 59. Common JPA Performance Problem: N+1

Suppose you load:

```text
100 employees
```

and then lazily load a related entity for each employee.

You can accidentally produce:

```text
1 query
+
100 queries
=
101 queries
```

This is the famous:

> N+1 query problem.

We'll go much deeper into this with relationships and fetch strategies.

---

# 60. JPA Doesn't Mean "No SQL"

You should still understand:

```text
SELECT
JOIN
INDEX
GROUP BY
ORDER BY
WHERE
INSERT
UPDATE
DELETE
transactions
locking
execution plans
```

Because eventually you need to understand the SQL generated by your ORM.

A solutions architect must be able to reason about:

```text
Java code
   ↓
JPA/Hibernate
   ↓
Generated SQL
   ↓
Database execution
   ↓
Performance
```

---

# 61. Production Package Structure

For today's Employee application:

```text
src/main/java/
└── com.example.employee
    ├── entity
    │   └── Employee.java
    │
    ├── repository
    │   └── EmployeeRepository.java
    │
    ├── service
    │   └── EmployeeService.java
    │
    ├── controller
    │   └── EmployeeServlet.java
    │
    └── config
        └── PersistenceConfig.java

src/main/resources/
└── META-INF/
    └── persistence.xml
```

Web layer:

```text
src/main/webapp/
└── WEB-INF/
    └── views/
        ├── employees.jsp
        ├── employee-form.jsp
        └── employee-details.jsp
```

---

# 62. JPA Repository Abstraction

You can introduce:

```java
public interface EmployeeRepository {

    void save(Employee employee);

    Employee findById(Long id);

    void delete(Employee employee);

    List<Employee> findAll();
}
```

Implementation:

```java
public class JpaEmployeeRepository
        implements EmployeeRepository {

    private final EntityManager entityManager;

    public JpaEmployeeRepository(
            EntityManager entityManager) {

        this.entityManager = entityManager;
    }

    @Override
    public void save(Employee employee) {
        entityManager.persist(employee);
    }

    @Override
    public Employee findById(Long id) {
        return entityManager.find(Employee.class, id);
    }

    @Override
    public void delete(Employee employee) {
        entityManager.remove(employee);
    }
}
```

This keeps:

```text
Controller
     ↓
Service
     ↓
Repository
     ↓
JPA
     ↓
Database
```

---

# 63. JDBC vs JPA Architecture

### JDBC

```text
Servlet
   ↓
Service
   ↓
DAO
   ↓
JDBC
   ↓
DataSource
   ↓
Database
```

### JPA

```text
Servlet
   ↓
Service
   ↓
Repository
   ↓
EntityManager
   ↓
JPA Provider
   ↓
JDBC
   ↓
Database
```

Notice something important:

> JPA does not eliminate JDBC internally.

A JPA provider typically uses JDBC to communicate with relational databases.

---

# 64. Common JPA Mistakes

### Mistake 1

Putting database logic in JSP.

❌

### Mistake 2

Sharing one `EntityManager` across all threads.

❌

### Mistake 3

Calling `merge()` unnecessarily.

❌

### Mistake 4

Ignoring transactions.

❌

### Mistake 5

Ignoring generated SQL.

❌

### Mistake 6

Loading huge datasets into memory.

❌

### Mistake 7

Using eager relationships everywhere.

❌

### Mistake 8

Ignoring N+1 queries.

❌

### Mistake 9

Using `EnumType.ORDINAL` without understanding its schema risks.

❌

### Mistake 10

Assuming JPA automatically makes inefficient queries efficient.

❌

---

# 65. Day 10 Hands-On Exercises

## Exercise 1 — Basic Entity

Create:

```java
Employee
```

with:

```text
id
name
email
salary
department
```

Map it to:

```text
employees
```

---

## Exercise 2 — Generated ID

Use:

```java
@GeneratedValue
```

and experiment with different strategies supported by your database/provider.

---

## Exercise 3 — CRUD

Implement:

```text
createEmployee()
findEmployee()
updateEmployee()
deleteEmployee()
```

using `EntityManager`.

---

## Exercise 4 — Persistence States

Demonstrate:

```text
Transient
Managed
Detached
Removed
```

using:

```java
persist()
detach()
clear()
remove()
merge()
```

---

## Exercise 5 — Dirty Checking

Write:

```java
@Transactional
public void update(Long id)
```

and modify a managed entity without calling `merge()`.

Observe the generated SQL.

---

## Exercise 6 — First-Level Cache

Execute:

```java
Employee e1 =
    entityManager.find(Employee.class, 1L);

Employee e2 =
    entityManager.find(Employee.class, 1L);
```

Observe the SQL generated.

---

## Exercise 7 — JPQL

Find all employees:

```sql
SELECT e FROM Employee e
```

---

## Exercise 8

Find employees whose salary is greater than:

```text
50000
```

---

## Exercise 9

Find employee by email.

---

## Exercise 10

Sort employees by salary:

```text
DESC
```

---

## Exercise 11

Implement pagination:

```text
page size = 20
```

---

## Exercise 12

Create a named query:

```text
Employee.findByEmail
```

---

## Exercise 13

Use:

```java
@Transient
```

for a calculated property.

---

## Exercise 14

Use:

```java
@Enumerated(EnumType.STRING)
```

for employee status.

---

## Exercise 15 — Production Exercise

Build:

```text
Servlet
   ↓
Service
   ↓
JPA Repository
   ↓
EntityManager
   ↓
Database
```

for your Employee Management System.

---

# 66. Day 10 Mini Project

## Employee Management System — JPA Edition

Build:

```text
Employee Management System
```

Features:

```text
Create Employee
List Employees
Get Employee
Update Employee
Delete Employee
Search Employee
Pagination
```

Architecture:

```text
Browser
   ↓
EmployeeServlet
   ↓
EmployeeService
   ↓
EmployeeRepository
   ↓
EntityManager
   ↓
Hibernate/JPA
   ↓
JDBC
   ↓
MySQL/PostgreSQL
```

Entity:

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String email;

    private BigDecimal salary;

    protected Employee() {
    }

    // constructors/getters/setters
}
```

---

# 67. Interview Questions for Day 10

You should be able to answer these without notes.

### Beginner

1. What is JPA?
2. Is JPA an implementation?
3. What is ORM?
4. What is Hibernate?
5. JPA vs Hibernate?
6. JPA vs JDBC?
7. What is an entity?
8. What does `@Entity` do?
9. What does `@Id` do?
10. What does `@GeneratedValue` do?

### Intermediate

11. What is EntityManager?
12. What is EntityManagerFactory?
13. What is persistence context?
14. What is first-level cache?
15. What is dirty checking?
16. What is `persist()`?
17. What is `find()`?
18. What is `merge()`?
19. What is `remove()`?
20. What is `flush()`?

### Advanced

21. Explain entity lifecycle.
22. Transient vs managed vs detached?
23. What happens during `persist()`?
24. What happens during `find()`?
25. What happens during `merge()`?
26. Why does `merge()` return an object?
27. Why shouldn't EntityManager be shared across threads?
28. What is JPQL?
29. JPQL vs SQL?
30. What is a persistence unit?
31. What is dirty checking internally trying to achieve?
32. Flush vs commit?
33. What is first-level cache?
34. What causes N+1 queries?
35. How does JPA interact with JDBC?
36. Why is `EnumType.STRING` generally safer than ordinal persistence?
37. What is `@Version`?
38. What is a detached entity?
39. What does `clear()` do?
40. What does `refresh()` do?

---

# 68. Critical Interview Question

### What happens when you modify a managed entity?

Answer:

```text
Entity loaded
      ↓
Entity becomes managed
      ↓
Application modifies entity
      ↓
Persistence context tracks it
      ↓
Dirty checking
      ↓
Flush
      ↓
UPDATE SQL
      ↓
Transaction commit
```

Example:

```java
@Transactional
public void changeName(Long id) {

    Employee employee =
        entityManager.find(Employee.class, id);

    employee.setName("Alex");
}
```

No explicit update statement is required.

---

# 69. Another Critical Interview Question

### Is JPA a replacement for JDBC?

Not exactly.

A better answer:

> JPA provides a higher-level persistence abstraction based on ORM. JPA providers such as Hibernate commonly use JDBC underneath to communicate with relational databases. Therefore, JPA reduces application-level JDBC boilerplate but does not remove the need to understand JDBC, SQL, transactions, and database performance.

---

# 70. Your Day 10 Mental Model

Memorize this:

```text
Java Object
     ↓
@Entity
     ↓
EntityManager
     ↓
Persistence Context
     ↓
JPA Provider
     ↓
Hibernate
     ↓
JDBC
     ↓
SQL
     ↓
Database
```

And:

```text
new Entity()
     ↓
 TRANSIENT
     ↓
persist()
     ↓
 MANAGED
     ↓
modify object
     ↓
DIRTY CHECKING
     ↓
flush()
     ↓
SQL
     ↓
COMMIT
```

---

# 71. Day 10 Mastery Checklist

Before moving to Day 11, you should understand:

* [ ] JPA
* [ ] Jakarta Persistence terminology
* [ ] ORM
* [ ] JPA vs Hibernate
* [ ] JPA vs JDBC
* [ ] `@Entity`
* [ ] `@Table`
* [ ] `@Id`
* [ ] `@GeneratedValue`
* [ ] `@Column`
* [ ] `@Transient`
* [ ] `@Enumerated`
* [ ] EntityManager
* [ ] EntityManagerFactory
* [ ] Persistence Context
* [ ] First-level cache
* [ ] Dirty checking
* [ ] `persist()`
* [ ] `find()`
* [ ] `merge()`
* [ ] `remove()`
* [ ] `flush()`
* [ ] `detach()`
* [ ] `clear()`
* [ ] `refresh()`
* [ ] Entity lifecycle
* [ ] JPQL
* [ ] Named queries
* [ ] Pagination
* [ ] Persistence unit
* [ ] Transactions
* [ ] Generated SQL
* [ ] N+1 awareness

## ⭐ Day 10 Golden Rule

> **JPA is not primarily about writing fewer SQL statements. It is about managing the lifecycle and persistence of Java entities through a persistence context.**

Your next step is **Day 11 — JPA Relationships**, where we'll go deeply into **`@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`, foreign keys, `@JoinColumn`, owning/inverse sides, cascade types, orphan removal, lazy/eager loading, and real enterprise entity modeling.**
