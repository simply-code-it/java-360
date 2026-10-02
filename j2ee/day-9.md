# J2EE Mastery — Day 9

## JDBC Deep Dive & Production-Grade Database Access

**Date:** October 11, 2026
**Goal:** Master JDBC deeply enough to build reliable, secure, transactional enterprise applications and understand what higher-level persistence frameworks such as JPA/Hibernate are doing underneath.

Today is one of the most important days in the J2EE roadmap.

---

# 1. What You Will Master Today

By the end of Day 9, you should understand:

1. What JDBC is
2. JDBC architecture
3. JDBC API
4. JDBC drivers
5. `DriverManager`
6. `DataSource`
7. `Connection`
8. `Statement`
9. `PreparedStatement`
10. `CallableStatement`
11. `ResultSet`
12. `ResultSetMetaData`
13. `DatabaseMetaData`
14. CRUD with JDBC
15. DAO pattern
16. SQL injection
17. Resource management
18. Try-with-resources
19. JDBC exceptions
20. Transactions
21. Commit
22. Rollback
23. Auto-commit
24. Savepoints
25. Transaction isolation
26. Batch processing
27. Generated keys
28. Connection pooling
29. `DataSource`
30. JDBC architecture in J2EE
31. JDBC + Servlet + Service + JSP
32. Production JDBC practices

---

# 2. What Is JDBC?

**JDBC = Java Database Connectivity.**

It is the standard Java API for interacting with relational databases.

Examples:

```text
MySQL
PostgreSQL
Oracle
SQL Server
MariaDB
H2
```

Java application:

```text
Java
  ↓
JDBC API
  ↓
JDBC Driver
  ↓
Database
```

---

# 3. Why JDBC Exists

Without JDBC, Java code would need database-specific APIs.

For example:

```text
Java
 ↓
MySQL-specific API
```

Another application:

```text
Java
 ↓
Oracle-specific API
```

JDBC provides a common programming model:

```text
Java Application
       ↓
     JDBC
       ↓
 JDBC Driver
       ↓
   Database
```

The driver translates JDBC operations into the database's protocol.

---

# 4. JDBC Architecture

```text
┌─────────────────────────┐
│    Java Application     │
│                         │
│ Controller → Service    │
│             → DAO      │
└────────────┬────────────┘
             ↓
┌─────────────────────────┐
│       JDBC API          │
│                         │
│ Connection              │
│ PreparedStatement       │
│ ResultSet               │
│ DataSource              │
└────────────┬────────────┘
             ↓
┌─────────────────────────┐
│      JDBC Driver        │
└────────────┬────────────┘
             ↓
┌─────────────────────────┐
│        Database         │
└─────────────────────────┘
```

---

# 5. Important JDBC Interfaces

Memorize these:

```text
Connection
Statement
PreparedStatement
CallableStatement
ResultSet
DataSource
DatabaseMetaData
ResultSetMetaData
Savepoint
```

---

# 6. JDBC Driver

The JDBC driver is the implementation that allows Java to communicate with a specific database.

For example:

```text
Java
 ↓
JDBC API
 ↓
PostgreSQL JDBC Driver
 ↓
PostgreSQL
```

or:

```text
Java
 ↓
JDBC API
 ↓
MySQL JDBC Driver
 ↓
MySQL
```

Modern JDBC drivers generally register themselves automatically through Java's service-provider mechanism.

---

# 7. JDBC Connection

A `Connection` represents a database session from your application.

Conceptually:

```java
Connection connection =
        dataSource.getConnection();
```

You can then execute SQL:

```java
PreparedStatement statement =
        connection.prepareStatement(
                "SELECT * FROM employee"
        );
```

---

# 8. DriverManager

The older/simple approach:

```java
Connection connection =
        DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/company",
                "root",
                "password"
        );
```

Useful for:

```text
learning
small utilities
simple standalone applications
```

For enterprise applications, prefer a managed `DataSource` and connection pooling.

---

# 9. DataSource

`DataSource` is generally preferred in enterprise applications.

```java
DataSource dataSource;
```

Then:

```java
try (Connection connection =
         dataSource.getConnection()) {

    // database operations
}
```

Why?

Because a container/application infrastructure can manage:

```text
connection pooling
configuration
credentials
timeouts
resource lifecycle
JNDI
```

---

# 10. DriverManager vs DataSource

| Feature                  | DriverManager  | DataSource               |
| ------------------------ | -------------- | ------------------------ |
| Simple connection        | Yes            | Yes                      |
| Connection pooling       | Not by itself  | Commonly                 |
| Enterprise usage         | Less preferred | Preferred                |
| JNDI integration         | Limited        | Strong                   |
| Configuration management | Basic          | Better                   |
| Resource management      | Application    | Infrastructure/container |

---

# 11. First JDBC Program

Example:

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class JdbcExample {

    public static void main(String[] args)
            throws Exception {

        String url =
                "jdbc:mysql://localhost:3306/company";

        String username = "root";
        String password = "password";

        try (Connection connection =
                     DriverManager.getConnection(
                             url,
                             username,
                             password
                     );

             Statement statement =
                     connection.createStatement();

             ResultSet resultSet =
                     statement.executeQuery(
                             "SELECT id, name FROM employee"
                     )) {

            while (resultSet.next()) {

                long id =
                        resultSet.getLong("id");

                String name =
                        resultSet.getString("name");

                System.out.println(
                        id + " " + name
                );
            }
        }
    }
}
```

---

# 12. JDBC Resource Lifecycle

Think:

```text
Connection
   ↓
Statement
   ↓
ResultSet
```

More precisely:

```text
getConnection()
      ↓
prepare/create statement
      ↓
execute SQL
      ↓
process ResultSet
      ↓
close ResultSet
      ↓
close Statement
      ↓
close Connection
```

Try-with-resources handles this automatically.

---

# 13. Try-With-Resources

Bad:

```java
Connection connection = null;
PreparedStatement statement = null;

try {
    connection = ...
    statement = ...
} finally {
    statement.close();
    connection.close();
}
```

Better:

```java
try (Connection connection =
         dataSource.getConnection();

     PreparedStatement statement =
         connection.prepareStatement(sql)) {

    // execute

}
```

Resources are automatically closed.

---

# 14. Statement

You can execute SQL using:

```java
Statement statement =
        connection.createStatement();
```

Example:

```java
ResultSet resultSet =
        statement.executeQuery(
                "SELECT * FROM employee"
        );
```

But don't use `Statement` with user-controlled values.

---

# 15. SQL Injection

Never do this:

```java
String username =
        request.getParameter("username");

String sql =
        "SELECT * FROM users WHERE username = '"
        + username
        + "'";
```

An attacker could manipulate the input to alter the SQL semantics.

This is SQL injection.

---

# 16. PreparedStatement

Use:

```java
String sql =
        "SELECT * FROM employee WHERE id = ?";

PreparedStatement statement =
        connection.prepareStatement(sql);

statement.setLong(1, employeeId);
```

Then:

```java
ResultSet resultSet =
        statement.executeQuery();
```

The SQL structure and parameter values are handled separately.

---

# 17. PreparedStatement Example

```java
public Employee findById(Long id)
        throws SQLException {

    String sql =
        """
        SELECT id, name, department, salary
        FROM employee
        WHERE id = ?
        """;

    try (Connection connection =
             dataSource.getConnection();

         PreparedStatement statement =
             connection.prepareStatement(sql)) {

        statement.setLong(1, id);

        try (ResultSet resultSet =
                 statement.executeQuery()) {

            if (resultSet.next()) {

                return mapRow(resultSet);
            }

            return null;
        }
    }
}
```

---

# 18. Parameter Binding

Different Java types have corresponding setter methods.

```java
statement.setString(1, name);
```

```java
statement.setLong(1, id);
```

```java
statement.setInt(1, age);
```

```java
statement.setBigDecimal(1, salary);
```

```java
statement.setBoolean(1, active);
```

```java
statement.setTimestamp(1, timestamp);
```

---

# 19. `ResultSet`

A `ResultSet` represents rows returned by a query.

Example:

```java
while (resultSet.next()) {

    long id =
            resultSet.getLong("id");

    String name =
            resultSet.getString("name");
}
```

Initially the cursor is positioned before the first row.

```text
Before first
     ↓ next()
Row 1
     ↓ next()
Row 2
     ↓ next()
Row 3
     ↓ next()
false
```

---

# 20. ResultSet Access

By column name:

```java
resultSet.getString("name");
```

By column index:

```java
resultSet.getString(2);
```

Column names are usually more maintainable than numeric positions.

---

# 21. INSERT

Example:

```java
String sql =
    """
    INSERT INTO employee
        (name, department, salary)
    VALUES
        (?, ?, ?)
    """;

try (Connection connection =
         dataSource.getConnection();

     PreparedStatement statement =
         connection.prepareStatement(sql)) {

    statement.setString(1, employee.getName());
    statement.setString(2, employee.getDepartment());
    statement.setBigDecimal(
            3,
            employee.getSalary()
    );

    int rows =
            statement.executeUpdate();
}
```

---

# 22. `executeQuery()`

Use for queries that return a `ResultSet`.

```java
ResultSet resultSet =
        statement.executeQuery();
```

Typically:

```text
SELECT
```

---

# 23. `executeUpdate()`

Used for:

```text
INSERT
UPDATE
DELETE
```

Example:

```java
int affectedRows =
        statement.executeUpdate();
```

It returns the number of affected rows.

---

# 24. `execute()`

More general-purpose:

```java
boolean result =
        statement.execute();
```

It can handle SQL where the type/result isn't known in advance.

For normal CRUD, prefer the more specific APIs.

---

# 25. UPDATE

```java
String sql =
    """
    UPDATE employee
    SET name = ?,
        department = ?,
        salary = ?
    WHERE id = ?
    """;

try (Connection connection =
         dataSource.getConnection();

     PreparedStatement statement =
         connection.prepareStatement(sql)) {

    statement.setString(1, employee.getName());
    statement.setString(
            2,
            employee.getDepartment()
    );
    statement.setBigDecimal(
            3,
            employee.getSalary()
    );
    statement.setLong(4, employee.getId());

    int rows =
            statement.executeUpdate();

    if (rows == 0) {
        throw new EmployeeNotFoundException(
                employee.getId()
        );
    }
}
```

---

# 26. DELETE

```java
String sql =
    "DELETE FROM employee WHERE id = ?";

try (Connection connection =
         dataSource.getConnection();

     PreparedStatement statement =
         connection.prepareStatement(sql)) {

    statement.setLong(1, id);

    int rows =
            statement.executeUpdate();
}
```

---

# 27. DAO Pattern

A DAO encapsulates database operations.

```java
public interface EmployeeDAO {

    Employee findById(Long id);

    List<Employee> findAll();

    void save(Employee employee);

    void update(Employee employee);

    void delete(Long id);
}
```

Implementation:

```text
EmployeeDAO
     ↑
     |
JdbcEmployeeDAO
```

---

# 28. JDBC DAO Architecture

```text
EmployeeServlet
      ↓
EmployeeService
      ↓
EmployeeDAO
      ↓
JdbcEmployeeDAO
      ↓
JDBC
      ↓
Database
```

This keeps JDBC details out of the Controller.

---

# 29. Row Mapping

Avoid repeating:

```java
Employee employee =
        new Employee();

employee.setId(
        resultSet.getLong("id")
);

employee.setName(
        resultSet.getString("name")
);

employee.setDepartment(
        resultSet.getString("department")
);
```

Create a mapper:

```java
private Employee mapRow(
        ResultSet rs)
        throws SQLException {

    Employee employee =
            new Employee();

    employee.setId(
            rs.getLong("id")
    );

    employee.setName(
            rs.getString("name")
    );

    employee.setDepartment(
            rs.getString("department")
    );

    employee.setSalary(
            rs.getBigDecimal("salary")
    );

    return employee;
}
```

Then:

```java
while (rs.next()) {
    employees.add(mapRow(rs));
}
```

---

# 30. Generated Keys

Suppose the database generates:

```text
employee.id
```

You can request generated keys.

```java
String sql =
    """
    INSERT INTO employee
        (name, department, salary)
    VALUES (?, ?, ?)
    """;

try (PreparedStatement statement =
         connection.prepareStatement(
             sql,
             Statement.RETURN_GENERATED_KEYS
         )) {

    statement.setString(1, employee.getName());
    statement.setString(2, employee.getDepartment());
    statement.setBigDecimal(
            3,
            employee.getSalary()
    );

    statement.executeUpdate();

    try (ResultSet keys =
             statement.getGeneratedKeys()) {

        if (keys.next()) {
            long generatedId =
                    keys.getLong(1);

            employee.setId(generatedId);
        }
    }
}
```

---

# 31. Transactions

This is one of the most important JDBC concepts.

Suppose we transfer:

```text
₹1,000
```

from:

```text
Account A
```

to:

```text
Account B
```

We need:

```text
Debit A
+
Credit B
```

to behave as one logical operation.

---

# 32. Transaction Without Proper Handling

Imagine:

```text
Debit A
   ↓
SUCCESS

Credit B
   ↓
FAILURE
```

Now:

```text
A = -₹1000
B = unchanged
```

That's inconsistent.

We need atomicity.

---

# 33. JDBC Auto-Commit

By default, connections generally operate with auto-commit enabled.

That means each successful SQL statement may be committed automatically.

For multi-step transactions:

```java
connection.setAutoCommit(false);
```

Then:

```text
operation 1
operation 2
operation 3
     ↓
   commit
```

---

# 34. Commit

```java
connection.commit();
```

Example:

```java
connection.setAutoCommit(false);

try {

    debitAccount(connection);

    creditAccount(connection);

    connection.commit();

} catch (Exception e) {

    connection.rollback();

    throw e;
}
```

---

# 35. Rollback

If something fails:

```java
connection.rollback();
```

The transaction's changes since the relevant transaction boundary are undone according to the database's transaction semantics.

---

# 36. Complete Transfer Example

```java
public void transfer(
        Long fromAccount,
        Long toAccount,
        BigDecimal amount)
        throws SQLException {

    try (Connection connection =
             dataSource.getConnection()) {

        connection.setAutoCommit(false);

        try {

            debit(
                connection,
                fromAccount,
                amount
            );

            credit(
                connection,
                toAccount,
                amount
            );

            connection.commit();

        } catch (Exception e) {

            connection.rollback();

            throw e;
        }
    }
}
```

Mental model:

```text
BEGIN
  |
  ├── Debit
  |
  ├── Credit
  |
  └── COMMIT

Failure anywhere
       ↓
    ROLLBACK
```

---

# 37. Savepoint

A savepoint allows partial rollback within a transaction.

```java
Savepoint savepoint =
        connection.setSavepoint();
```

Later:

```java
connection.rollback(savepoint);
```

Conceptually:

```text
Operation A
Operation B
SAVEPOINT
Operation C
Operation D
   ↓
rollback(savepoint)
```

A savepoint does not mean the transaction itself has committed.

---

# 38. Transaction Isolation

Transactions can interact with concurrent transactions.

Common isolation levels:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

The exact behavior depends on the database engine.

---

# 39. Dirty Read

Transaction A:

```text
UPDATE salary
```

but hasn't committed.

Transaction B reads that uncommitted value.

If A rolls back, B saw data that never actually committed.

That's a **dirty read**.

---

# 40. Non-Repeatable Read

Transaction A:

```text
SELECT salary
```

gets:

```text
50000
```

Transaction B updates and commits:

```text
75000
```

Transaction A performs the same query again:

```text
75000
```

The same transaction observed different committed values.

---

# 41. Phantom Read

Transaction A:

```sql
SELECT *
FROM employee
WHERE salary > 50000;
```

Later Transaction B inserts a new matching employee and commits.

Transaction A repeats the query and sees an additional row.

That is a phantom read.

---

# 42. Setting Isolation

JDBC:

```java
connection.setTransactionIsolation(
        Connection.TRANSACTION_READ_COMMITTED
);
```

Possible constants:

```text
TRANSACTION_READ_UNCOMMITTED
TRANSACTION_READ_COMMITTED
TRANSACTION_REPEATABLE_READ
TRANSACTION_SERIALIZABLE
```

Don't blindly select the strongest isolation level; it can affect concurrency and performance.

---

# 43. Batch Processing

Suppose you need to insert 10,000 employees.

Doing:

```text
INSERT
INSERT
INSERT
...
```

individually can be inefficient.

Use batching:

```java
String sql =
    """
    INSERT INTO employee
        (name, department, salary)
    VALUES (?, ?, ?)
    """;

try (PreparedStatement statement =
         connection.prepareStatement(sql)) {

    for (Employee employee : employees) {

        statement.setString(
                1,
                employee.getName()
        );

        statement.setString(
                2,
                employee.getDepartment()
        );

        statement.setBigDecimal(
                3,
                employee.getSalary()
        );

        statement.addBatch();
    }

    statement.executeBatch();
}
```

---

# 44. Why Batch Processing?

Potential benefits:

```text
fewer round trips
better throughput
lower per-statement overhead
```

Actual performance depends on:

```text
database
driver
network
batch size
SQL
hardware
configuration
```

---

# 45. CallableStatement

Used to invoke database stored procedures.

Example:

```java
CallableStatement statement =
        connection.prepareCall(
            "{call get_employee_count(?)}"
        );
```

Then:

```java
statement.registerOutParameter(
        1,
        Types.INTEGER
);

statement.execute();

int count =
        statement.getInt(1);
```

Whether stored procedures are appropriate depends on the application's architecture and database strategy.

---

# 46. ResultSetMetaData

Provides information about query result columns.

Example:

```java
ResultSetMetaData metadata =
        resultSet.getMetaData();

int columnCount =
        metadata.getColumnCount();
```

Then:

```java
for (int i = 1;
     i <= columnCount;
     i++) {

    System.out.println(
        metadata.getColumnName(i)
    );
}
```

Useful for:

```text
dynamic tools
database utilities
frameworks
debugging
```

---

# 47. DatabaseMetaData

Provides information about the database and driver.

```java
DatabaseMetaData metadata =
        connection.getMetaData();

System.out.println(
        metadata.getDatabaseProductName()
);

System.out.println(
        metadata.getDatabaseProductVersion()
);

System.out.println(
        metadata.getDriverName()
);
```

---

# 48. SQLException

JDBC operations commonly throw:

```java
SQLException
```

It contains information such as:

```text
message
SQLState
vendor error code
cause
```

Example:

```java
catch (SQLException e) {

    System.err.println(
        e.getMessage()
    );

    System.err.println(
        e.getSQLState()
    );
}
```

---

# 49. Don't Log Sensitive Data

Avoid:

```java
logger.error(
    "Database error for password={}",
    password
);
```

Also avoid logging:

```text
passwords
tokens
session identifiers
credit-card data
secrets
```

Production logging needs security controls.

---

# 50. Connection Pooling

Opening a database connection can be relatively expensive.

Without pooling:

```text
Request
 ↓
Create connection
 ↓
Query
 ↓
Close connection

Request
 ↓
Create connection
 ↓
Query
 ↓
Close connection
```

With pooling:

```text
              Connection Pool
           ┌─────┬─────┬─────┐
           │ C1  │ C2  │ C3  │
           └─────┴─────┴─────┘
                ↑
                |
              Request
```

Application borrows:

```text
C1
```

uses it, then returns it to the pool.

---

# 51. Why Connection Pooling?

Benefits can include:

```text
reduced connection creation overhead
controlled database concurrency
better throughput
connection reuse
centralized configuration
```

But pool sizing must match:

```text
database capacity
application concurrency
query latency
workload
```

More connections is **not automatically better**.

---

# 52. J2EE/Jakarta EE DataSource

In a managed environment, you can configure a datasource on the application server and access it through JNDI.

Conceptually:

```text
Application
    ↓
JNDI lookup
    ↓
DataSource
    ↓
Connection Pool
    ↓
Database
```

Example lookup:

```java
InitialContext context =
        new InitialContext();

DataSource dataSource =
        (DataSource) context.lookup(
            "java:comp/env/jdbc/EmployeeDB"
        );
```

Exact configuration differs between servers such as Tomcat, WildFly, Payara, etc.

---

# 53. JNDI Architecture

```text
Servlet
   |
   ↓
JNDI
   |
   ↓
DataSource
   |
   ↓
Connection Pool
   |
   ↓
Database
```

This means your application doesn't need to hardcode:

```text
database URL
username
password
```

inside application source code.

---

# 54. Production Database Configuration

Avoid:

```java
String password = "my-secret-password";
```

Prefer external configuration through appropriate deployment mechanisms:

```text
environment variables
secret managers
application-server configuration
container/orchestration secrets
managed configuration
```

---

# 55. JDBC + J2EE MVC

Now connect everything you've learned.

```text
                    Browser
                       |
                       ↓
                    Filter
                       |
                       ↓
                  Servlet
                 Controller
                       |
                       ↓
                    Service
                       |
                       ↓
                 EmployeeDAO
                       |
                       ↓
                    JDBC
                       |
                       ↓
                Connection Pool
                       |
                       ↓
                   Database
```

Response:

```text
Database
   ↓
ResultSet
   ↓
DAO
   ↓
Employee
   ↓
Service
   ↓
Servlet
   ↓
request.setAttribute()
   ↓
JSP
   ↓
EL + JSTL
   ↓
HTML
```

---

# 56. Production-Grade Employee DAO

Interface:

```java
public interface EmployeeDAO {

    Optional<Employee> findById(Long id)
            throws SQLException;

    List<Employee> findAll()
            throws SQLException;

    long save(Employee employee)
            throws SQLException;

    boolean update(Employee employee)
            throws SQLException;

    boolean delete(Long id)
            throws SQLException;
}
```

Implementation:

```java
public class JdbcEmployeeDAO
        implements EmployeeDAO {

    private final DataSource dataSource;

    public JdbcEmployeeDAO(
            DataSource dataSource) {

        this.dataSource = dataSource;
    }

    @Override
    public Optional<Employee> findById(Long id)
            throws SQLException {

        String sql =
            """
            SELECT id,
                   name,
                   department,
                   salary
            FROM employee
            WHERE id = ?
            """;

        try (Connection connection =
                 dataSource.getConnection();

             PreparedStatement statement =
                 connection.prepareStatement(sql)) {

            statement.setLong(1, id);

            try (ResultSet rs =
                     statement.executeQuery()) {

                if (!rs.next()) {
                    return Optional.empty();
                }

                return Optional.of(mapRow(rs));
            }
        }
    }

    private Employee mapRow(
            ResultSet rs)
            throws SQLException {

        Employee employee =
                new Employee();

        employee.setId(
                rs.getLong("id")
        );

        employee.setName(
                rs.getString("name")
        );

        employee.setDepartment(
                rs.getString("department")
        );

        employee.setSalary(
                rs.getBigDecimal("salary")
        );

        return employee;
    }
}
```

---

# 57. JDBC Layer Responsibilities

Your DAO should handle:

```text
SQL
PreparedStatement
ResultSet
mapping
database exceptions
resource management
```

Your DAO should generally **not** handle:

```text
HTTP
JSP
HTML
Servlet request
authentication UI
business workflow
```

---

# 58. Service Layer with DAO

```java
public class EmployeeService {

    private final EmployeeDAO employeeDAO;

    public EmployeeService(
            EmployeeDAO employeeDAO) {

        this.employeeDAO = employeeDAO;
    }

    public Employee getEmployee(Long id)
            throws SQLException {

        return employeeDAO.findById(id)
                .orElseThrow(
                    () -> new EmployeeNotFoundException(id)
                );
    }
}
```

The Service understands the application operation.

The DAO understands persistence.

---

# 59. Important JDBC Best Practices

Memorize these.

### 1. Prefer `PreparedStatement`

```text
Security + parameterization
```

### 2. Use try-with-resources

```text
Automatic cleanup
```

### 3. Use connection pooling

For production workloads.

### 4. Keep SQL out of controllers

Use DAO/repository.

### 5. Keep transactions at the business-operation boundary

Usually service/application level.

### 6. Don't hardcode secrets

Use external configuration/secret management.

### 7. Handle `SQLException` appropriately

Don't silently ignore it.

### 8. Don't leak database internals

Don't expose raw SQL/database errors to end users.

### 9. Use batch processing when appropriate

For bulk operations.

### 10. Tune connection pools carefully

Don't simply maximize pool size.

---

# 60. Common JDBC Mistakes

### Mistake 1

```java
Statement statement =
    connection.createStatement();

statement.executeQuery(
    "SELECT * FROM employee WHERE name='"
    + name
    + "'"
);
```

❌ SQL injection risk.

---

### Mistake 2

```java
Connection connection =
        DriverManager.getConnection(...);
```

for every high-volume request without pooling.

❌ Can create unnecessary overhead and database pressure.

---

### Mistake 3

Forgetting:

```java
resultSet.close();
statement.close();
connection.close();
```

❌ Resource leak.

Use try-with-resources.

---

### Mistake 4

Putting transactions inside JSP.

❌ Wrong architectural layer.

---

### Mistake 5

Doing business logic in DAO.

Example:

```java
if (employee.salary > 50000) {
    ...
}
```

❌ Usually belongs in the service/domain layer if it is a business rule.

---

# 61. JDBC Interview Questions

### Q1. What is JDBC?

Java's standard API for accessing relational databases.

---

### Q2. What is a JDBC Driver?

A database-specific implementation that enables JDBC operations to communicate with a particular database.

---

### Q3. `Statement` vs `PreparedStatement`?

`PreparedStatement` supports parameter binding and is generally preferred for application SQL involving dynamic values.

---

### Q4. Why does PreparedStatement help against SQL injection?

It separates SQL structure from parameter values rather than constructing SQL by concatenating untrusted input.

---

### Q5. What is ResultSet?

An object representing rows returned from a database query.

---

### Q6. What is DataSource?

An abstraction for obtaining database connections, commonly used with connection pools and enterprise/container-managed configuration.

---

### Q7. What is connection pooling?

Reusing a managed collection of database connections instead of repeatedly establishing new physical connections.

---

### Q8. What is auto-commit?

A connection mode where individual successful SQL statements are generally committed automatically.

---

### Q9. How do you implement a transaction?

```java
connection.setAutoCommit(false);

try {
    // operations

    connection.commit();

} catch (Exception e) {

    connection.rollback();

    throw e;
}
```

---

### Q10. What is batch processing?

Executing multiple parameterized operations as a batch to potentially reduce overhead and improve throughput.

---

### Q11. What is a Savepoint?

A marker inside a transaction to which you can roll back without necessarily rolling back the entire transaction.

---

### Q12. What is transaction isolation?

The degree to which one transaction's operations are isolated from concurrent transactions.

---

### Q13. What is DAO?

An object that encapsulates data-access operations.

---

### Q14. Why use DAO?

To separate persistence concerns from business and presentation layers.

---

### Q15. `executeQuery()` vs `executeUpdate()`?

Typically:

```text
executeQuery()  → SELECT/result set
executeUpdate() → INSERT/UPDATE/DELETE
```

---

# 62. Advanced Interview Questions

Be able to explain:

```text
1. Why is DataSource preferred in enterprise applications?
2. How does connection pooling work?
3. What happens if a connection isn't closed?
4. What happens when a pooled connection is closed?
5. How does PreparedStatement prevent SQL injection?
6. What happens internally when executeQuery() is called?
7. What is JDBC transaction management?
8. What is auto-commit?
9. What is dirty read?
10. What is non-repeatable read?
11. What is phantom read?
12. What are isolation levels?
13. What is optimistic vs pessimistic concurrency?
14. What is batch processing?
15. What are generated keys?
16. What is CallableStatement?
17. What is ResultSetMetaData?
18. What is DatabaseMetaData?
19. What is JNDI?
20. How does JNDI + DataSource + connection pool work?
```

---

# 63. Day 9 Hands-On Exercises

## Beginner

1. Connect Java to MySQL.
2. Connect Java to PostgreSQL.
3. Create an Employee table.
4. Insert one employee.
5. Select employees.
6. Update an employee.
7. Delete an employee.
8. Search employee by ID.
9. Search employee by department.
10. Count employees.

---

## Intermediate

11. Build `EmployeeDAO`.
12. Implement `findById()`.
13. Implement `findAll()`.
14. Implement `save()`.
15. Implement `update()`.
16. Implement `delete()`.
17. Add `Optional<Employee>`.
18. Add row mapper.
19. Add generated-key handling.
20. Add pagination.

---

## Advanced

21. Implement money transfer transaction.
22. Implement rollback.
23. Implement savepoints.
24. Implement batch employee insertion.
25. Test different isolation levels.
26. Configure a connection pool.
27. Configure a JNDI datasource.
28. Add centralized JDBC exception handling.
29. Measure query execution time.
30. Build a production-style JDBC DAO layer.

---

# 64. Day 9 Mini Project

## Employee Management System — JDBC Edition

Build:

```text
                    Browser
                       |
                       ↓
                    Filter
                       |
                       ↓
              EmployeeServlet
                       |
                       ↓
                EmployeeService
                       |
                       ↓
               EmployeeDAO
                       |
                       ↓
             JdbcEmployeeDAO
                       |
                       ↓
                   DataSource
                       |
                       ↓
              Connection Pool
                       |
                       ↓
                  MySQL
```

Implement:

```text
CREATE employee
READ employee
UPDATE employee
DELETE employee
SEARCH employee
LIST employees
```

Add:

```text
validation
transactions
prepared statements
connection pooling
exception handling
generated IDs
pagination
logging
```

---

# 65. Suggested Database

Create:

```sql
CREATE TABLE employee (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Sample data:

```sql
INSERT INTO employee
    (name, department, salary)
VALUES
    ('Pradeep', 'Engineering', 75000),
    ('Rahul', 'Finance', 65000),
    ('Anita', 'HR', 60000);
```

---

# 66. End-to-End Create Flow

Practice explaining this without looking:

```text
User
 ↓
POST /employees
 ↓
Authentication Filter
 ↓
EmployeeCreateServlet
 ↓
Read request parameters
 ↓
Validate input
 ↓
EmployeeCreateRequest
 ↓
EmployeeService.create()
 ↓
JdbcEmployeeDAO.save()
 ↓
DataSource
 ↓
Connection Pool
 ↓
Connection
 ↓
PreparedStatement
 ↓
INSERT
 ↓
Generated Key
 ↓
Commit
 ↓
Servlet
 ↓
Redirect
 ↓
GET /employees
 ↓
DAO.findAll()
 ↓
JSP
 ↓
EL + JSTL
 ↓
HTML
```

If you can explain this clearly, you've connected **Days 1–9** into one working architecture.

---

# 67. Day 9 Mastery Checklist

Before moving to Day 10, you should be able to explain from memory:

```text
☐ JDBC
☐ JDBC Driver
☐ Connection
☐ DriverManager
☐ DataSource
☐ Statement
☐ PreparedStatement
☐ CallableStatement
☐ ResultSet
☐ ResultSetMetaData
☐ DatabaseMetaData
☐ executeQuery()
☐ executeUpdate()
☐ execute()
☐ SQL injection
☐ Try-with-resources
☐ SQLException
☐ Transactions
☐ Commit
☐ Rollback
☐ Savepoint
☐ Auto-commit
☐ Isolation levels
☐ Batch processing
☐ Generated keys
☐ Connection pooling
☐ JNDI DataSource
☐ DAO pattern
☐ Row mapping
☐ JDBC + MVC
```

---

# 68. The Most Important Mental Model

Don't think of JDBC as simply:

```text
Java → SQL
```

Think:

```text
                    WEB
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
                 DataSource
                     |
                     ↓
              Connection Pool
                     |
                     ↓
                 Connection
                     |
                     ↓
             PreparedStatement
                     |
                     ↓
                  Database
                     |
                     ↓
                ResultSet
                     |
                     ↓
                   DAO
                     |
                     ↓
                  Service
                     |
                     ↓
                Controller
                     |
                     ↓
                    JSP
```

### Day 9 Golden Rule

> **JDBC is the low-level persistence mechanism; DAO encapsulates JDBC, Service owns business operations and transaction boundaries, and Controllers/JSPs should remain unaware of JDBC details.**

---

## Day 10 → JPA Fundamentals

Tomorrow you'll move from **manual JDBC persistence** to **JPA**, which is one of the most important transitions in enterprise Java.

You'll learn:

```text
JPA
Entity
@Entity
@Id
@GeneratedValue
EntityManager
Persistence Context
Entity Lifecycle
Managed
Detached
Transient
Removed
JPQL
Persistence Unit
ORM
Hibernate
First-level Cache
Dirty Checking
Flush
Persist
Merge
Remove
Find
```

The key comparison will be:

```text
JDBC                         JPA
────────────────────────────────────────
SQL manually                 JPQL/ORM
ResultSet                    Entity
PreparedStatement            EntityManager
DAO mapping                  Automatic mapping
Manual persistence           ORM persistence
Manual dirty handling        Dirty checking
Manual relationships         @OneToMany etc.
```

This is where your J2EE knowledge starts moving from **database connectivity** into **enterprise persistence architecture**.
