# Database-Question---SQL-
# SQL Server Interview Preparation

> **Target Role:** React.js + .NET + SQL Developer
> **Purpose:** Interview revision and repeated speaking practice
> **Database:** Microsoft SQL Server

---

# 1. How do you optimize a Stored Procedure?

## Interview Answer

> "I optimize a stored procedure by first identifying where the performance bottleneck is. I usually check the execution plan, logical reads, CPU time and execution time. Then I optimize the query by making sure appropriate indexes exist, avoiding unnecessary columns and data, reducing unnecessary joins, using proper filtering, and avoiding functions on indexed columns when possible. I also check for parameter sniffing, outdated statistics and unnecessary cursors or loops. Finally, I compare the execution metrics before and after the changes to verify that the optimization actually improved performance."

## Deep Explanation

A stored procedure itself is not automatically fast just because it is stored in the database.

The queries **inside** the stored procedure determine most of its performance.

### Things I check:

### 1. Execution Plan

Check whether SQL Server is doing:

* Index Seek
* Index Scan
* Table Scan
* Key Lookup
* Expensive Sort
* Hash Match
* Nested Loop
* Missing index recommendation

Example:

```sql
EXEC sp_GetEmployees @Department = 'IT';
```

Then enable the actual execution plan in SSMS.

---

### 2. Avoid `SELECT *`

Bad:

```sql
SELECT *
FROM Employees
WHERE Department = 'IT';
```

Better:

```sql
SELECT EmployeeId, Name, Salary
FROM Employees
WHERE Department = 'IT';
```

Why?

Because `SELECT *` may retrieve unnecessary columns and increase I/O and network traffic.

---

### 3. Use appropriate indexes

Suppose:

```sql
SELECT *
FROM Employees
WHERE Department = 'IT';
```

If `Department` is frequently used for filtering, an index may help:

```sql
CREATE INDEX IX_Employees_Department
ON Employees(Department);
```

---

### 4. Avoid functions on indexed columns

Instead of:

```sql
WHERE YEAR(HireDate) = 2025
```

prefer a range:

```sql
WHERE HireDate >= '2025-01-01'
  AND HireDate < '2026-01-01'
```

This can allow SQL Server to use an index on `HireDate` more effectively.

---

### 5. Avoid unnecessary loops/cursors

SQL is designed to work with sets.

Prefer:

```sql
UPDATE Employees
SET Salary = Salary * 1.10
WHERE Department = 'IT';
```

instead of processing employees one by one using a cursor.

---

### 6. Check parameter sniffing

Sometimes SQL Server creates an execution plan based on the first parameter value passed to a stored procedure.

That plan may not be optimal for other parameter values.

Example:

```sql
CREATE PROCEDURE GetEmployees
    @Department NVARCHAR(50)
AS
BEGIN
    SELECT *
    FROM Employees
    WHERE Department = @Department;
END;
```

If one department contains 10 rows and another contains 100,000 rows, the same cached plan may not always be ideal.

---

## Interview Follow-up

**Interviewer:** "How do you know whether your optimization actually worked?"

Answer:

> "I compare the execution plan, execution time, CPU time and logical reads before and after the change. I don't consider a query optimized just because it looks cleaner; I verify the actual performance improvement."

## Memory Trick

**Plan → Index → Query → Statistics → Parameters → Measure**

---

# 2. How do you measure Stored Procedure execution time?

## Interview Answer

> "In SQL Server, I can measure stored procedure performance using execution time, CPU time and logical reads. In SSMS, I commonly use `SET STATISTICS TIME ON` and `SET STATISTICS IO ON`. I can also check the actual execution plan to understand where the time and I/O are being spent."

## Example

```sql
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

EXEC GetEmployees;

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
```

SQL Server will show information such as:

```text
SQL Server Execution Times:
CPU time = 10 ms
elapsed time = 15 ms
```

And IO information such as:

```text
logical reads
physical reads
read-ahead reads
```

## What do these mean?

### CPU Time

Time spent by CPU processing the query.

### Elapsed Time

Total wall-clock time taken.

For example:

```text
CPU time = 100 ms
Elapsed time = 500 ms
```

This could indicate that the query spent time waiting for I/O, locks, network, etc.

### Logical Reads

Number of pages read from SQL Server's buffer cache.

Logical reads are especially useful when comparing query versions.

## Interview Follow-up

**Interviewer:** "Would you only look at execution time?"

> "No. Execution time can vary depending on server load and other factors. I also look at CPU time, logical reads and the execution plan to understand why the query is slow."

## Memory Trick

**TIME = CPU + Elapsed**

**IO = Logical Reads + Physical Reads**

---

# 3. Clustered Index vs Non-Clustered Index

## Interview Answer

> "A clustered index determines the physical order of rows in a table, so a table can have only one clustered index. A non-clustered index is a separate data structure that stores indexed values along with a reference to the actual row, so a table can have multiple non-clustered indexes."

## Clustered Index

Think of a dictionary.

The words are physically arranged alphabetically.

Similarly, a clustered index determines how the table's rows are organized.

Example:

```sql
CREATE CLUSTERED INDEX IX_Employees_EmployeeId
ON Employees(EmployeeId);
```

A table can have **only one clustered index** because the rows cannot be physically ordered in multiple ways at the same time.

---

## Non-Clustered Index

Think of a book's index.

The index tells you:

> "Search this keyword → go to this page."

Similarly:

```sql
CREATE NONCLUSTERED INDEX IX_Employees_Name
ON Employees(Name);
```

SQL Server can have multiple non-clustered indexes.

---

## Example

Suppose:

```text
Employees

EmployeeId   Name       Salary
1            Aman       50000
2            Rahul      60000
3            Priya      70000
```

Clustered index:

```text
EmployeeId → actual table rows
```

Non-clustered index on `Name`:

```text
Aman   → row reference
Priya  → row reference
Rahul  → row reference
```

## Important Point

A clustered index is not simply "the primary key."

A primary key **usually** creates a clustered index by default in SQL Server, but it doesn't have to.

You can create a primary key as:

```sql
PRIMARY KEY NONCLUSTERED
```

## Memory Trick

**Clustered = Data organized**

**Non-clustered = Separate lookup structure**

**One clustered, many non-clustered**

---

# 4. How do Transactions Work?

## Interview Answer

> "A transaction is a logical unit of work containing one or more SQL operations. The purpose of a transaction is to ensure that either all required operations succeed or the changes are rolled back. SQL Server provides commands such as `BEGIN TRANSACTION`, `COMMIT` and `ROLLBACK` to control transactions."

## Example

Suppose we transfer ₹5,000 from Account A to Account B.

Two operations are required:

```sql
UPDATE Accounts
SET Balance = Balance - 5000
WHERE AccountId = 1;

UPDATE Accounts
SET Balance = Balance + 5000
WHERE AccountId = 2;
```

Both operations should succeed together.

```sql
BEGIN TRANSACTION;

UPDATE Accounts
SET Balance = Balance - 5000
WHERE AccountId = 1;

UPDATE Accounts
SET Balance = Balance + 5000
WHERE AccountId = 2;

COMMIT TRANSACTION;
```

If something goes wrong:

```sql
ROLLBACK TRANSACTION;
```

---

## TRY/CATCH Example

```sql
BEGIN TRY

    BEGIN TRANSACTION;

    UPDATE Accounts
    SET Balance = Balance - 5000
    WHERE AccountId = 1;

    UPDATE Accounts
    SET Balance = Balance + 5000
    WHERE AccountId = 2;

    COMMIT TRANSACTION;

END TRY
BEGIN CATCH

    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    THROW;

END CATCH;
```

## Interview Follow-up

**Why use transactions?**

> "To maintain data consistency when multiple related operations must succeed or fail together."

## Memory Trick

**BEGIN → Work → COMMIT**

**Error → ROLLBACK**

---

# 5. Function vs Stored Procedure

## Interview Answer

> "A function is generally used to calculate or return a value and can be used inside SQL expressions such as SELECT, WHERE or JOIN depending on the function type. A stored procedure is mainly used to perform operations or execute a set of SQL statements and can return result sets and output parameters. A function must return a value, while a stored procedure does not have that requirement."

## Function

Example:

```sql
CREATE FUNCTION dbo.GetBonus
(
    @Salary DECIMAL(10,2)
)
RETURNS DECIMAL(10,2)
AS
BEGIN
    RETURN @Salary * 0.10;
END;
```

Call:

```sql
SELECT
    Name,
    Salary,
    dbo.GetBonus(Salary) AS Bonus
FROM Employees;
```

---

## Stored Procedure

```sql
CREATE PROCEDURE GetEmployees
    @Department NVARCHAR(50)
AS
BEGIN
    SELECT *
    FROM Employees
    WHERE Department = @Department;
END;
```

Call:

```sql
EXEC GetEmployees 'IT';
```

## Key Differences

| Function                                            | Stored Procedure                                           |
| --------------------------------------------------- | ---------------------------------------------------------- |
| Must return a value/table                           | Does not have to return a value                            |
| Can be used in SELECT expressions depending on type | Usually executed with EXEC                                 |
| Commonly used for calculations/reusable logic       | Commonly used for operations/business processes            |
| Can be used in queries                              | Cannot simply be embedded in SELECT like a scalar function |
| Generally has more restrictions                     | More procedural capabilities                               |

## Memory Trick

**Function = Calculate/Return**

**Procedure = Perform/Process**

---

# 6. Stored Procedure vs View

## Interview Answer

> "A view is a virtual table based on a SELECT query and is mainly used to simplify or secure data access. A stored procedure is a programmable database object that can contain multiple SQL statements, parameters, conditional logic and transactions. A view is queried like a table, while a stored procedure is executed."

## View

```sql
CREATE VIEW ITEmployees
AS
SELECT EmployeeId, Name, Salary
FROM Employees
WHERE Department = 'IT';
```

Use:

```sql
SELECT *
FROM ITEmployees;
```

---

## Stored Procedure

```sql
CREATE PROCEDURE GetEmployeesByDepartment
    @Department NVARCHAR(50)
AS
BEGIN
    SELECT *
    FROM Employees
    WHERE Department = @Department;
END;
```

Use:

```sql
EXEC GetEmployeesByDepartment 'IT';
```

## Memory Trick

**View = Saved SELECT**

**Procedure = Saved Program**

---

# 7. Find the 5th Highest Salary

## Approach 1 — DENSE_RANK

If duplicate salaries should have the same rank:

```sql
SELECT Salary
FROM
(
    SELECT
        Salary,
        DENSE_RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM Employees
) E
WHERE SalaryRank = 5;
```

Example:

```text
100000
90000
90000
80000
70000
60000
```

Ranks:

```text
100000 → 1
90000  → 2
90000  → 2
80000  → 3
70000  → 4
60000  → 5
```

Therefore:

```text
5th highest distinct salary = 60000
```

## Why DENSE_RANK?

Because the question usually means the **5th distinct highest salary**.

## Alternative

```sql
SELECT DISTINCT Salary
FROM Employees
ORDER BY Salary DESC
OFFSET 4 ROWS
FETCH NEXT 1 ROW ONLY;
```

## Interview Follow-up

**What if there are duplicate salaries?**

> "I would clarify whether the interviewer wants the 5th row or the 5th distinct salary. For the 5th distinct salary, I prefer `DENSE_RANK()`."

---

# 8. Find the Top 3 Highest Salaries

## Distinct Top 3 Salaries

```sql
SELECT DISTINCT TOP 3 Salary
FROM Employees
ORDER BY Salary DESC;
```

If you want all employees who belong to the top 3 salary levels:

```sql
SELECT *
FROM
(
    SELECT
        *,
        DENSE_RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM Employees
) E
WHERE SalaryRank <= 3;
```

## Important Difference

```sql
TOP 3
```

means three rows.

```sql
DENSE_RANK() <= 3
```

means the first three **salary levels**.

---

# 9. What are SQL Constraints?

## Interview Answer

> "Constraints are rules applied to table columns to maintain data integrity and prevent invalid data from being inserted or updated."

## Main SQL Server Constraints

### 1. PRIMARY KEY

Uniquely identifies each row.

```sql
EmployeeId INT PRIMARY KEY
```

Properties:

* Unique
* Cannot be NULL
* One primary key constraint per table

---

### 2. FOREIGN KEY

Maintains a relationship between tables.

```sql
DepartmentId INT
FOREIGN KEY REFERENCES Departments(DepartmentId)
```

---

### 3. UNIQUE

Prevents duplicate values.

```sql
Email VARCHAR(100) UNIQUE
```

---

### 4. NOT NULL

Prevents NULL values.

```sql
Name VARCHAR(100) NOT NULL
```

---

### 5. CHECK

Enforces a condition.

```sql
Salary DECIMAL(10,2)
CHECK (Salary > 0)
```

---

### 6. DEFAULT

Provides a value when one isn't supplied.

```sql
Status VARCHAR(20) DEFAULT 'Active'
```

## Memory Trick

**P F U N C D**

**Primary → Foreign → Unique → Not Null → Check → Default**

---

# 10. DELETE vs TRUNCATE vs DROP

| Feature                         | DELETE                   | TRUNCATE                              | DROP                                    |
| ------------------------------- | ------------------------ | ------------------------------------- | --------------------------------------- |
| Removes rows                    | Yes                      | Yes                                   | Entire object                           |
| WHERE allowed                   | Yes                      | No                                    | No                                      |
| Removes table structure         | No                       | No                                    | Yes                                     |
| Can rollback inside transaction | Yes                      | Yes                                   | Yes, when executed within a transaction |
| Logging                         | Row-level logging        | Minimal/deallocation-oriented logging | Object removal is logged                |
| Identity reset                  | No                       | Usually yes                           | Object removed                          |
| Triggers                        | DELETE triggers can fire | DELETE triggers don't fire            | Table removed                           |
| Purpose                         | Selective deletion       | Remove all rows quickly               | Remove table/object                     |

### DELETE

```sql
DELETE FROM Employees
WHERE Department = 'IT';
```

Deletes selected rows.

### TRUNCATE

```sql
TRUNCATE TABLE Employees;
```

Removes all rows while keeping the table structure.

### DROP

```sql
DROP TABLE Employees;
```

Removes the table itself.

## Memory Trick

**DELETE = rows selectively**

**TRUNCATE = all rows**

**DROP = entire object**

---

# 11. Why can DELETE be rolled back but TRUNCATE usually cannot?

## Important Correction

A common interview statement is:

> "DELETE can be rolled back but TRUNCATE cannot."

That is **not technically correct for SQL Server**.

`TRUNCATE TABLE` **can be rolled back if it is executed inside an explicit transaction**.

Example:

```sql
BEGIN TRANSACTION;

TRUNCATE TABLE Employees;

ROLLBACK TRANSACTION;
```

The rows can be restored by the rollback.

## Then what is the real difference?

`DELETE` logs individual row deletions, while `TRUNCATE` uses fewer log records and deallocates data pages/extents rather than logging every deleted row individually.

Therefore, `TRUNCATE` is generally faster for removing all rows.

## Interview Answer

> "The common statement that TRUNCATE cannot be rolled back is a misconception in SQL Server. TRUNCATE can be rolled back when it is executed inside a transaction. The major difference is that DELETE logs row-level deletions, while TRUNCATE uses minimal logging and deallocates the data pages. DELETE also supports a WHERE clause, whereas TRUNCATE removes all rows."

This answer can actually impress an interviewer because you are correcting a common misconception.

---

# 12. What is CTE?

## Interview Answer

> "CTE stands for Common Table Expression. It is a temporary named result set that exists only for the duration of a single SQL statement. It improves query readability and is especially useful for complex queries, recursive queries and operations involving window functions."

## Syntax

```sql
WITH EmployeeCTE AS
(
    SELECT
        EmployeeId,
        Name,
        Salary
    FROM Employees
)
SELECT *
FROM EmployeeCTE;
```

## Example — Employees with Salary > 50,000

```sql
WITH HighSalaryEmployees AS
(
    SELECT *
    FROM Employees
    WHERE Salary > 50000
)
SELECT *
FROM HighSalaryEmployees;
```

## CTE with Window Function

```sql
WITH RankedEmployees AS
(
    SELECT
        Name,
        Salary,
        DENSE_RANK() OVER (ORDER BY Salary DESC) AS SalaryRank
    FROM Employees
)
SELECT *
FROM RankedEmployees
WHERE SalaryRank <= 3;
```

## Important

A CTE is **not automatically a physical temporary table**.

It is mainly a way to define a named query expression for the following statement.

## Memory Trick

**CTE = temporary named query for one statement**

---

# 13. Temp Table vs Table Variable

## Temp Table

```sql
CREATE TABLE #Employees
(
    Id INT,
    Name VARCHAR(100)
);
```

Use:

```sql
INSERT INTO #Employees
VALUES (1, 'Aman');
```

Temp table:

```text
#Employees
```

---

## Table Variable

```sql
DECLARE @Employees TABLE
(
    Id INT,
    Name VARCHAR(100)
);
```

Use:

```sql
INSERT INTO @Employees
VALUES (1, 'Aman');
```

## Comparison

| Temp Table                                                              | Table Variable                           |
| ----------------------------------------------------------------------- | ---------------------------------------- |
| `#Table`                                                                | `@Table`                                 |
| Created in tempdb                                                       | Also uses tempdb internally              |
| Better suited to larger/intermediate datasets                           | Often convenient for smaller datasets    |
| Supports indexes and statistics with more flexibility                   | More limited statistics behavior         |
| Can be useful across multiple statements in a batch/procedure           | Scope is more limited                    |
| Generally preferred when optimizer needs better cardinality information | Good for small/simple temporary datasets |

## Interview Answer

> "I generally use a temp table when I have a larger intermediate dataset or when I need indexes and better optimizer information. For a small temporary dataset or simple procedural operation, a table variable can be convenient. The choice depends on workload rather than simply saying one is always faster."

---

# 14. Find Duplicate Records using CTE

Suppose duplicate employees are identified by email.

```sql
WITH DuplicateEmployees AS
(
    SELECT
        Email,
        COUNT(*) AS DuplicateCount
    FROM Employees
    GROUP BY Email
    HAVING COUNT(*) > 1
)
SELECT *
FROM DuplicateEmployees;
```

## If you want complete duplicate rows

```sql
WITH DuplicateEmployees AS
(
    SELECT
        *,
        COUNT(*) OVER (PARTITION BY Email) AS DuplicateCount
    FROM Employees
)
SELECT *
FROM DuplicateEmployees
WHERE DuplicateCount > 1;
```

## Memory Trick

**GROUP BY + HAVING COUNT(*) > 1**

---

# 15. ROW_NUMBER vs RANK vs DENSE_RANK

This is a very common interview question.

Suppose salaries are:

```text
100000
90000
90000
80000
70000
```

## ROW_NUMBER()

```sql
ROW_NUMBER() OVER (ORDER BY Salary DESC)
```

Result:

```text
100000 → 1
90000  → 2
90000  → 3
80000  → 4
70000  → 5
```

Every row gets a unique number.

---

## RANK()

```sql
RANK() OVER (ORDER BY Salary DESC)
```

Result:

```text
100000 → 1
90000  → 2
90000  → 2
80000  → 4
70000  → 5
```

Duplicates receive the same rank, and gaps are created.

---

## DENSE_RANK()

```sql
DENSE_RANK() OVER (ORDER BY Salary DESC)
```

Result:

```text
100000 → 1
90000  → 2
90000  → 2
80000  → 3
70000  → 4
```

Duplicates receive the same rank, but there are no gaps.

## Memory Trick

**ROW_NUMBER = No duplicates in numbering**

**RANK = Same rank + gaps**

**DENSE_RANK = Same rank + no gaps**

---

# 16. EXISTS vs IN

## Interview Answer

> "`IN` checks whether a value exists in a set of values, while `EXISTS` checks whether the subquery returns at least one row. `EXISTS` is often useful for correlated subqueries because it can stop checking once a matching row is found."

## IN Example

```sql
SELECT *
FROM Employees
WHERE DepartmentId IN
(
    SELECT DepartmentId
    FROM Departments
    WHERE Location = 'Noida'
);
```

---

## EXISTS Example

```sql
SELECT *
FROM Employees E
WHERE EXISTS
(
    SELECT 1
    FROM Departments D
    WHERE D.DepartmentId = E.DepartmentId
      AND D.Location = 'Noida'
);
```

## Important Interview Point

Don't say:

> "EXISTS is always faster than IN."

That is too absolute.

SQL Server's optimizer may transform the queries into similar execution strategies depending on the query and data.

## Memory Trick

**IN = Is this value in the set?**

**EXISTS = Does at least one matching row exist?**

---

# 17. INNER JOIN vs LEFT JOIN vs RIGHT JOIN

## INNER JOIN

Returns only matching rows.

```sql
SELECT
    E.Name,
    D.DepartmentName
FROM Employees E
INNER JOIN Departments D
    ON E.DepartmentId = D.DepartmentId;
```

Think:

```text
A ∩ B
```

---

## LEFT JOIN

Returns:

* All rows from left table
* Matching rows from right table
* NULL when no match exists

```sql
SELECT
    E.Name,
    D.DepartmentName
FROM Employees E
LEFT JOIN Departments D
    ON E.DepartmentId = D.DepartmentId;
```

Useful when you want:

> "All employees, even employees without a department."

---

## RIGHT JOIN

Returns all rows from the right table and matching rows from the left table.

```sql
SELECT
    E.Name,
    D.DepartmentName
FROM Employees E
RIGHT JOIN Departments D
    ON E.DepartmentId = D.DepartmentId;
```

## Memory Trick

**INNER = Matching**

**LEFT = Everything from left**

**RIGHT = Everything from right**

---

# 18. UNION vs UNION ALL

## UNION

Combines result sets and removes duplicate rows.

```sql
SELECT Name FROM Employees
UNION
SELECT Name FROM Managers;
```

---

## UNION ALL

Combines result sets without removing duplicates.

```sql
SELECT Name FROM Employees
UNION ALL
SELECT Name FROM Managers;
```

## Performance

`UNION` generally requires additional work to remove duplicates.

Therefore, when duplicates are acceptable or impossible:

```sql
UNION ALL
```

is generally preferable.

## Memory Trick

**UNION = Combine + Remove duplicates**

**UNION ALL = Combine everything**

---

# 19. What is Normalization?

## Interview Answer

> "Normalization is the process of organizing data into related tables to reduce data redundancy and improve data integrity. Instead of storing the same information repeatedly, we separate entities into appropriate tables and establish relationships using keys."

## Example

Bad design:

```text
StudentId | StudentName | Course1 | Course2 | Course3
```

Better:

### Students

```text
StudentId | StudentName
```

### Courses

```text
CourseId | CourseName
```

### StudentCourses

```text
StudentId | CourseId
```

Now the relationships are clear.

## Common Normal Forms

### 1NF

* Atomic values
* No repeating groups

### 2NF

* 1NF
* No partial dependency on part of a composite key

### 3NF

* 2NF
* No transitive dependency

## Why normalize?

* Reduce duplicate data
* Improve consistency
* Reduce update anomalies
* Improve data integrity

## Memory Trick

**Normalization = Remove unnecessary duplication**

---

# 20. What is Denormalization?

## Interview Answer

> "Denormalization is intentionally introducing some redundancy into a database to improve read performance or simplify queries. It is usually considered when the cost of joins is significant and read performance is more important than eliminating all redundancy."

Example:

Normalized:

```text
Orders
CustomerId

Customers
CustomerId
CustomerName
```

A denormalized Orders table might also store:

```text
OrderId
CustomerId
CustomerName
Amount
```

Now the customer name is duplicated.

## Advantage

Fewer joins and potentially faster reads.

## Disadvantage

* Duplicate data
* More storage
* More complex updates
* Possible inconsistency

## Memory Trick

**Normalization = Less redundancy**

**Denormalization = Intentional redundancy for performance/read simplicity**

---

# 21. What are ACID Properties?

ACID describes transaction properties.

## A — Atomicity

All operations succeed or all fail.

Example:

```text
Debit account
Credit account
```

Both should happen, or neither should happen.

---

## C — Consistency

Transaction takes the database from one valid state to another valid state.

Constraints and rules must remain satisfied.

---

## I — Isolation

Concurrent transactions should not improperly interfere with each other.

SQL Server provides isolation levels such as:

* Read Uncommitted
* Read Committed
* Repeatable Read
* Serializable
* Snapshot

---

## D — Durability

Once a transaction is committed, its changes should survive failures.

## Interview Answer

> "ACID stands for Atomicity, Consistency, Isolation and Durability. Together, these properties ensure that database transactions are reliable and maintain data integrity."

## Memory Trick

**A = All or Nothing**

**C = Correct State**

**I = Independent**

**D = Doesn't disappear after commit**

---

# 22. What is a Deadlock?

## Interview Answer

> "A deadlock occurs when two or more transactions are waiting for resources locked by each other, so none of them can continue. SQL Server detects the deadlock and chooses one transaction as the deadlock victim, rolling it back so the other transaction can continue."

## Example

Transaction 1:

```text
Locks Row A
Waits for Row B
```

Transaction 2:

```text
Locks Row B
Waits for Row A
```

Now:

```text
T1 → waits for T2
T2 → waits for T1
```

Deadlock.

## How to Reduce Deadlocks?

1. Access tables/resources in consistent order.
2. Keep transactions short.
3. Avoid unnecessary locks.
4. Use appropriate indexes.
5. Avoid user interaction inside transactions.
6. Investigate deadlock graphs.

## Interview Follow-up

**Is a blocking situation always a deadlock?**

No.

Blocking:

```text
T1 → holds lock
T2 → waits
```

Once T1 finishes, T2 can continue.

Deadlock:

```text
T1 → waits for T2
T2 → waits for T1
```

Neither can continue without intervention.

## Memory Trick

**Blocking = Waiting**

**Deadlock = Waiting in a circle**

---

# 23. How do you read an Execution Plan?

## Interview Answer

> "I use the execution plan to understand how SQL Server is executing a query. I look for expensive operators, table scans, index scans, key lookups, joins, sorts, estimated versus actual rows, warnings and missing-index suggestions. I focus on understanding the root cause rather than simply choosing the operator with the highest percentage."

## Important Operators

### Index Seek

Usually means SQL Server can efficiently locate required rows.

### Index Scan

SQL Server reads many or all index pages.

Not automatically bad.

### Table Scan

Reads the table.

Can be reasonable for a very small table.

### Key Lookup

SQL Server finds rows using a non-clustered index but must go back to the clustered index/table to retrieve additional columns.

Frequent expensive key lookups may indicate that a covering index could help.

### Sort

Sorting data can be expensive for large datasets.

### Hash Match

Common for joins/aggregations.

### Nested Loops

Often useful when one input is small and the other side can be efficiently accessed.

## Estimated vs Actual Rows

Suppose:

```text
Estimated Rows = 10
Actual Rows = 100,000
```

That large mismatch may indicate poor cardinality estimation, outdated statistics, data distribution issues, or query-related problems.

## Memory Trick

**Scan → Seek → Lookup → Join → Sort → Rows**

---

# 24. What is Index Fragmentation?

## Interview Answer

> "Index fragmentation occurs when the logical ordering of index pages does not match the physical ordering or when pages have excessive free space due to modifications. It can increase I/O and reduce efficiency, especially for range scans. SQL Server provides index rebuild and reorganize operations to address fragmentation when appropriate."

## Why does it happen?

Suppose:

```text
Page 1
Page 2
Page 3
```

After many INSERT/UPDATE/DELETE operations, pages can become less efficiently organized.

This can cause:

* More page reads
* Poor range-scan efficiency
* Additional I/O

## Reorganize

```sql
ALTER INDEX IX_Employees_Name
ON Employees
REORGANIZE;
```

Generally a lighter, online operation.

## Rebuild

```sql
ALTER INDEX IX_Employees_Name
ON Employees
REBUILD;
```

Recreates the index and can be more resource-intensive.

## Important

Don't blindly rebuild every index.

Check:

* fragmentation
* index usage
* workload
* page count
* maintenance window
* SQL Server version/features

## Memory Trick

**Reorganize = lighter maintenance**

**Rebuild = recreate index**

---

# 25. What are Window Functions?

## Interview Answer

> "Window functions perform calculations across a set of related rows without collapsing those rows into a single result like GROUP BY does. They are commonly used for ranking, running totals, comparisons with previous or next rows and partition-based calculations."

## Example

```sql
SELECT
    Name,
    Department,
    Salary,
    ROW_NUMBER() OVER
    (
        PARTITION BY Department
        ORDER BY Salary DESC
    ) AS RowNum
FROM Employees;
```

Here:

```sql
PARTITION BY Department
```

creates a separate logical group for each department.

```sql
ORDER BY Salary DESC
```

determines ranking within each department.

## Running Total

```sql
SELECT
    OrderDate,
    Amount,
    SUM(Amount) OVER
    (
        ORDER BY OrderDate
    ) AS RunningTotal
FROM Orders;
```

## Common Window Functions

* ROW_NUMBER()
* RANK()
* DENSE_RANK()
* SUM()
* AVG()
* COUNT()
* LAG()
* LEAD()

## Memory Trick

**GROUP BY = collapse rows**

**Window Function = calculate while keeping rows**

---

# 26. What is a Cursor?

## Interview Answer

> "A cursor is a database object used to process query results row by row. It is useful in scenarios where row-by-row processing is genuinely required, but it is generally avoided for large datasets because set-based SQL operations are usually more efficient."

## Example

```sql
DECLARE EmployeeCursor CURSOR FOR
SELECT EmployeeId
FROM Employees;

OPEN EmployeeCursor;

FETCH NEXT FROM EmployeeCursor;

-- Processing

CLOSE EmployeeCursor;

DEALLOCATE EmployeeCursor;
```

## Why avoid cursors?

Suppose there are:

```text
1,000,000 rows
```

A cursor may process them one by one.

A set-based query can often process the operation more efficiently.

Instead of:

```text
Row 1 → process
Row 2 → process
Row 3 → process
...
```

prefer:

```sql
UPDATE Employees
SET Salary = Salary * 1.10
WHERE Department = 'IT';
```

## Interview Answer

> "I would use a cursor only when the business logic genuinely requires sequential row-by-row processing and cannot be reasonably expressed using set-based operations."

---

# 27. How do you improve SQL Query Performance?

## Interview Answer

> "I start by measuring the query rather than guessing. I check the actual execution plan, CPU time, logical reads and execution time. Then I look at indexing, filtering, joins, statistics, unnecessary columns, data volume and query structure. I also check for issues such as parameter sniffing, implicit conversions, blocking and excessive key lookups. After making a change, I compare the metrics again."

## Practical Checklist

### 1. Check Execution Plan

```text
What is SQL Server actually doing?
```

### 2. Check Indexes

Ask:

```text
Can SQL Server efficiently find the required rows?
```

### 3. Avoid `SELECT *`

Return only required columns.

### 4. Filter Early

```sql
WHERE Department = 'IT'
```

### 5. Optimize Joins

Make sure join columns are appropriate and indexed when useful.

### 6. Avoid Functions on Indexed Columns

Instead of:

```sql
WHERE YEAR(HireDate) = 2025
```

use:

```sql
WHERE HireDate >= '2025-01-01'
AND HireDate < '2026-01-01'
```

### 7. Avoid Unnecessary DISTINCT

`DISTINCT` can introduce additional sorting/hashing work.

### 8. Avoid Unnecessary Cursors

Prefer set-based operations.

### 9. Check Statistics

SQL Server's optimizer relies heavily on statistics.

### 10. Check Blocking/Deadlocks

A query may be slow because it is waiting, not because its execution itself is inefficient.

## Strong Interview Answer

> "I don't optimize SQL queries based only on syntax. I first measure the problem, inspect the execution plan and identify whether the bottleneck is CPU, I/O, locking, poor cardinality estimation or query design. Then I make a targeted change and measure again."

This is a **very strong interview answer**.

---

# 28. Primary Key vs Unique Key

## Interview Answer

> "Both primary key and unique constraints enforce uniqueness, but the primary key is the main identifier of a row and cannot contain NULL values. A table can have only one primary key, while it can have multiple unique constraints. A unique constraint is commonly used for alternate candidate keys such as email addresses."

## Example

```sql
CREATE TABLE Employees
(
    EmployeeId INT PRIMARY KEY,
    Email VARCHAR(100) UNIQUE,
    Name VARCHAR(100)
);
```

Here:

```text
EmployeeId → Primary Key
Email      → Unique Key
```

## Difference

| Primary Key                     | Unique                                                                                                 |
| ------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Main row identifier             | Enforces uniqueness                                                                                    |
| Only one primary key constraint | Multiple unique constraints possible                                                                   |
| Cannot contain NULL             | NULL handling differs; SQL Server permits NULL in a unique constraint, subject to its uniqueness rules |
| Often used for relationships    | Often used for alternate unique values                                                                 |

## Memory Trick

**Primary = Main identity**

**Unique = Another value must be unique**

---

# 29. Foreign Key vs Composite Key

This is an important point because these two concepts are **not alternatives**.

## Foreign Key

A foreign key establishes a relationship between tables.

Example:

```sql
CREATE TABLE Departments
(
    DepartmentId INT PRIMARY KEY,
    DepartmentName VARCHAR(100)
);

CREATE TABLE Employees
(
    EmployeeId INT PRIMARY KEY,
    Name VARCHAR(100),
    DepartmentId INT,

    FOREIGN KEY (DepartmentId)
        REFERENCES Departments(DepartmentId)
);
```

Here:

```text
Departments.DepartmentId
        ↑
        |
Employees.DepartmentId
```

The foreign key maintains referential integrity.

---

# Composite Key

A composite key consists of **two or more columns together** that uniquely identify a row.

Example:

```sql
CREATE TABLE StudentCourses
(
    StudentId INT,
    CourseId INT,

    PRIMARY KEY (StudentId, CourseId)
);
```

Here:

```text
StudentId + CourseId
```

together identify one record.

For example:

```text
StudentId | CourseId
----------|---------
1         | 101
1         | 102
2         | 101
```

`StudentId` alone is not unique.

`CourseId` alone is not unique.

But:

```text
(StudentId, CourseId)
```

is unique.

---

## Can a Composite Key also be a Foreign Key?

Yes.

For example:

```sql
CREATE TABLE StudentCourses
(
    StudentId INT,
    CourseId INT,

    PRIMARY KEY (StudentId, CourseId),

    FOREIGN KEY (StudentId)
        REFERENCES Students(StudentId),

    FOREIGN KEY (CourseId)
        REFERENCES Courses(CourseId)
);
```

## Interview Answer

> "A foreign key defines a relationship between tables and references a key in another table. A composite key is a key made up of multiple columns that together uniquely identify a row. They solve different problems, and a composite key can also participate in foreign-key relationships."

## Memory Trick

**Foreign Key = Relationship**

**Composite Key = Multiple columns form one key**

---

# Quick Revision Sheet

## Stored Procedure Optimization

```text
Execution Plan
      ↓
Indexes
      ↓
Query Structure
      ↓
Statistics
      ↓
Parameter Sniffing
      ↓
CPU / IO / Locks
      ↓
Measure Again
```

---

## DELETE / TRUNCATE / DROP

```text
DELETE     → Remove selected rows
TRUNCATE   → Remove all rows
DROP       → Remove object
```

---

## Ranking Functions

```text
ROW_NUMBER
1
2
3
4

RANK
1
2
2
4

DENSE_RANK
1
2
2
3
```

---

## Joins

```text
INNER → Matching rows

LEFT → All left + matching right

RIGHT → All right + matching left
```

---

## UNION

```text
UNION     → Remove duplicates
UNION ALL → Keep duplicates
```

---

## Keys

```text
PRIMARY KEY  → Main identity
UNIQUE       → Unique value
FOREIGN KEY  → Relationship
COMPOSITE    → Multiple columns together form key
```

---

## Transactions

```text
BEGIN TRANSACTION
       ↓
     Work
       ↓
Success? ───── Yes → COMMIT
       |
       No
       ↓
    ROLLBACK
```

---

## ACID

```text
A → Atomicity   → All or nothing
C → Consistency → Valid state
I → Isolation   → Transactions don't improperly interfere
D → Durability  → Committed data survives failures
```

---

# How to Answer SQL Questions in an Interview

Don't immediately jump into SQL syntax.

Use this structure:

### Step 1 — Definition

> "A clustered index determines the ordering of rows..."

### Step 2 — Why

> "The main benefit is efficient retrieval based on the indexed key..."

### Step 3 — Example

```sql
CREATE CLUSTERED INDEX ...
```

### Step 4 — Real-world usage

> "For example, if I frequently search employees by EmployeeId..."

### Step 5 — Follow-up

> "One important point is that a table can have only one clustered index..."

This makes your answer sound much more experienced than simply giving a definition.

---


