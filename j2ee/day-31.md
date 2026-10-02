Yes. If you want a **simple full-stack CRUD application using Java only**, the cleanest learning project is:

# Employee Management System

**Tech stack:**

* Java 17+
* Jakarta Servlet
* JSP
* JSTL
* JDBC
* MySQL
* Maven
* Tomcat
* HTML/CSS/JavaScript are technically needed for the browser UI, but **all application/backend logic is Java**.

Architecture:

```text
Browser
   ↓
JSP/HTML
   ↓
Servlet Controller
   ↓
Service
   ↓
DAO
   ↓
JDBC
   ↓
MySQL
```

## 1. Features

We'll implement:

* Create employee
* Read/list employees
* Read employee details
* Update employee
* Delete employee
* Search employees
* Basic validation
* MVC architecture
* JDBC database access

Example employee:

```text
id
name
email
department
salary
```

---

# 2. Project structure

```text
employee-crud/
│
├── pom.xml
│
└── src/
    └── main/
        ├── java/
        │   └── com/example/
        │       └── employee/
        │           ├── model/
        │           │   └── Employee.java
        │           │
        │           ├── dao/
        │           │   ├── EmployeeDAO.java
        │           │   └── EmployeeDAOImpl.java
        │           │
        │           ├── service/
        │           │   ├── EmployeeService.java
        │           │   └── EmployeeServiceImpl.java
        │           │
        │           ├── util/
        │           │   └── DBConnection.java
        │           │
        │           └── servlet/
        │               ├── EmployeeListServlet.java
        │               ├── EmployeeCreateServlet.java
        │               ├── EmployeeEditServlet.java
        │               └── EmployeeDeleteServlet.java
        │
        └── webapp/
            ├── index.jsp
            │
            ├── WEB-INF/
            │   └── views/
            │       ├── employee-list.jsp
            │       ├── employee-form.jsp
            │       └── employee-details.jsp
            │
            └── css/
                └── style.css
```

This is a good beginner-to-intermediate Java enterprise structure.

---

# 3. Database

Create a database:

```sql
CREATE DATABASE employee_db;

USE employee_db;
```

Create table:

```sql
CREATE TABLE employees (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    department VARCHAR(100) NOT NULL,
    salary DECIMAL(12,2) NOT NULL
);
```

Insert some test data:

```sql
INSERT INTO employees(name, email, department, salary)
VALUES
('John Doe', 'john@example.com', 'IT', 75000),
('Alice Smith', 'alice@example.com', 'HR', 65000),
('Bob Wilson', 'bob@example.com', 'Finance', 80000);
```

---

# 4. Maven `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
         http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>employee-crud</artifactId>
    <version>1.0-SNAPSHOT</version>

    <packaging>war</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>

        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>9.0.0</version>
        </dependency>

        <dependency>
            <groupId>jakarta.servlet.jsp.jstl</groupId>
            <artifactId>jakarta.servlet.jsp.jstl-api</artifactId>
            <version>3.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.glassfish.web</groupId>
            <artifactId>jakarta.servlet.jsp.jstl</artifactId>
            <version>3.0.1</version>
        </dependency>

    </dependencies>

    <build>
        <finalName>employee-crud</finalName>

        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>

</project>
```

This uses the modern Jakarta namespace.

---

# 5. Model

`Employee.java`

```java
package com.example.employee.model;

import java.math.BigDecimal;

public class Employee {

    private Long id;
    private String name;
    private String email;
    private String department;
    private BigDecimal salary;

    public Employee() {
    }

    public Employee(Long id,
                    String name,
                    String email,
                    String department,
                    BigDecimal salary) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.department = department;
        this.salary = salary;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getDepartment() {
        return department;
    }

    public void setDepartment(String department) {
        this.department = department;
    }

    public BigDecimal getSalary() {
        return salary;
    }

    public void setSalary(BigDecimal salary) {
        this.salary = salary;
    }
}
```

---

# 6. Database connection

`DBConnection.java`

```java
package com.example.employee.util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DBConnection {

    private static final String URL =
            "jdbc:mysql://localhost:3306/employee_db";

    private static final String USER = "root";

    private static final String PASSWORD = "root";

    private DBConnection() {
    }

    public static Connection getConnection()
            throws SQLException {

        return DriverManager.getConnection(
                URL,
                USER,
                PASSWORD
        );
    }
}
```

For a real production application, don't hard-code credentials.

---

# 7. DAO interface

`EmployeeDAO.java`

```java
package com.example.employee.dao;

import com.example.employee.model.Employee;

import java.util.List;
import java.util.Optional;

public interface EmployeeDAO {

    Employee save(Employee employee);

    List<Employee> findAll();

    Optional<Employee> findById(Long id);

    boolean update(Employee employee);

    boolean deleteById(Long id);
}
```

DAO = **Data Access Object**.

Its responsibility is database access.

---

# 8. DAO implementation

`EmployeeDAOImpl.java`

```java
package com.example.employee.dao;

import com.example.employee.model.Employee;
import com.example.employee.util.DBConnection;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class EmployeeDAOImpl implements EmployeeDAO {

    private static final String INSERT_SQL =
            """
            INSERT INTO employees
            (name, email, department, salary)
            VALUES (?, ?, ?, ?)
            """;

    private static final String SELECT_ALL_SQL =
            """
            SELECT id, name, email, department, salary
            FROM employees
            ORDER BY id DESC
            """;

    private static final String SELECT_BY_ID_SQL =
            """
            SELECT id, name, email, department, salary
            FROM employees
            WHERE id = ?
            """;

    private static final String UPDATE_SQL =
            """
            UPDATE employees
            SET name = ?,
                email = ?,
                department = ?,
                salary = ?
            WHERE id = ?
            """;

    private static final String DELETE_SQL =
            """
            DELETE FROM employees
            WHERE id = ?
            """;

    @Override
    public Employee save(Employee employee) {

        try (
                Connection connection =
                        DBConnection.getConnection();

                PreparedStatement statement =
                        connection.prepareStatement(
                                INSERT_SQL,
                                Statement.RETURN_GENERATED_KEYS
                        )
        ) {

            statement.setString(1, employee.getName());
            statement.setString(2, employee.getEmail());
            statement.setString(3, employee.getDepartment());
            statement.setBigDecimal(4, employee.getSalary());

            statement.executeUpdate();

            try (ResultSet rs =
                         statement.getGeneratedKeys()) {

                if (rs.next()) {
                    employee.setId(rs.getLong(1));
                }
            }

            return employee;

        } catch (SQLException e) {
            throw new RuntimeException(
                    "Unable to create employee", e);
        }
    }

    @Override
    public List<Employee> findAll() {

        List<Employee> employees = new ArrayList<>();

        try (
                Connection connection =
                        DBConnection.getConnection();

                PreparedStatement statement =
                        connection.prepareStatement(
                                SELECT_ALL_SQL);

                ResultSet rs =
                        statement.executeQuery()
        ) {

            while (rs.next()) {
                employees.add(mapRow(rs));
            }

            return employees;

        } catch (SQLException e) {
            throw new RuntimeException(
                    "Unable to fetch employees", e);
        }
    }

    @Override
    public Optional<Employee> findById(Long id) {

        try (
                Connection connection =
                        DBConnection.getConnection();

                PreparedStatement statement =
                        connection.prepareStatement(
                                SELECT_BY_ID_SQL)
        ) {

            statement.setLong(1, id);

            try (ResultSet rs =
                         statement.executeQuery()) {

                if (rs.next()) {
                    return Optional.of(mapRow(rs));
                }

                return Optional.empty();
            }

        } catch (SQLException e) {
            throw new RuntimeException(
                    "Unable to find employee", e);
        }
    }

    @Override
    public boolean update(Employee employee) {

        try (
                Connection connection =
                        DBConnection.getConnection();

                PreparedStatement statement =
                        connection.prepareStatement(
                                UPDATE_SQL)
        ) {

            statement.setString(1, employee.getName());
            statement.setString(2, employee.getEmail());
            statement.setString(3, employee.getDepartment());
            statement.setBigDecimal(4, employee.getSalary());
            statement.setLong(5, employee.getId());

            return statement.executeUpdate() > 0;

        } catch (SQLException e) {
            throw new RuntimeException(
                    "Unable to update employee", e);
        }
    }

    @Override
    public boolean deleteById(Long id) {

        try (
                Connection connection =
                        DBConnection.getConnection();

                PreparedStatement statement =
                        connection.prepareStatement(
                                DELETE_SQL)
        ) {

            statement.setLong(1, id);

            return statement.executeUpdate() > 0;

        } catch (SQLException e) {
            throw new RuntimeException(
                    "Unable to delete employee", e);
        }
    }

    private Employee mapRow(ResultSet rs)
            throws SQLException {

        return new Employee(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("email"),
                rs.getString("department"),
                rs.getBigDecimal("salary")
        );
    }
}
```

Notice the use of:

```java
PreparedStatement
```

instead of string-concatenated SQL.

That's important for SQL injection protection.

---

# 9. Service layer

`EmployeeService.java`

```java
package com.example.employee.service;

import com.example.employee.model.Employee;

import java.util.List;
import java.util.Optional;

public interface EmployeeService {

    Employee create(Employee employee);

    List<Employee> getAll();

    Optional<Employee> getById(Long id);

    boolean update(Employee employee);

    boolean delete(Long id);
}
```

Implementation:

```java
package com.example.employee.service;

import com.example.employee.dao.EmployeeDAO;
import com.example.employee.dao.EmployeeDAOImpl;
import com.example.employee.model.Employee;

import java.util.List;
import java.util.Optional;

public class EmployeeServiceImpl
        implements EmployeeService {

    private final EmployeeDAO employeeDAO;

    public EmployeeServiceImpl() {
        this.employeeDAO =
                new EmployeeDAOImpl();
    }

    @Override
    public Employee create(Employee employee) {

        validate(employee);

        return employeeDAO.save(employee);
    }

    @Override
    public List<Employee> getAll() {
        return employeeDAO.findAll();
    }

    @Override
    public Optional<Employee> getById(Long id) {
        return employeeDAO.findById(id);
    }

    @Override
    public boolean update(Employee employee) {

        validate(employee);

        return employeeDAO.update(employee);
    }

    @Override
    public boolean delete(Long id) {

        if (id == null || id <= 0) {
            throw new IllegalArgumentException(
                    "Invalid employee ID");
        }

        return employeeDAO.deleteById(id);
    }

    private void validate(Employee employee) {

        if (employee.getName() == null ||
                employee.getName().isBlank()) {

            throw new IllegalArgumentException(
                    "Name is required");
        }

        if (employee.getEmail() == null ||
                employee.getEmail().isBlank()) {

            throw new IllegalArgumentException(
                    "Email is required");
        }

        if (employee.getDepartment() == null ||
                employee.getDepartment().isBlank()) {

            throw new IllegalArgumentException(
                    "Department is required");
        }

        if (employee.getSalary() == null ||
                employee.getSalary().signum() < 0) {

            throw new IllegalArgumentException(
                    "Salary must be positive");
        }
    }
}
```

---

# 10. List employees servlet

`EmployeeListServlet.java`

```java
package com.example.employee.servlet;

import com.example.employee.model.Employee;
import com.example.employee.service.EmployeeService;
import com.example.employee.service.EmployeeServiceImpl;

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;

import java.io.IOException;
import java.util.List;

@WebServlet("/employees")
public class EmployeeListServlet
        extends HttpServlet {

    private final EmployeeService employeeService =
            new EmployeeServiceImpl();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        List<Employee> employees =
                employeeService.getAll();

        request.setAttribute(
                "employees",
                employees);

        request.getRequestDispatcher(
                "/WEB-INF/views/employee-list.jsp")
                .forward(request, response);
    }
}
```

---

# 11. Create employee servlet

`EmployeeCreateServlet.java`

```java
package com.example.employee.servlet;

import com.example.employee.model.Employee;
import com.example.employee.service.EmployeeService;
import com.example.employee.service.EmployeeServiceImpl;

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;

import java.io.IOException;
import java.math.BigDecimal;

@WebServlet("/employees/create")
public class EmployeeCreateServlet
        extends HttpServlet {

    private final EmployeeService employeeService =
            new EmployeeServiceImpl();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        request.getRequestDispatcher(
                "/WEB-INF/views/employee-form.jsp")
                .forward(request, response);
    }

    @Override
    protected void doPost(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        String name =
                request.getParameter("name");

        String email =
                request.getParameter("email");

        String department =
                request.getParameter("department");

        BigDecimal salary =
                new BigDecimal(
                        request.getParameter("salary"));

        Employee employee =
                new Employee(
                        null,
                        name,
                        email,
                        department,
                        salary
                );

        employeeService.create(employee);

        response.sendRedirect(
                request.getContextPath()
                        + "/employees");
    }
}
```

This implements the **POST → Redirect → GET** pattern.

---

# 12. Edit servlet

`EmployeeEditServlet.java`

```java
package com.example.employee.servlet;

import com.example.employee.model.Employee;
import com.example.employee.service.EmployeeService;
import com.example.employee.service.EmployeeServiceImpl;

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;

import java.io.IOException;
import java.math.BigDecimal;

@WebServlet("/employees/edit")
public class EmployeeEditServlet
        extends HttpServlet {

    private final EmployeeService employeeService =
            new EmployeeServiceImpl();

    @Override
    protected void doGet(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        Long id = Long.parseLong(
                request.getParameter("id"));

        Employee employee =
                employeeService.getById(id)
                        .orElseThrow(
                                () -> new ServletException(
                                        "Employee not found"));

        request.setAttribute(
                "employee",
                employee);

        request.getRequestDispatcher(
                "/WEB-INF/views/employee-form.jsp")
                .forward(request, response);
    }

    @Override
    protected void doPost(
            HttpServletRequest request,
            HttpServletResponse response)
            throws ServletException, IOException {

        Long id = Long.parseLong(
                request.getParameter("id"));

        Employee employee =
                new Employee(
                        id,
                        request.getParameter("name"),
                        request.getParameter("email"),
                        request.getParameter("department"),
                        new BigDecimal(
                                request.getParameter("salary"))
                );

        employeeService.update(employee);

        response.sendRedirect(
                request.getContextPath()
                        + "/employees");
    }
}
```

---

# 13. Delete servlet

`EmployeeDeleteServlet.java`

```java
package com.example.employee.servlet;

import com.example.employee.service.EmployeeService;
import com.example.employee.service.EmployeeServiceImpl;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;

import java.io.IOException;

@WebServlet("/employees/delete")
public class EmployeeDeleteServlet
        extends HttpServlet {

    private final EmployeeService employeeService =
            new EmployeeServiceImpl();

    @Override
    protected void doPost(
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        Long id = Long.parseLong(
                request.getParameter("id"));

        employeeService.delete(id);

        response.sendRedirect(
                request.getContextPath()
                        + "/employees");
    }
}
```

Using POST for delete is preferable to exposing destructive actions through a simple GET link.

---

# 14. Employee list JSP

`employee-list.jsp`

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ taglib prefix="c"
           uri="jakarta.tags.core" %>

<!DOCTYPE html>

<html>

<head>

    <title>Employees</title>

    <link rel="stylesheet"
          href="${pageContext.request.contextPath}/css/style.css">

</head>

<body>

<h1>Employee Management</h1>

<a href="${pageContext.request.contextPath}/employees/create">
    Add Employee
</a>

<br><br>

<table border="1">

    <thead>

    <tr>
        <th>ID</th>
        <th>Name</th>
        <th>Email</th>
        <th>Department</th>
        <th>Salary</th>
        <th>Actions</th>
    </tr>

    </thead>

    <tbody>

    <c:forEach var="employee"
               items="${employees}">

        <tr>

            <td>${employee.id}</td>

            <td>${employee.name}</td>

            <td>${employee.email}</td>

            <td>${employee.department}</td>

            <td>${employee.salary}</td>

            <td>

                <a href="${pageContext.request.contextPath}
                    /employees/edit?id=${employee.id}">
                    Edit
                </a>

                <form method="post"
                      action="${pageContext.request.contextPath}/employees/delete"
                      style="display:inline;">

                    <input type="hidden"
                           name="id"
                           value="${employee.id}">

                    <button type="submit">
                        Delete
                    </button>

                </form>

            </td>

        </tr>

    </c:forEach>

    </tbody>

</table>

</body>

</html>
```

---

# 15. Employee form

`employee-form.jsp`

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>

<!DOCTYPE html>

<html>

<head>

    <title>
        ${employee == null ? "Create Employee" : "Edit Employee"}
    </title>

</head>

<body>

<h1>
    ${employee == null ? "Create Employee" : "Edit Employee"}
</h1>

<form method="post"
      action="${pageContext.request.contextPath}/employees/${employee == null ? 'create' : 'edit'}">

    <input type="hidden"
           name="id"
           value="${employee.id}">

    <label>Name:</label>

    <input type="text"
           name="name"
           value="${employee.name}"
           required>

    <br><br>

    <label>Email:</label>

    <input type="email"
           name="email"
           value="${employee.email}"
           required>

    <br><br>

    <label>Department:</label>

    <input type="text"
           name="department"
           value="${employee.department}"
           required>

    <br><br>

    <label>Salary:</label>

    <input type="number"
           step="0.01"
           name="salary"
           value="${employee.salary}"
           required>

    <br><br>

    <button type="submit">
        Save
    </button>

</form>

<br>

<a href="${pageContext.request.contextPath}/employees">
    Back
</a>

</body>

</html>
```

---

# 16. `index.jsp`

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>

<%

    response.sendRedirect(
            request.getContextPath()
                    + "/employees"
    );

%>
```

---

# 17. CSS

`style.css`

```css
body {
    font-family: Arial, sans-serif;
    margin: 40px;
}

table {
    border-collapse: collapse;
    width: 80%;
}

th,
td {
    padding: 10px;
    border: 1px solid #ccc;
}

th {
    background: #f2f2f2;
}

input {
    padding: 8px;
    margin: 5px;
}

button {
    padding: 8px 15px;
}

a {
    text-decoration: none;
}
```

---

# 18. Run the application

Build:

```bash
mvn clean package
```

You'll get:

```text
target/employee-crud.war
```

Deploy that WAR to **Tomcat 10+**.

Then open:

```text
http://localhost:8080/employee-crud/employees
```

You'll get:

```text
Employee Management

+----+------------+-------------------+------------+--------+----------------+
| ID | Name       | Email             | Department | Salary | Actions        |
+----+------------+-------------------+------------+--------+----------------+
| 3  | Bob Wilson | bob@example.com   | Finance    | 80000  | Edit | Delete   |
| 2  | Alice      | alice@example.com | HR         | 65000  | Edit | Delete   |
| 1  | John Doe   | john@example.com  | IT         | 75000  | Edit | Delete   |
+----+------------+-------------------+------------+--------+----------------+

[ Add Employee ]
```

---

# 19. Complete CRUD flow

### Create

```text
Browser
   ↓
GET /employees/create
   ↓
CreateEmployeeServlet
   ↓
JSP Form
   ↓
POST /employees/create
   ↓
EmployeeCreateServlet
   ↓
EmployeeService
   ↓
EmployeeDAO
   ↓
JDBC
   ↓
MySQL
   ↓
Redirect /employees
```

### Read

```text
GET /employees
      ↓
EmployeeListServlet
      ↓
EmployeeService
      ↓
EmployeeDAO
      ↓
JDBC
      ↓
MySQL
      ↓
employees
      ↓
JSP
      ↓
HTML
```

### Update

```text
GET /employees/edit?id=1
        ↓
Load employee
        ↓
Display form
        ↓
POST /employees/edit
        ↓
Service
        ↓
DAO
        ↓
UPDATE employees
        ↓
Redirect
```

### Delete

```text
POST /employees/delete
        ↓
Servlet
        ↓
Service
        ↓
DAO
        ↓
DELETE FROM employees
        ↓
Redirect
```

---

# 20. What you are actually learning

This tiny application teaches you the fundamental Java enterprise stack:

| Layer            | Technology     |
| ---------------- | -------------- |
| UI               | JSP + HTML/CSS |
| Controller       | Servlet        |
| Business         | Service        |
| Persistence      | DAO            |
| Database access  | JDBC           |
| Database         | MySQL          |
| Dependency/build | Maven          |
| Runtime          | Tomcat         |
| Packaging        | WAR            |
| Architecture     | MVC            |

And the most important mental model is:

```text
                 BROWSER
                    │
                    ▼
              ┌───────────┐
              │    JSP    │
              │    View   │
              └─────┬─────┘
                    │
                    ▼
              ┌───────────┐
              │  Servlet  │
              │ Controller│
              └─────┬─────┘
                    │
                    ▼
              ┌───────────┐
              │  Service  │
              │   Layer   │
              └─────┬─────┘
                    │
                    ▼
              ┌───────────┐
              │    DAO    │
              │   Layer   │
              └─────┬─────┘
                    │
                    ▼
              ┌───────────┐
              │   JDBC    │
              └─────┬─────┘
                    │
                    ▼
              ┌───────────┐
              │   MySQL   │
              └───────────┘
```

This is an excellent **first Java-only full-stack project** before moving to Spring Boot.

The natural next evolution is:

```text
Version 1
Servlet + JSP + JDBC
       ↓
Version 2
Servlet + JSP + JPA
       ↓
Version 3
Jakarta REST API + JSP
       ↓
Version 4
Spring Boot + REST + JPA
       ↓
Version 5
Spring Boot + React
       ↓
Version 6
Spring Boot Microservices
       ↓
Version 7
Cloud + Docker + Kubernetes
```

For your Java/J2EE mastery path, I would specifically build this application **phase-by-phase**, adding authentication, filters, validation, JPA, REST, transactions, JMS, Docker, testing, and deployment rather than jumping directly to Spring Boot.
