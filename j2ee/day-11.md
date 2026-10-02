# J2EE 30-Day Mastery — Day 11

## JPA Relationships — Mapping Real-World Data Models

**Date:** October 13, 2026
**Goal:** Master how JPA represents relationships between database tables and Java objects.

Today is extremely important because most real enterprise applications contain relationships such as:

```text
Customer → Orders
Order → OrderItems
Employee → Department
User → Roles
Product → Category
Student ↔ Course
```

You need to understand both sides:

```text
Object-oriented model
        ↕
Relational database model
        ↕
       JPA
```

---

# 1. Why Do We Need Relationships?

Suppose you have:

```text
Employee
---------
id
name
department_id
```

and:

```text
Department
----------
id
name
```

Database relationship:

```text
employees.department_id
          │
          ▼
departments.id
```

Java model:

```java
class Employee {
    private Department department;
}
```

JPA connects these two models.

---

# 2. The Four Major JPA Relationships

You must master:

```text
@OneToOne
@OneToMany
@ManyToOne
@ManyToMany
```

Visual overview:

```text
One Employee ───── One Laptop
@OneToOne

One Department ─── Many Employees
@OneToMany

Many Employees ─── One Department
@ManyToOne

Many Students ─── Many Courses
@ManyToMany
```

---

# 3. First Important Concept: Foreign Key

Suppose:

```text
departments
----------------
id
name
```

```text
employees
----------------
id
name
department_id
```

`department_id` is a foreign key:

```text
employees.department_id
           │
           ▼
departments.id
```

Usually, the table containing the foreign key represents the **many side**.

Therefore:

```text
Many Employees
      ↓
One Department
```

This naturally maps to:

```java
@ManyToOne
```

---

# 4. `@ManyToOne`

This is probably the most commonly used JPA relationship.

Example:

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
}
```

Department:

```java
@Entity
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

Database:

```text
employees
--------------------------------
id | name | department_id
--------------------------------
1  | John | 10
2  | Alice| 10
3  | Bob  | 20
```

```text
departments
----------------
id | name
----------------
10 | IT
20 | HR
```

Object relationship:

```text
John ───────┐
Alice ──────┼──→ IT
Bob ────────┘     │
                  │
                  ↓
                 HR
```

---

# 5. Why `@JoinColumn`?

```java
@JoinColumn(name = "department_id")
```

tells JPA:

> The foreign key column for this relationship is `department_id`.

Without explicit configuration, the provider may derive a column name according to its naming rules.

Production applications often make important mappings explicit.

---

# 6. Owning Side

This is one of the most important concepts today.

Consider:

```text
Employee → Department
```

The `employees` table contains:

```text
department_id
```

Therefore `Employee` controls the foreign-key relationship.

So:

```java
@ManyToOne
@JoinColumn(name = "department_id")
private Department department;
```

is the **owning side**.

Think:

> The side that controls the foreign key is generally the owning side.

---

# 7. Bidirectional Relationship

You can navigate:

```text
Employee → Department
```

and also:

```text
Department → Employees
```

Employee:

```java
@ManyToOne
@JoinColumn(name = "department_id")
private Department department;
```

Department:

```java
@OneToMany(mappedBy = "department")
private List<Employee> employees;
```

Complete:

```java
@Entity
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "department")
    private List<Employee> employees = new ArrayList<>();
}
```

---

# 8. What Does `mappedBy` Mean?

This is a critical interview topic.

```java
@OneToMany(mappedBy = "department")
```

means:

> This relationship is mapped by the `department` field in `Employee`.

It tells JPA:

```text
Department.employees
        ↓
Employee.department
```

Therefore:

```java
Employee.department
```

is the owning side.

---

# 9. `mappedBy` Does NOT Mean Database Column Name

This is a common mistake.

Wrong:

```java
@OneToMany(mappedBy = "department_id")
```

Usually wrong because:

```text
mappedBy → Java field name
```

Correct:

```java
@OneToMany(mappedBy = "department")
```

because:

```java
private Department department;
```

---

# 10. `@OneToMany`

Example:

```java
@OneToMany(mappedBy = "department")
private List<Employee> employees;
```

Meaning:

```text
One Department
      ↓
Many Employees
```

Example:

```text
IT
├── John
├── Alice
└── Bob
```

---

# 11. Unidirectional vs Bidirectional

## Unidirectional

Only one side knows about the other.

```text
Employee → Department
```

Employee:

```java
@ManyToOne
private Department department;
```

Department does not have:

```java
List<Employee>
```

---

## Bidirectional

Both sides know each other.

```text
Employee ↔ Department
```

Employee:

```java
@ManyToOne
private Department department;
```

Department:

```java
@OneToMany(mappedBy = "department")
private List<Employee> employees;
```

---

# 12. Which One Should You Use?

Don't automatically make every relationship bidirectional.

Bidirectional relationships provide convenient navigation but increase:

* Complexity
* Object graph size
* Serialization problems
* Potential recursion
* Maintenance burden

Use bidirectional relationships when both navigation directions are genuinely useful.

---

# 13. `@OneToOne`

Example:

```text
Employee
   │
   │
   ▼
Laptop
```

One employee has one laptop.

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToOne
    @JoinColumn(name = "laptop_id")
    private Laptop laptop;
}
```

Laptop:

```java
@Entity
public class Laptop {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String serialNumber;
}
```

Database:

```text
employees
--------------------------
id | name | laptop_id
--------------------------

laptops
--------------------------
id | serial_number
--------------------------
```

---

# 14. Bidirectional One-to-One

Employee:

```java
@OneToOne
@JoinColumn(name = "laptop_id")
private Laptop laptop;
```

Laptop:

```java
@OneToOne(mappedBy = "laptop")
private Employee employee;
```

Here:

```text
Employee.laptop
```

is the owning side.

---

# 15. `@OneToOne` with Unique Constraint

Conceptually:

```text
employee.laptop_id
```

should be unique if one laptop can belong to only one employee.

You can enforce this at the database level.

For example:

```java
@JoinColumn(
    name = "laptop_id",
    unique = true
)
```

But production database constraints should be considered part of the schema design rather than relying solely on application annotations.

---

# 16. `@ManyToMany`

Example:

```text
Students ↔ Courses
```

A student can take many courses.

A course can have many students.

```text
Student A ── Course 1
     │
     ├────── Course 2
     │
Student B ── Course 1
```

This cannot normally be represented with one foreign key column.

We need a **join table**.

---

# 17. Join Table

Database:

```text
students
---------
id
name
```

```text
courses
---------
id
name
```

Join table:

```text
student_course
-------------------
student_id
course_id
```

Conceptually:

```text
students
   │
   ▼
student_course
   ▲
   │
courses
```

---

# 18. JPA `@ManyToMany`

```java
@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}
```

Course:

```java
@Entity
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

---

# 19. Bidirectional Many-to-Many

Student:

```java
@ManyToMany
@JoinTable(
    name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
private Set<Course> courses;
```

Course:

```java
@ManyToMany(mappedBy = "courses")
private Set<Student> students;
```

The `Student` side owns the relationship.

---

# 20. Why `Set` Is Often Useful

For relationships such as:

```text
Student ↔ Course
```

you often don't want duplicate associations.

Using:

```java
Set<Course>
```

expresses:

> A student should not have the same course association twice.

But collection choice should reflect actual domain semantics.

---

# 21. Cascade

Cascade controls whether certain operations applied to one entity are propagated to related entities.

Common cascade types:

```text
PERSIST
MERGE
REMOVE
REFRESH
DETACH
ALL
```

Example:

```java
@OneToMany(
    mappedBy = "department",
    cascade = CascadeType.ALL
)
private List<Employee> employees;
```

---

# 22. Cascade Persist

Suppose:

```java
Department department = new Department();

Employee employee = new Employee();

department.getEmployees().add(employee);
```

With:

```java
cascade = CascadeType.PERSIST
```

persisting the department can propagate persistence to its employees.

Conceptually:

```text
persist(department)
       ↓
Employee also persisted
```

---

# 23. Cascade Remove

```java
@OneToMany(
    mappedBy = "department",
    cascade = CascadeType.REMOVE
)
```

Deleting the parent can cause related children to be removed.

Be extremely careful.

For example:

```text
Department
   ↓
Employees
```

Deleting a department should not automatically delete employees if employees have an independent lifecycle.

---

# 24. `CascadeType.ALL`

```java
cascade = CascadeType.ALL
```

means all cascade operations are propagated.

This is convenient but **not automatically correct**.

Don't blindly use:

```java
CascadeType.ALL
```

everywhere.

---

# 25. Cascade vs Database Cascade

These are different concepts.

### JPA cascade

Controls entity operations:

```text
persist
merge
remove
...
```

### Database cascade

Database foreign-key behavior:

```sql
ON DELETE CASCADE
```

They operate at different layers.

---

# 26. `orphanRemoval`

Example:

```java
@OneToMany(
    mappedBy = "order",
    orphanRemoval = true
)
private List<OrderItem> items;
```

Suppose:

```java
order.getItems().remove(item);
```

The removed child can be deleted from the database when the relationship is synchronized.

Conceptually:

```text
Order
 ├── Item A
 ├── Item B
 └── Item C

remove(Item B)

Order
 ├── Item A
 └── Item C
```

Item B becomes an orphan and can be removed.

---

# 27. Cascade vs `orphanRemoval`

Don't confuse them.

### Cascade

```text
Parent operation
       ↓
Child operation
```

### orphanRemoval

```text
Child removed from parent's relationship
       ↓
Child may be deleted
```

---

# 28. Real-World Example: Order

Consider:

```text
Order
 ├── OrderItem
 ├── OrderItem
 └── OrderItem
```

An `OrderItem` usually has no independent meaning outside its order.

This is a good candidate for:

```java
@OneToMany(
    mappedBy = "order",
    cascade = CascadeType.ALL,
    orphanRemoval = true
)
private List<OrderItem> items;
```

But this depends on your domain and retention requirements.

---

# 29. Lazy Loading

Relationships can be loaded lazily.

Conceptually:

```text
Employee loaded
      ↓
Department NOT loaded yet
      ↓
employee.getDepartment()
      ↓
Department loaded
```

This can reduce unnecessary database work.

---

# 30. Eager Loading

Eager means the relationship is intended to be loaded along with the entity.

Conceptually:

```text
Employee
   +
Department
   ↓
loaded together
```

But "eager" does not mean:

> always exactly one SQL query.

The provider may use different SQL/query strategies.

---

# 31. Default Fetch Types

For JPA relationship mappings, know the standard defaults:

```text
@ManyToOne → EAGER
@OneToOne  → EAGER

@OneToMany → LAZY
@ManyToMany → LAZY
```

These defaults are one reason developers often explicitly review fetch behavior.

---

# 32. Why Lazy Loading Is Important

Suppose:

```text
10,000 Employees
```

and every employee has:

```text
Department
Address
Manager
Projects
Skills
```

If everything is eagerly loaded:

```text
Employee
 ↓
Department
 ↓
Address
 ↓
Manager
 ↓
Projects
 ↓
Skills
```

You can create:

* Huge queries
* Large object graphs
* Memory consumption
* Slow response times
* Excessive database access

---

# 33. LazyInitializationException

A classic Hibernate/JPA problem.

Conceptually:

```java
Employee employee = repository.findById(id);
```

Persistence context ends.

Later:

```java
employee.getDepartment().getName();
```

If `department` is lazy and no active persistence context exists, the provider may be unable to initialize it.

This can result in:

```text
LazyInitializationException
```

The proper solution is **not** simply:

```text
Make everything EAGER
```

Instead design transaction boundaries and queries deliberately.

---

# 34. N+1 Problem

Suppose:

```java
List<Employee> employees = ...
```

Then:

```java
for (Employee employee : employees) {
    employee.getDepartment().getName();
}
```

Potential SQL:

```text
1 query → employees

+
N queries → departments
```

For 100 employees:

```text
101 queries
```

This is N+1.

---

# 35. Entity Graph / Fetch Join Preview

Later you'll learn ways to fetch exactly what a use case needs.

For example, JPQL fetch join:

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

loaded as part of the query.

This is preferable to blindly changing everything to EAGER.

---

# 36. Owning Side — Interview Example

Suppose:

```java
class Department {

    @OneToMany(mappedBy = "department")
    private List<Employee> employees;
}
```

and:

```java
class Employee {

    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
}
```

Question:

> Which side owns the relationship?

Answer:

```text
Employee
```

because:

```java
@JoinColumn(name = "department_id")
```

controls the foreign key.

---

# 37. Synchronizing Both Sides

Suppose:

```java
Department department = ...;
Employee employee = ...;
```

Don't do only:

```java
department.getEmployees().add(employee);
```

and assume the database relationship is correctly synchronized.

Also establish:

```java
employee.setDepartment(department);
```

A useful domain helper:

```java
public void addEmployee(Employee employee) {

    employees.add(employee);
    employee.setDepartment(this);
}
```

And:

```java
public void removeEmployee(Employee employee) {

    employees.remove(employee);
    employee.setDepartment(null);
}
```

This keeps the in-memory object graph consistent.

---

# 38. Real Enterprise Example

Let's model:

```text
Department
     │
     │ 1
     │
     │
     │ N
   Employee
```

Department:

```java
@Entity
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(
        mappedBy = "department",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    private List<Employee> employees = new ArrayList<>();

    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }

    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
}
```

Employee:

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    public void setDepartment(Department department) {
        this.department = department;
    }
}
```

---

# 39. Why `FetchType.LAZY` for Many-to-One?

For many-to-one relationships, JPA's default is EAGER.

In many enterprise applications, developers explicitly choose:

```java
@ManyToOne(fetch = FetchType.LAZY)
```

so the related entity is loaded only when the use case needs it.

But remember:

> Fetch strategy should be designed around query/use-case requirements.

---

# 40. Many-to-Many Warning

A direct:

```java
@ManyToMany
```

is convenient for simple relationships.

But enterprise systems frequently need additional information about the association.

Example:

```text
Student
Course
Enrollment
```

Enrollment might contain:

```text
id
student_id
course_id
enrollment_date
status
grade
```

Now a direct many-to-many mapping becomes inadequate.

Instead model:

```text
Student
   │
   │ 1
   ▼
Enrollment
   ▲
   │ N
Course
```

This is a very important architectural modeling technique.

---

# 41. Many-to-Many → Association Entity

Instead of:

```text
Student ↔ Course
```

use:

```text
Student
   │
   ▼
Enrollment
   ▲
   │
Course
```

Entity:

```java
@Entity
public class Enrollment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    private Course course;

    private LocalDate enrollmentDate;

    private String status;
}
```

This is often much more flexible.

---

# 42. Relationship Decision Framework

When designing entities, ask:

### Question 1

Can one A have multiple B?

### Question 2

Can one B belong to multiple A?

Then:

```text
A 1 → 1 B
@OneToOne
```

```text
A 1 → N B
@OneToMany
```

```text
A N → 1 B
@ManyToOne
```

```text
A N ↔ N B
@ManyToMany
```

---

# 43. Database vs JPA Thinking

Suppose the database has:

```text
orders
----------------
id
customer_id
```

Database thinking:

```text
orders.customer_id → customers.id
```

JPA thinking:

```java
@ManyToOne
private Customer customer;
```

JPA translates the object relationship into the relational relationship.

---

# 44. Don't Design Only From Java

Bad approach:

```text
Create Java classes
       ↓
Add annotations
       ↓
Hope database works
```

Better:

```text
Business requirements
       ↓
Domain model
       ↓
Data model
       ↓
Relationships
       ↓
Constraints
       ↓
Indexes
       ↓
JPA mappings
       ↓
Queries
```

For a solutions architect, this distinction is crucial.

---

# 45. Relationship Constraints

Don't rely only on Java/JPA.

Database should enforce important invariants.

Examples:

```text
PRIMARY KEY
FOREIGN KEY
UNIQUE
NOT NULL
CHECK
INDEX
```

For example:

```text
employee.department_id
```

should have an appropriate foreign key constraint.

---

# 46. Relationship Performance

Every relationship introduces potential performance considerations:

```text
JOIN
FETCH
LAZY
EAGER
N+1
INDEX
CARDINALITY
OBJECT GRAPH
```

Suppose:

```text
Customer → Orders
```

and customer has:

```text
1,000,000 orders
```

You absolutely do not want:

```java
customer.getOrders()
```

to accidentally load a million objects.

Design collection access carefully.

---

# 47. Pagination with Relationships

Avoid blindly loading:

```java
List<Order> orders;
```

for huge collections.

Prefer queries that retrieve the required page:

```text
GET /customers/10/orders?page=0&size=20
```

Database:

```text
LIMIT 20
```

Conceptually:

```text
Customer
   ↓
Orders
   ↓
Pagination
   ↓
20 rows
```

---

# 48. JSON Serialization Problem

Bidirectional relationships can create:

```text
Employee
  ↓
Department
  ↓
Employees
  ↓
Department
  ↓
Employees
```

This can create infinite recursion during naive JSON serialization.

Architecture lesson:

> Don't expose your JPA entity graph directly as your public REST API model.

Prefer DTOs.

Example:

```java
public record EmployeeResponse(
    Long id,
    String name,
    Long departmentId,
    String departmentName
) {
}
```

---

# 49. Entity Graph vs API Model

Your entity might contain:

```text
Employee
 ├── Department
 ├── Manager
 ├── Projects
 ├── Skills
 └── Address
```

But API might need:

```json
{
  "id": 1,
  "name": "John",
  "department": "Engineering"
}
```

DTOs prevent accidental exposure/loading of the entire entity graph.

---

# 50. `CascadeType.REMOVE` Danger

Consider:

```text
Company
   ↓
Employee
```

If you write:

```java
@OneToMany(cascade = CascadeType.ALL)
private List<Employee> employees;
```

then deleting the company can cascade deletion to employees.

That may be completely wrong if employees must remain in the system.

Therefore:

> Cascade decisions must follow ownership/lifecycle semantics, not convenience.

---

# 51. `orphanRemoval` Danger

Consider:

```java
@OneToMany(orphanRemoval = true)
private List<Address> addresses;
```

Removing:

```java
customer.getAddresses().remove(address);
```

can result in database deletion.

That's appropriate only when:

> The child is conceptually owned by the parent and should not exist independently.

---

# 52. Production Design Example

For an e-commerce application:

```text
Customer
   │
   └── 1:N → Order
                  │
                  └── 1:N → OrderItem
                               │
                               └── N:1 → Product
```

Mappings:

```java
Customer
@OneToMany → Order
```

```java
Order
@ManyToOne → Customer
@OneToMany → OrderItem
```

```java
OrderItem
@ManyToOne → Order
@ManyToOne → Product
```

This model is far more common in enterprise systems than simplistic many-to-many mappings.

---

# 53. Complete E-Commerce Relationship Model

```text
Customer
   │
   │ 1
   ▼
 Order
   │
   │ 1
   ▼
OrderItem ───────── N:1 ─────── Product
```

Database:

```text
customers
orders
order_items
products
```

Foreign keys:

```text
orders.customer_id
order_items.order_id
order_items.product_id
```

JPA:

```text
Customer
   ↓
@OneToMany
   ↓
Order
   ↓
@OneToMany
   ↓
OrderItem
   ↓
@ManyToOne
   ↓
Product
```

---

# 54. Interview Questions — Day 11

## Beginner

1. What is a JPA relationship?
2. What are the four relationship types?
3. What is `@ManyToOne`?
4. What is `@OneToMany`?
5. What is `@OneToOne`?
6. What is `@ManyToMany`?
7. What is `@JoinColumn`?
8. What is a foreign key?
9. What is a join table?
10. What is `mappedBy`?

## Intermediate

11. What is owning side?
12. What is inverse side?
13. Unidirectional vs bidirectional?
14. What does `mappedBy` reference?
15. What is cascade?
16. What is `CascadeType.ALL`?
17. What is orphan removal?
18. Cascade vs orphan removal?
19. What is lazy loading?
20. What is eager loading?

## Advanced

21. What is the default fetch type for `@ManyToOne`?
22. What is the default fetch type for `@OneToMany`?
23. Why can EAGER loading cause performance problems?
24. What is N+1?
25. How do you solve N+1?
26. Why is `mappedBy` not a column name?
27. Why should both sides of a bidirectional relationship be synchronized?
28. Why can bidirectional relationships cause JSON recursion?
29. Why use DTOs instead of exposing entities?
30. When should you avoid `@ManyToMany`?
31. What is an association entity?
32. Cascade vs database cascade?
33. When should you use `orphanRemoval`?
34. Why is `CascadeType.REMOVE` dangerous?
35. How do you model `Order → OrderItem`?
36. How do you model `Student → Course` with enrollment information?
37. How does JPA represent foreign keys?
38. How does lazy loading work conceptually?
39. What causes `LazyInitializationException`?
40. How would you design relationships for millions of child records?

---

# 55. Day 11 Coding Exercises

### Exercise 1

Create:

```text
Department → Employee
```

using:

```java
@OneToMany
@ManyToOne
```

---

### Exercise 2

Make the relationship bidirectional.

---

### Exercise 3

Identify the owning side.

---

### Exercise 4

Add:

```java
cascade = CascadeType.PERSIST
```

and test it.

---

### Exercise 5

Test:

```java
CascadeType.REMOVE
```

and observe what happens.

---

### Exercise 6

Implement:

```java
department.addEmployee(employee);
department.removeEmployee(employee);
```

while keeping both sides synchronized.

---

### Exercise 7

Create:

```text
Employee → Laptop
```

using `@OneToOne`.

---

### Exercise 8

Create:

```text
Student ↔ Course
```

using `@ManyToMany`.

---

### Exercise 9

Replace the many-to-many relationship with:

```text
Student
Enrollment
Course
```

---

### Exercise 10

Create:

```text
Customer → Order → OrderItem → Product
```

with correct JPA relationships.

---

# 56. Day 11 Mini Project

Upgrade your Employee Management System.

## Version 5 — Department Management

Add:

```text
Department
Employee
```

Relationship:

```text
Department 1 ───── N Employee
```

Features:

```text
Create Department
Create Employee
Assign Employee to Department
Remove Employee from Department
Find Employees by Department
List Departments
View Department Employees
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
JPA/Hibernate
   ↓
Database
```

---

# 57. Suggested Entity Design

### Department

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @OneToMany(
        mappedBy = "department",
        cascade = CascadeType.PERSIST
    )
    private List<Employee> employees = new ArrayList<>();

    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }
}
```

### Employee

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    public void setDepartment(Department department) {
        this.department = department;
    }
}
```

---

# 58. Day 11 Architecture Mental Model

When you see:

```text
Business Requirement
```

translate it into:

```text
Business relationship
       ↓
Cardinality
       ↓
Database foreign key
       ↓
Owning side
       ↓
JPA annotation
       ↓
Fetch strategy
       ↓
Cascade strategy
       ↓
Transaction boundary
       ↓
Query strategy
       ↓
API DTO
```

That's the level at which you should start thinking as a backend engineer progressing toward architecture.

---

# 59. Day 11 Cheat Sheet

```text
@OneToOne
A ─── B

@OneToMany
A ───< B

@ManyToOne
A >─── B

@ManyToMany
A >───< B
```

### Foreign key

```text
Many side usually contains FK
```

### Owning side

```text
Side controlling relationship/FK
```

### `mappedBy`

```text
References Java field
```

### Lazy

```text
Load when needed
```

### Eager

```text
Intended to load immediately
```

### Cascade

```text
Propagate entity operation
```

### orphanRemoval

```text
Remove child when orphaned from relationship
```

### N+1

```text
1 parent query + N child queries
```

---

# 60. Day 11 Golden Rules

### Rule 1

**Understand the database relationship before writing JPA annotations.**

### Rule 2

**The foreign-key side is usually the owning side.**

### Rule 3

**`mappedBy` references a Java property, not a database column.**

### Rule 4

**Don't make every relationship bidirectional.**

### Rule 5

**Don't use `CascadeType.ALL` blindly.**

### Rule 6

**Don't use EAGER loading as a solution to lazy-loading problems.**

### Rule 7

**Always think about N+1 queries.**

### Rule 8

**For many-to-many relationships containing attributes, prefer an association entity.**

### Rule 9

**Keep both sides of bidirectional relationships synchronized in the Java object graph.**

### Rule 10

**Don't expose large JPA entity graphs directly through REST APIs; use DTOs.**

---

## 🧠 Day 11 Final Mental Model

Memorize this:

```text
                JPA RELATIONSHIPS

                  Database
                     │
              Foreign Keys
                     │
                     ▼
              Cardinality
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   One-to-One    One-to-Many    Many-to-Many
                     │
                     ▼
                Owning Side
                     │
                     ▼
                mappedBy
                     │
                     ▼
             Cascade / Orphan
                     │
                     ▼
             Lazy / Eager
                     │
                     ▼
                JPQL / SQL
                     │
                     ▼
                Performance
```

The most important progression for today is:

> **Business relationship → database foreign key/join table → owning side → JPA mapping → fetch strategy → cascade strategy → query/performance behavior.**

**Day 12** will build on this with **Advanced JPA**: entity lifecycle callbacks, embedded objects, inheritance strategies, composite keys, `@Version`, optimistic/pessimistic locking, JPQL joins, Criteria API, native queries, specifications, projections, pagination, bulk updates/deletes, and advanced persistence-context behavior.
