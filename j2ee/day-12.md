# J2EE 30-Day Mastery — Day 12

## Advanced JPA — Queries, Inheritance, Embedded Types, Locking & Persistence Techniques

**Date:** October 14, 2026

**Day 12 objective:** Move from “I can map entities” to **“I can design and query a production-grade JPA persistence layer.”**

By the end of today, you should understand:

```text
JPA
├── Entity lifecycle
├── Embedded objects
├── Inheritance
├── Composite keys
├── JPQL joins
├── Parameters
├── Aggregations
├── Native SQL
├── Criteria API
├── Bulk operations
├── Optimistic locking
├── Pessimistic locking
├── Entity graphs
├── Pagination
└── Persistence-context behavior
```

---

# 1. Where Day 12 Fits

You now have:

```text
Day 9  → JDBC
Day 10 → JPA Fundamentals
Day 11 → JPA Relationships
Day 12 → Advanced JPA
Day 13 → Transactions
Day 14 → Employee Management System
```

Today's architecture:

```text
Servlet
   ↓
Service
   ↓
Repository
   ↓
EntityManager
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

# 2. Entity Lifecycle — Deep Dive

Yesterday you learned:

```text
TRANSIENT
MANAGED
DETACHED
REMOVED
```

Let's understand exactly what they mean.

---

## Transient

```java
Employee employee = new Employee();
```

At this point:

```text
Java Object
    ↓
TRANSIENT
```

JPA isn't managing it.

---

## Managed

```java
entityManager.persist(employee);
```

Now:

```text
Employee
   ↓
Persistence Context
   ↓
MANAGED
```

Changes are tracked.

---

## Detached

```java
entityManager.detach(employee);
```

Now:

```text
Persistence Context
        X
        ↓
Employee
        ↓
DETACHED
```

Changes are no longer automatically tracked by that persistence context.

---

## Removed

```java
entityManager.remove(employee);
```

Entity is marked for deletion.

```text
MANAGED
   ↓
remove()
   ↓
REMOVED
```

---

# 3. `merge()` — Deep Understanding

This is one of the most frequently misunderstood operations.

Suppose:

```java
Employee detached = ...;
```

Then:

```java
Employee managed =
        entityManager.merge(detached);
```

Important:

```text
detached != managed
```

Conceptually:

```text
Detached Employee
       │
       │ merge()
       ▼
Managed copy
```

The returned object is the one you should continue using.

---

# 4. `persist()` vs `merge()`

| `persist()`                                             | `merge()`                        |
| ------------------------------------------------------- | -------------------------------- |
| Usually used for new entity                             | Used for detached/entity state   |
| Makes supplied entity managed                           | Returns managed instance         |
| Does not create a separate managed copy in the same way | Copies state into managed entity |
| Common for INSERT                                       | Common for detached updates      |

Example:

```java
Employee employee = new Employee();

entityManager.persist(employee);
```

versus:

```java
Employee managed =
        entityManager.merge(detachedEmployee);
```

---

# 5. Persistence Context Operations

Know these:

```java
persist()
find()
merge()
remove()
detach()
clear()
refresh()
contains()
flush()
```

Example:

```java
entityManager.contains(employee);
```

returns whether the entity is currently managed by that persistence context.

---

# 6. `flush()` vs `clear()`

### flush()

Synchronizes pending persistence-context changes with the database.

```java
entityManager.flush();
```

### clear()

Detaches all managed entities.

```java
entityManager.clear();
```

They are completely different.

```text
flush()
  ↓
Synchronize with DB

clear()
  ↓
Empty persistence context
```

---

# 7. `refresh()`

Suppose:

```java
Employee employee =
    entityManager.find(Employee.class, 1L);
```

Then:

```java
entityManager.refresh(employee);
```

asks the persistence provider to refresh the entity's state from the database.

Useful when database-side changes need to be reflected in the managed object.

Be careful:

> Local unflushed changes can be overwritten.

---

# 8. Embedded Objects

Sometimes a class isn't really an independent entity.

Example:

```text
Employee
 ├── id
 ├── name
 └── Address
      ├── street
      ├── city
      └── zipCode
```

You may want `Address` to be part of `Employee`.

Use:

```java
@Embeddable
```

---

# 9. `@Embeddable`

```java
@Embeddable
public class Address {

    private String street;

    private String city;

    private String zipCode;
}
```

Then:

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Embedded
    private Address address;
}
```

---

# 10. Database Representation

You might have:

```text
employees
------------------------------------------------
id | name | street | city | zip_code
------------------------------------------------
```

There doesn't have to be a separate `address` table.

Conceptually:

```text
Employee
   +
Embedded Address
   ↓
One database row
```

---

# 11. Entity vs Embeddable

### Entity

Has:

```text
identity
lifecycle
persistence independently
```

### Embeddable

Usually:

```text
part of another entity
no independent entity identity
```

Example:

```text
Employee → Entity

Address → Embeddable
```

---

# 12. Attribute Overrides

Suppose:

```java
@Embedded
private Address homeAddress;

@Embedded
private Address officeAddress;
```

Both contain:

```text
city
street
zipCode
```

You need different database columns.

Use:

```java
@AttributeOverrides({
    @AttributeOverride(
        name = "street",
        column = @Column(name = "home_street")
    ),
    @AttributeOverride(
        name = "city",
        column = @Column(name = "home_city")
    )
})
private Address homeAddress;
```

This becomes useful in real domain models.

---

# 13. JPA Inheritance

Suppose:

```text
Employee
   │
   ├── FullTimeEmployee
   └── ContractEmployee
```

You can represent inheritance using JPA.

Major strategies:

```text
SINGLE_TABLE
JOINED
TABLE_PER_CLASS
```

---

# 14. `SINGLE_TABLE`

One table contains all subclasses.

```text
employees
------------------------------------------------
id | name | type | salary | contract_end_date
------------------------------------------------
```

Example:

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "employee_type")
public abstract class Employee {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
}
```

Subclass:

```java
@Entity
public class FullTimeEmployee extends Employee {

    private BigDecimal salary;
}
```

Another:

```java
@Entity
public class ContractEmployee extends Employee {

    private LocalDate contractEndDate;
}
```

---

# 15. Advantages of SINGLE_TABLE

Pros:

* Simple querying
* No joins for polymorphic queries
* Good read performance in many cases

Cons:

* Many nullable columns
* Large table
* Schema can become awkward

---

# 16. `JOINED`

Parent table:

```text
employees
-----------
id
name
```

Full-time table:

```text
full_time_employees
-------------------
id
salary
```

Contract table:

```text
contract_employees
------------------
id
contract_end_date
```

Relationship:

```text
employees
   │
   ├── full_time_employees
   │
   └── contract_employees
```

Java:

```java
@Inheritance(strategy = InheritanceType.JOINED)
```

---

# 17. Advantages of JOINED

Pros:

* Normalized structure
* Less duplicated data
* Good relational modeling

Cons:

* Polymorphic queries may require joins
* More complex schema

---

# 18. `TABLE_PER_CLASS`

Each concrete subclass gets its own table.

Example:

```text
full_time_employees
-------------------
id
name
salary
```

```text
contract_employees
------------------
id
name
contract_end_date
```

No shared parent table for persistent state.

Potential drawback:

> Polymorphic queries can become expensive/complex because data is spread across tables.

---

# 19. Inheritance Strategy Summary

| Strategy        | Structure                |
| --------------- | ------------------------ |
| SINGLE_TABLE    | One table                |
| JOINED          | Parent + child tables    |
| TABLE_PER_CLASS | Table per concrete class |

You should be able to explain the trade-offs rather than memorizing one universally “best” strategy.

---

# 20. Composite Primary Keys

Sometimes one column isn't enough.

Example:

```text
OrderItem
------------------
order_id
product_id
quantity
```

Perhaps:

```text
(order_id, product_id)
```

forms the identity.

JPA supports composite identifiers.

Two common approaches:

```text
@EmbeddedId
@IdClass
```

---

# 21. `@EmbeddedId`

Key class:

```java
@Embeddable
public class OrderItemId implements Serializable {

    private Long orderId;

    private Long productId;

    // equals and hashCode
}
```

Entity:

```java
@Entity
public class OrderItem {

    @EmbeddedId
    private OrderItemId id;

    private Integer quantity;
}
```

---

# 22. Why `equals()` and `hashCode()` Matter

For identifier classes, correct:

```java
equals()
hashCode()
```

implementation is important.

Especially when used with:

```text
HashSet
HashMap
Persistence operations
Entity identity logic
```

Composite-key design deserves careful testing.

---

# 23. JPQL — Advanced Queries

Yesterday:

```java
SELECT e
FROM Employee e
WHERE e.name = :name
```

Today we'll go further.

---

# 24. JPQL Join

Suppose:

```text
Employee → Department
```

JPQL:

```sql
SELECT e
FROM Employee e
JOIN e.department d
WHERE d.name = :departmentName
```

This uses the Java relationship:

```text
e.department
```

rather than directly writing:

```text
employees.department_id
```

---

# 25. JOIN FETCH

Important for solving certain N+1 scenarios.

```sql
SELECT e
FROM Employee e
JOIN FETCH e.department
```

Conceptually:

```text
Employee
   +
Department
```

are retrieved as part of the query.

This lets you explicitly control what data a particular use case needs.

---

# 26. LEFT JOIN

Suppose some employees don't have a department.

Use:

```sql
SELECT e
FROM Employee e
LEFT JOIN e.department d
```

This can retain employees even when no department relationship exists.

---

# 27. JOIN vs LEFT JOIN

```text
JOIN
```

typically returns rows where the relationship matches.

```text
LEFT JOIN
```

retains the left-side entity even if the relationship is absent.

Same fundamental relational concepts as SQL, expressed through JPQL's entity model.

---

# 28. Aggregation

JPQL supports:

```text
COUNT
SUM
AVG
MIN
MAX
```

Example:

```sql
SELECT COUNT(e)
FROM Employee e
```

---

# 29. Group By

```sql
SELECT d.name, COUNT(e)
FROM Employee e
JOIN e.department d
GROUP BY d.name
```

Conceptually:

```text
IT       → 50
HR       → 20
Finance  → 30
```

---

# 30. Having

```sql
SELECT d.name, COUNT(e)
FROM Employee e
JOIN e.department d
GROUP BY d.name
HAVING COUNT(e) > 10
```

---

# 31. DTO Projection

You don't always need full entities.

Suppose API needs:

```text
Employee ID
Employee Name
Department Name
```

You can use a DTO projection.

Example DTO:

```java
public record EmployeeSummary(
    Long id,
    String name,
    String departmentName
) {
}
```

JPQL constructor expression:

```sql
SELECT new com.example.dto.EmployeeSummary(
    e.id,
    e.name,
    d.name
)
FROM Employee e
JOIN e.department d
```

This can avoid loading unnecessary parts of a large entity graph.

---

# 32. Why Projections Matter

Suppose `Employee` has:

```text
Employee
 ├── Department
 ├── Address
 ├── Projects
 ├── Skills
 ├── Manager
 └── AuditRecords
```

But your screen only needs:

```text
id
name
department
```

Don't automatically load everything.

Use a query designed for the use case.

---

# 33. Native Queries

JPA can execute native SQL.

Example:

```java
Query query =
    entityManager.createNativeQuery(
        "SELECT * FROM employees WHERE salary > ?"
    );

query.setParameter(1, 50000);
```

Native SQL is useful when:

* Database-specific features are required
* Complex SQL is easier to express directly
* Existing SQL must be reused
* Performance tuning requires precise SQL

But it reduces database portability and moves you outside some of JPQL's abstraction.

---

# 34. JPQL vs Native SQL

| JPQL                      | Native SQL                      |
| ------------------------- | ------------------------------- |
| Entity-oriented           | Table-oriented                  |
| More portable             | Database-specific possibilities |
| Uses entity relationships | Uses tables/columns             |
| Provider generates SQL    | You write SQL                   |
| Easier domain abstraction | Maximum SQL control             |

---

# 35. Criteria API

JPA also provides a programmatic query API.

Conceptually:

```java
CriteriaBuilder cb =
        entityManager.getCriteriaBuilder();

CriteriaQuery<Employee> query =
        cb.createQuery(Employee.class);

Root<Employee> employee =
        query.from(Employee.class);

query.select(employee)
     .where(
         cb.equal(employee.get("name"), "John")
     );
```

---

# 36. Why Criteria API?

It can be useful when query conditions are dynamically assembled.

For example:

```text
Search employees by:
 ├── name?
 ├── email?
 ├── department?
 ├── minimum salary?
 ├── maximum salary?
 └── status?
```

Instead of concatenating SQL/JPQL strings, conditions can be built programmatically.

---

# 37. Dynamic Search

Imagine:

```text
GET /employees?
name=John&
department=IT&
minSalary=50000
```

You might dynamically create predicates:

```text
name condition
+
department condition
+
salary condition
```

This is a common enterprise use case for criteria-style APIs.

---

# 38. Pagination

Never casually retrieve millions of records.

Use:

```java
query.setFirstResult(offset);
query.setMaxResults(pageSize);
```

Example:

```java
query.setFirstResult(0);
query.setMaxResults(20);
```

Next page:

```java
query.setFirstResult(20);
query.setMaxResults(20);
```

---

# 39. Offset Pagination

Example:

```text
page = 3
size = 20

offset = 60
```

Conceptually:

```text
Database
 ↓
skip 60
 ↓
return 20
```

Works well for many normal administrative/listing use cases.

But for extremely large/high-churn datasets, offset pagination can become increasingly expensive.

---

# 40. Keyset/Cursor Pagination

Alternative:

```text
WHERE id > :lastSeenId
ORDER BY id
```

Then:

```text
lastSeenId = 100
```

next query:

```sql
WHERE id > 100
ORDER BY id
LIMIT 20
```

This can be more efficient for certain large ordered datasets.

---

# 41. Bulk UPDATE

JPQL:

```sql
UPDATE Employee e
SET e.salary = e.salary * 1.10
WHERE e.department.name = :department
```

Bulk operations are powerful.

But there is an important catch.

---

# 42. Bulk Operations and Persistence Context

Suppose persistence context contains:

```text
Employee #1
Employee #2
Employee #3
```

Then a bulk JPQL update executes directly against the database.

The persistence context may now contain stale entity state.

Conceptually:

```text
Persistence Context
      │
      │ stale
      ▼
Entity state

Database
      │
      │ updated
      ▼
New state
```

Therefore, bulk updates require careful synchronization/clearing strategy.

This is an advanced but important production concern.

---

# 43. Bulk DELETE

Example:

```sql
DELETE FROM Employee e
WHERE e.status = :status
```

Again:

> Bulk DML bypasses normal per-entity lifecycle behavior in ways you must account for.

Don't assume callbacks, dirty checking, and entity state handling behave exactly like calling `remove()` on each entity.

---

# 44. Optimistic Locking

This is critical in enterprise applications.

Suppose two users load:

```text
Employee #10
```

Both see:

```text
salary = 50000
version = 1
```

User A updates salary:

```text
55000
```

User B updates salary:

```text
60000
```

Without concurrency control, one update may overwrite another.

---

# 45. `@Version`

Add:

```java
@Version
private Long version;
```

Database:

```text
employees
---------------------------
id | salary | version
---------------------------
10 | 50000  | 1
```

Update:

```text
version 1 → version 2
```

The provider can use the version value to detect conflicting updates.

---

# 46. Optimistic Locking Flow

User A:

```text
read version = 1
```

User B:

```text
read version = 1
```

A updates:

```text
version 1 → 2
```

B attempts:

```text
UPDATE ... WHERE id = 10 AND version = 1
```

No matching current version remains.

JPA can detect the conflict and raise an optimistic-locking exception.

Conceptually:

```text
A → succeeds
B → conflict detected
```

---

# 47. When Optimistic Locking Is Useful

Good fit when:

```text
Reads >> writes
```

and conflicts are relatively uncommon.

Examples:

```text
Product catalog
Employee records
Customer profiles
Configuration
Business documents
```

---

# 48. Pessimistic Locking

Instead of detecting conflicts later, you can request database-level locking.

Example:

```java
entityManager.find(
    Employee.class,
    id,
    LockModeType.PESSIMISTIC_WRITE
);
```

Conceptually:

```text
Transaction A
    ↓
locks row
    ↓
Transaction B waits/fails depending on DB/lock settings
```

---

# 49. Optimistic vs Pessimistic

| Optimistic                         | Pessimistic                              |
| ---------------------------------- | ---------------------------------------- |
| Detect conflicts                   | Prevent/serialize conflicting access     |
| `@Version` commonly used           | Database locking                         |
| Good for lower conflict            | Useful when conflicts must be controlled |
| Less locking overhead              | Can increase lock contention             |
| Conflict may fail at commit/update | Transaction can block                    |

Neither is universally correct.

Choose based on business concurrency requirements and database behavior.

---

# 50. Lock Modes

Know the concepts:

```text
OPTIMISTIC
OPTIMISTIC_FORCE_INCREMENT

PESSIMISTIC_READ
PESSIMISTIC_WRITE
PESSIMISTIC_FORCE_INCREMENT
```

Exact locking behavior depends on provider/database support.

---

# 51. Entity Graphs

Entity graphs provide another way to control which relationships should be fetched for a particular operation.

Conceptually:

```text
Employee
 ├── Department ← fetch
 ├── Manager    ← fetch
 ├── Projects   ← don't fetch
 └── Skills     ← don't fetch
```

This is much more deliberate than making every relationship EAGER.

---

# 52. Why Entity Graphs Matter

Imagine two use cases.

### Employee List

Needs:

```text
id
name
department
```

### Employee Details

Needs:

```text
id
name
department
manager
projects
```

You shouldn't necessarily use one giant entity-loading strategy for both.

Entity graphs and tailored queries allow use-case-specific fetching.

---

# 53. `@NamedEntityGraph`

Conceptually:

```java
@NamedEntityGraph(
    name = "Employee.withDepartment",
    attributeNodes = {
        @NamedAttributeNode("department")
    }
)
@Entity
public class Employee {
}
```

The exact usage depends on the JPA environment/provider.

---

# 54. Query Design Is More Important Than Annotation Count

Bad mindset:

```text
"I know 50 JPA annotations."
```

Better mindset:

```text
Business requirement
       ↓
Required data
       ↓
Query
       ↓
Fetch plan
       ↓
Transaction
       ↓
Indexes
       ↓
Performance
```

This is how production persistence engineering works.

---

# 55. JPA and Database Indexes

Suppose:

```text
Employee.email
```

is searched frequently.

Database:

```sql
CREATE INDEX idx_employee_email
ON employees(email);
```

JPA mapping can also describe indexes at table level:

```java
@Table(
    name = "employees",
    indexes = {
        @Index(name = "idx_employee_email", columnList = "email")
    }
)
```

But production schema management should generally be handled deliberately through database migration tooling rather than relying on automatic schema generation.

---

# 56. Schema Generation

You may encounter:

```text
create
create-drop
update
validate
none
```

These are commonly associated with provider/schema-generation configuration.

For development, automatic schema generation can be convenient.

For production:

> Prefer controlled database migrations.

Examples include:

```text
Flyway
Liquibase
```

The important architectural principle:

```text
Application deployment
        +
Versioned DB migrations
```

rather than allowing an ORM to unpredictably modify production schemas.

---

# 57. Entity Callback Methods

JPA supports lifecycle callbacks.

Examples:

```java
@PrePersist
@PostPersist

@PreUpdate
@PostUpdate

@PreRemove
@PostRemove

@PostLoad
```

Example:

```java
@PrePersist
public void beforeInsert() {
    createdAt = LocalDateTime.now();
}
```

---

# 58. Why Callbacks?

Possible use cases:

```text
Audit fields
Normalization
Default values
Lifecycle bookkeeping
```

But don't put complex business workflows inside entity callbacks.

For example, avoid turning:

```java
@PreUpdate
```

into a hidden business-service execution engine.

---

# 59. Auditing Fields

Common entity fields:

```java
private Instant createdAt;
private Instant updatedAt;
private String createdBy;
private String updatedBy;
```

These are frequently required in enterprise systems.

How they're populated depends on the application architecture.

---

# 60. Attribute Converter

Suppose Java has:

```java
enum PaymentStatus {
    PENDING,
    PAID,
    FAILED
}
```

but database uses a custom representation.

JPA provides:

```java
@Converter
```

and:

```java
@Convert
```

to convert between Java and database representations.

Conceptually:

```text
Java type
   ↓
AttributeConverter
   ↓
Database type
```

---

# 61. Example Converter

```java
@Converter
public class BooleanConverter
        implements AttributeConverter<Boolean, String> {

    @Override
    public String convertToDatabaseColumn(Boolean value) {
        return Boolean.TRUE.equals(value) ? "Y" : "N";
    }

    @Override
    public Boolean convertToEntityAttribute(String value) {
        return "Y".equals(value);
    }
}
```

Then:

```java
@Convert(converter = BooleanConverter.class)
private Boolean active;
```

---

# 62. Entity Design and `equals/hashCode`

Entity equality is surprisingly difficult.

Don't blindly generate:

```java
equals()
hashCode()
```

using every field.

Consider:

```text
ID
business key
mutable fields
Hibernate proxies
entity lifecycle
```

A careless implementation can cause bugs in:

```text
HashSet
HashMap
collections
entity comparisons
```

For today's mastery, understand that **entity identity and Java equality are related but not identical concepts**.

---

# 63. Lazy Collections

Suppose:

```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

Don't automatically do:

```java
customer.getOrders().size();
```

in a loop.

You need to understand:

```text
When is the collection initialized?
How many SQL statements are executed?
What transaction is active?
How many rows are being loaded?
```

---

# 64. JPA Performance Checklist

When an application becomes slow, inspect:

```text
1. Generated SQL
2. Number of queries
3. N+1 queries
4. Fetch strategy
5. Join strategy
6. Indexes
7. Pagination
8. Entity graph size
9. Persistence-context size
10. Database execution plan
11. Connection pool
12. Transaction duration
```

Never assume:

```text
"JPA is slow."
```

First determine where the actual bottleneck is.

---

# 65. Persistence Context Memory

Suppose you process:

```text
1,000,000 records
```

with one persistence context.

The context can become very large.

A common batch-processing technique is periodically:

```java
entityManager.flush();
entityManager.clear();
```

Conceptually:

```text
Process 1000
   ↓
flush()
   ↓
clear()

Process next 1000
   ↓
flush()
   ↓
clear()
```

The appropriate batch size depends on the workload and environment.

---

# 66. Complete Advanced Repository

Example:

```java
public class JpaEmployeeRepository {

    private final EntityManager entityManager;

    public JpaEmployeeRepository(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    public Employee findById(Long id) {
        return entityManager.find(Employee.class, id);
    }

    public List<Employee> findByDepartment(String name) {

        return entityManager.createQuery("""
            SELECT e
            FROM Employee e
            JOIN e.department d
            WHERE d.name = :name
            ORDER BY e.name
            """, Employee.class)
            .setParameter("name", name)
            .getResultList();
    }

    public List<Employee> findPage(
            int offset,
            int limit) {

        return entityManager.createQuery("""
            SELECT e
            FROM Employee e
            ORDER BY e.id
            """, Employee.class)
            .setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
    }
}
```

---

# 67. Advanced Architecture

Your persistence layer should now look like:

```text
                 HTTP
                  │
                  ▼
             Controller
                  │
                  ▼
               Service
                  │
             Transaction
                  │
                  ▼
             Repository
                  │
                  ▼
            EntityManager
                  │
                  ▼
              JPA Provider
                  │
                  ▼
                JDBC
                  │
                  ▼
               Database
```

And relationships:

```text
Entity
  │
  ├── Relationships
  ├── Embedded types
  ├── Inheritance
  ├── Version
  └── Lifecycle
```

---

# 68. Day 12 Hands-On Lab

Build an advanced Employee domain.

## Entity 1

```text
Employee
```

Fields:

```text
id
name
email
salary
status
address
version
```

---

## Entity 2

```text
Department
```

Relationship:

```text
Department 1 → N Employee
```

---

## Embeddable

```text
Address
```

Fields:

```text
street
city
state
country
zipCode
```

---

## Exercise 1

Use:

```java
@Embedded
```

for `Address`.

---

## Exercise 2

Add:

```java
@Version
private Long version;
```

---

## Exercise 3

Implement:

```text
find employee by ID
find employees by department
find employees by salary range
find employees by status
```

---

## Exercise 4

Implement pagination.

---

## Exercise 5

Write a JPQL join:

```text
Employee → Department
```

---

## Exercise 6

Write a fetch join.

---

## Exercise 7

Create an employee summary DTO.

---

## Exercise 8

Write a native SQL query.

---

## Exercise 9

Write a Criteria API search with optional:

```text
name
department
salary
status
```

---

## Exercise 10

Demonstrate optimistic locking by running two concurrent update scenarios.

---

# 69. Day 12 Mini Project

## Employee Management System — Version 6

Add:

```text
Employee
Department
Address
EmployeeStatus
```

Features:

```text
Create Employee
Update Employee
Delete Employee
Search Employee
Search by Department
Search by Salary
Search by Status
Pagination
Employee Details
Employee Summary
Optimistic Locking
```

Architecture:

```text
Browser
   ↓
Servlet
   ↓
Service
   ↓
Repository
   ↓
EntityManager
   ↓
JPA
   ↓
Hibernate
   ↓
JDBC
   ↓
PostgreSQL/MySQL
```

---

# 70. Day 12 Interview Questions

## Fundamentals

1. What is the JPA entity lifecycle?
2. What is the difference between transient and detached?
3. What does `merge()` actually do?
4. Why does `merge()` return an entity?
5. What does `flush()` do?
6. What does `clear()` do?
7. What does `refresh()` do?
8. What is an embeddable?
9. Entity vs embeddable?
10. What is `@EmbeddedId`?

---

## Advanced

11. Explain JPA inheritance strategies.
12. SINGLE_TABLE vs JOINED?
13. JOINED vs TABLE_PER_CLASS?
14. What is a discriminator column?
15. What is composite-key mapping?
16. `@EmbeddedId` vs `@IdClass`?
17. What is JPQL?
18. JPQL vs native SQL?
19. What is a fetch join?
20. What is a DTO projection?
21. What is Criteria API?
22. Why use Criteria API?
23. How does pagination work?
24. Offset vs keyset pagination?
25. What are bulk JPQL updates?
26. Why can bulk updates create stale persistence-context state?
27. What is an entity graph?
28. What is `@Version`?
29. Explain optimistic locking.
30. Explain pessimistic locking.

---

## Architect-Level Questions

31. How would you prevent N+1 queries?

32. How would you design JPA for a table containing 500 million records?

33. When would you choose native SQL over JPQL?

34. When would you choose DTO projections?

35. When would you use an association entity instead of `@ManyToMany`?

36. How would you process 10 million records using JPA?

37. Why shouldn't you keep one huge persistence context during batch processing?

38. How do you decide between LAZY and EAGER?

39. How would you investigate a JPA performance problem?

40. How would you handle concurrent updates to the same entity?

41. Optimistic vs pessimistic locking?

42. How can bulk updates affect the persistence context?

43. How would you model polymorphic employees?

44. How would you design JPA entities for a high-traffic e-commerce system?

45. How would you prevent your REST API from accidentally loading a huge JPA object graph?

---

# 71. Scenario-Based Interview Problems

### Problem 1

You have:

```text
Customer
  ↓
Orders
```

A customer has 50,000 orders.

Your API returns customers in a list.

**Question:** How would you prevent orders from being loaded unnecessarily?

Think about:

```text
LAZY
DTO
projection
pagination
fetch plan
```

---

### Problem 2

An employee list endpoint generates:

```text
1 + 1001 queries
```

What would you investigate?

Expected concepts:

```text
N+1
fetch join
entity graph
projection
batch fetching
SQL logs
```

---

### Problem 3

Two users edit the same employee.

Both read:

```text
version = 10
```

One updates successfully.

What should happen to the second update?

Think:

```text
@Version
optimistic locking
```

---

### Problem 4

You need:

```text
Student
Course
Enrollment Date
Grade
Status
```

Would you use direct `@ManyToMany`?

Think:

```text
Association Entity
```

---

### Problem 5

A report needs:

```text
department
employee count
average salary
```

Should you load every Employee entity?

Think:

```text
GROUP BY
aggregation
DTO projection
```

---

# 72. Day 12 Production Rules

Memorize these:

### Rule 1

**Understand persistence context before learning advanced Hibernate tricks.**

### Rule 2

**Don't use EAGER loading as a universal fix.**

### Rule 3

**Design queries around use cases.**

### Rule 4

**Monitor generated SQL.**

### Rule 5

**Avoid N+1 queries.**

### Rule 6

**Use pagination for large datasets.**

### Rule 7

**Use DTO projections when you don't need complete entities.**

### Rule 8

**Use optimistic locking where concurrent updates can conflict.**

### Rule 9

**Use pessimistic locking only when the business scenario genuinely requires database locking.**

### Rule 10

**Don't let the persistence context grow without bounds during large batch processing.**

### Rule 11

**Bulk updates require awareness of stale managed entities.**

### Rule 12

**Don't let JPA entities become your entire application architecture.**

---

# 73. The Most Important Day 12 Mental Model

Memorize this:

```text
                 JPA
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Entities  Queries   Lifecycle
        │         │         │
        ▼         ▼         ▼
 Relationships  JPQL    Persistence
 Embedded       Native    Context
 Inheritance    Criteria  Dirty Checking
        │         │         │
        └─────────┼─────────┘
                  ▼
             Transactions
                  │
                  ▼
              Database
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Indexes   Locks     SQL
```

The key architectural thought process is:

> **Don't ask “Which JPA annotation should I use?” first. Ask “What is the business relationship, what data does this use case need, what SQL should ideally execute, how will concurrency work, and what database constraints/indexes are required?”**

---

# 74. Day 12 Mastery Checklist

Before moving to Day 13, you should be able to explain and implement:

* [ ] Entity lifecycle
* [ ] Transient state
* [ ] Managed state
* [ ] Detached state
* [ ] Removed state
* [ ] `persist()`
* [ ] `merge()`
* [ ] `remove()`
* [ ] `detach()`
* [ ] `clear()`
* [ ] `refresh()`
* [ ] `flush()`
* [ ] `@Embeddable`
* [ ] `@Embedded`
* [ ] Attribute overrides
* [ ] JPA inheritance
* [ ] SINGLE_TABLE
* [ ] JOINED
* [ ] TABLE_PER_CLASS
* [ ] Composite keys
* [ ] `@EmbeddedId`
* [ ] `@IdClass`
* [ ] JPQL joins
* [ ] Fetch joins
* [ ] Aggregations
* [ ] `GROUP BY`
* [ ] `HAVING`
* [ ] DTO projections
* [ ] Native SQL
* [ ] Criteria API
* [ ] Pagination
* [ ] Bulk operations
* [ ] Entity graphs
* [ ] `@Version`
* [ ] Optimistic locking
* [ ] Pessimistic locking
* [ ] Entity callbacks
* [ ] Attribute converters
* [ ] JPA performance
* [ ] N+1 prevention
* [ ] Batch processing

## ⭐ Day 12 Golden Rule

> **Advanced JPA is not about knowing more annotations. It is about controlling entity state, query shape, fetch behavior, concurrency, and database interaction deliberately.**

### Your JPA progression so far

```text
DAY 9
JDBC
 │
 ▼
DAY 10
JPA Fundamentals
 │
 ├── Entity
 ├── EntityManager
 ├── Persistence Context
 └── Dirty Checking
 │
 ▼
DAY 11
Relationships
 │
 ├── OneToOne
 ├── OneToMany
 ├── ManyToOne
 └── ManyToMany
 │
 ▼
DAY 12
Advanced JPA
 │
 ├── Advanced Queries
 ├── Embedded Types
 ├── Inheritance
 ├── Composite Keys
 ├── Locking
 ├── Entity Graphs
 └── Performance
 │
 ▼
DAY 13
TRANSACTIONS
```

**Day 13 is the next critical step: JPA + Transactions** — transaction boundaries, ACID, `EntityTransaction`, JTA, container-managed transactions, `@Transactional` concepts, propagation, isolation, rollback, timeout, read-only transactions, nested transactions, transaction synchronization, and real enterprise transaction scenarios.
