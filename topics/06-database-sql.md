# 06 — Database & SQL

> SQL Server focused. Covers normalization, indexing, transactions, and T-SQL.

---

## Table of Contents
1. [Normalization](#1-normalization)
2. [Indexing](#2-indexing)
3. [Transactions & ACID](#3-transactions--acid)
4. [Isolation Levels](#4-isolation-levels)
5. [Joins](#5-joins)
6. [Window Functions](#6-window-functions)
7. [CTEs and Subqueries](#7-ctes-and-subqueries)
8. [Query Optimization](#8-query-optimization)
9. [Stored Procedures vs Functions](#9-stored-procedures-vs-functions)
10. [Common Interview Questions](#10-common-interview-questions)

---

## 1. Normalization

Normalization reduces data redundancy and improves integrity. Understand **1NF through BCNF**.

### 1NF — First Normal Form
Rules:
- Atomic values only (no repeating groups, no arrays)
- Each column has a single value
- Each row is unique (has a primary key)

```sql
-- ❌ VIOLATES 1NF — multiple values in Tags column
| TransactionId | Description | Tags              |
|---------------|-------------|-------------------|
| 1             | Salary      | income, monthly   |  ← not atomic!

-- ✅ 1NF — separate table for tags
| TransactionId | Description |
| 1             | Salary      |

| TransactionId | Tag     |
| 1             | income  |
| 1             | monthly |
```

### 2NF — Second Normal Form
Rules: Must be in 1NF + **no partial dependencies** (every non-key attribute must depend on the ENTIRE composite key).

```sql
-- ❌ VIOLATES 2NF — OrderDate depends only on OrderId (partial dependency)
| OrderId | ProductId | Quantity | OrderDate  | ProductName |
| 1       | 101       | 5        | 2024-01-15 | iPhone 15   |
-- OrderDate depends only on OrderId (not the full composite key OrderId+ProductId)
-- ProductName depends only on ProductId

-- ✅ 2NF — split into Orders, OrderItems, Products
Orders:     (OrderId, OrderDate, CustomerId)
OrderItems: (OrderId, ProductId, Quantity)    ← composite key, all cols depend on both
Products:   (ProductId, ProductName, Price)
```

### 3NF — Third Normal Form
Rules: Must be in 2NF + **no transitive dependencies** (non-key attributes must NOT depend on other non-key attributes).

```sql
-- ❌ VIOLATES 3NF — ZipCode → City, State (transitive dependency)
| EmployeeId | Name  | ZipCode | City      | State |
-- City and State depend on ZipCode (non-key), not directly on EmployeeId

-- ✅ 3NF
Employees: (EmployeeId, Name, ZipCode)
ZipCodes:  (ZipCode, City, State)
```

### BCNF — Boyce-Codd Normal Form
Stricter version of 3NF. Every determinant must be a candidate key.

### BudgetPH Normalization Examples
```sql
-- BudgetPH correctly normalized:
-- Category → CategoryType (not stored as string in Transaction)
-- Account → AccountType (enum, not string)
-- User → Accounts (1:many) — no user data duplicated in Account table
-- Budget → BudgetItems (1:many) — no budget data in BudgetItem
```

---

## 2. Indexing

### Clustered vs Non-Clustered

| Type | Stores | How many | Rule |
|------|--------|----------|------|
| **Clustered** | Actual data rows | 1 per table | Usually the Primary Key |
| **Non-Clustered** | Pointer + selected columns | Up to 999 | For search/filter columns |

```sql
-- Clustered index (usually auto-created with PRIMARY KEY)
CREATE CLUSTERED INDEX IX_Transactions_Id ON Transactions(Id);

-- Non-clustered index — speeds up filtering by UserId
CREATE NONCLUSTERED INDEX IX_Transactions_UserId 
ON Transactions(UserId);

-- Composite index — for queries filtering on both columns
CREATE NONCLUSTERED INDEX IX_Transactions_User_Date 
ON Transactions(UserId, TransactionDate DESC);

-- Covering index — includes extra columns to avoid key lookup
CREATE NONCLUSTERED INDEX IX_Transactions_User_Date_Covering
ON Transactions(UserId, TransactionDate DESC)
INCLUDE (Amount, Description, TransactionType);
-- ↑ queries selecting these columns won't need to hit the clustered index
```

### Index Guidelines

```sql
-- ✅ Good candidates for indexes:
-- Columns in WHERE clauses
WHERE t.UserId = @UserId AND t.TransactionDate >= @StartDate

-- Foreign key columns (EF Core does NOT auto-create FK indexes)
CREATE INDEX IX_Transactions_AccountId ON Transactions(AccountId);

-- Columns in ORDER BY
ORDER BY TransactionDate DESC

-- ❌ Bad index candidates:
-- Low-cardinality columns (IsActive, Gender, Status) — few distinct values
-- Frequently updated columns — index maintenance overhead
-- Small tables — full scan is faster than index lookup

-- Find missing indexes (SQL Server DMVs)
SELECT 
    DB_NAME(mid.database_id) AS DatabaseName,
    OBJECT_NAME(mid.object_id) AS TableName,
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * 
        (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX IX_' + OBJECT_NAME(mid.object_id) + '_' +
        REPLACE(REPLACE(REPLACE(ISNULL(mid.equality_columns,''),', ','_'),'[',''),']','') 
        + CASE WHEN mid.inequality_columns IS NULL THEN '' 
               ELSE '_' + REPLACE(REPLACE(REPLACE(mid.inequality_columns,', ','_'),'[',''),']','') 
          END AS create_index_statement
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY improvement_measure DESC;
```

---

## 3. Transactions & ACID

### ACID Properties

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All or nothing | Transfer: debit AND credit, or neither |
| **Consistency** | DB moves from valid state to valid state | Balance can't go negative |
| **Isolation** | Concurrent transactions don't interfere | Two users updating same account |
| **Durability** | Committed changes survive failures | Power outage after commit |

```sql
-- Explicit transaction (T-SQL)
BEGIN TRANSACTION;
    BEGIN TRY
        UPDATE Accounts SET Balance = Balance - 5000 WHERE Id = @FromId;
        UPDATE Accounts SET Balance = Balance + 5000 WHERE Id = @ToId;
        
        INSERT INTO Transactions (AccountId, Amount, TransactionType, TransactionDate)
        VALUES (@FromId, 5000, 3, GETUTCDATE()); -- 3 = Transfer
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW; -- re-raise the error
    END CATCH;

-- Savepoints — partial rollback
BEGIN TRANSACTION;
    INSERT INTO Accounts (...) VALUES (...);
    SAVE TRANSACTION AccountCreated; -- savepoint
    
    INSERT INTO Notifications (...) VALUES (...);
    IF @@ERROR <> 0
        ROLLBACK TRANSACTION AccountCreated; -- roll back only to savepoint
    
COMMIT TRANSACTION;
```

---

## 4. Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| **Read Uncommitted** | ✅ possible | ✅ possible | ✅ possible |
| **Read Committed** (default) | ❌ prevented | ✅ possible | ✅ possible |
| **Repeatable Read** | ❌ prevented | ❌ prevented | ✅ possible |
| **Serializable** | ❌ prevented | ❌ prevented | ❌ prevented |
| **Snapshot** | ❌ prevented | ❌ prevented | ❌ prevented |

**Definitions:**
- **Dirty Read**: Reading data another transaction hasn't committed yet
- **Non-Repeatable Read**: Same SELECT returns different data within the same transaction (because another transaction updated a row)
- **Phantom Read**: Same query returns different rows (because another transaction inserted/deleted rows)

```sql
-- Set isolation level for a session
SET TRANSACTION ISOLATION LEVEL READ COMMITTED; -- default

-- Snapshot isolation (optimistic concurrency — good for read-heavy workloads)
ALTER DATABASE BudgetPH_Dev SET ALLOW_SNAPSHOT_ISOLATION ON;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

---

## 5. Joins

```sql
-- Sample tables: Transactions, Categories, Accounts

-- INNER JOIN — only matching rows from both sides
SELECT t.Description, t.Amount, c.Name AS Category
FROM Transactions t
INNER JOIN Categories c ON t.CategoryId = c.Id;

-- LEFT JOIN — all from left, matching from right (NULL if no match)
SELECT t.Description, t.Amount, c.Name AS Category
FROM Transactions t
LEFT JOIN Categories c ON t.CategoryId = c.Id;
-- ↑ includes transactions with no category (CategoryId is NULL)

-- RIGHT JOIN — all from right, matching from left
SELECT t.Description, c.Name
FROM Transactions t
RIGHT JOIN Categories c ON t.CategoryId = c.Id;
-- ↑ includes categories with no transactions

-- FULL OUTER JOIN — all from both sides
SELECT t.Description, c.Name
FROM Transactions t
FULL OUTER JOIN Categories c ON t.CategoryId = c.Id;

-- CROSS JOIN — cartesian product (every row × every row)
SELECT a.Name, b.Name
FROM Accounts a
CROSS JOIN Accounts b; -- all possible account pairs

-- Self JOIN — joining a table to itself
SELECT c.Name, p.Name AS ParentCategory
FROM Categories c
LEFT JOIN Categories p ON c.ParentCategoryId = p.Id;
```

---

## 6. Window Functions

Window functions perform calculations across **related rows without collapsing them** (unlike GROUP BY).

```sql
-- Syntax: FUNCTION() OVER (PARTITION BY col ORDER BY col)

-- ROW_NUMBER — unique sequential number per partition
SELECT 
    UserId,
    TransactionDate,
    Amount,
    ROW_NUMBER() OVER (PARTITION BY UserId ORDER BY TransactionDate DESC) AS RowNum
FROM Transactions;
-- Row 1 = most recent transaction per user

-- RANK vs DENSE_RANK
-- RANK: 1,2,2,4 (gap after ties)
-- DENSE_RANK: 1,2,2,3 (no gaps)
SELECT 
    UserId,
    Amount,
    RANK()       OVER (PARTITION BY UserId ORDER BY Amount DESC) AS Rank,
    DENSE_RANK() OVER (PARTITION BY UserId ORDER BY Amount DESC) AS DenseRank
FROM Transactions;

-- Running total (cumulative sum)
SELECT 
    TransactionDate,
    Amount,
    SUM(Amount) OVER (PARTITION BY UserId ORDER BY TransactionDate 
                      ROWS UNBOUNDED PRECEDING) AS RunningBalance
FROM Transactions
WHERE UserId = @UserId AND TransactionType = 1; -- Income only

-- LAG / LEAD — access previous/next row values
SELECT 
    Month,
    TotalExpenses,
    LAG(TotalExpenses, 1) OVER (ORDER BY Month) AS PreviousMonthExpenses,
    TotalExpenses - LAG(TotalExpenses, 1) OVER (ORDER BY Month) AS MonthOverMonthChange
FROM MonthlyExpenses;

-- NTILE — divide into N equal groups (percentiles)
SELECT 
    TransactionId,
    Amount,
    NTILE(4) OVER (ORDER BY Amount) AS Quartile -- 1=lowest 25%, 4=highest 25%
FROM Transactions;

-- Top N per group (using ROW_NUMBER)
WITH RankedTransactions AS (
    SELECT *, 
        ROW_NUMBER() OVER (PARTITION BY AccountId ORDER BY Amount DESC) AS rn
    FROM Transactions
)
SELECT * FROM RankedTransactions WHERE rn <= 3; -- top 3 per account
```

---

## 7. CTEs and Subqueries

### CTE (Common Table Expression)
```sql
-- Simple CTE — makes complex queries readable
WITH MonthlyTotals AS (
    SELECT 
        UserId,
        YEAR(TransactionDate) AS Year,
        MONTH(TransactionDate) AS Month,
        SUM(CASE WHEN TransactionType = 1 THEN Amount ELSE 0 END) AS TotalIncome,
        SUM(CASE WHEN TransactionType = 2 THEN Amount ELSE 0 END) AS TotalExpenses
    FROM Transactions
    GROUP BY UserId, YEAR(TransactionDate), MONTH(TransactionDate)
)
SELECT 
    UserId, Year, Month,
    TotalIncome, TotalExpenses,
    TotalIncome - TotalExpenses AS NetSavings
FROM MonthlyTotals
WHERE Year = 2026
ORDER BY UserId, Month;

-- Recursive CTE — category hierarchy
WITH CategoryHierarchy AS (
    -- Anchor: top-level categories
    SELECT Id, Name, ParentCategoryId, 0 AS Level, CAST(Name AS NVARCHAR(500)) AS Path
    FROM Categories
    WHERE ParentCategoryId IS NULL
    
    UNION ALL
    
    -- Recursive: child categories
    SELECT c.Id, c.Name, c.ParentCategoryId, h.Level + 1, 
           CAST(h.Path + ' > ' + c.Name AS NVARCHAR(500))
    FROM Categories c
    INNER JOIN CategoryHierarchy h ON c.ParentCategoryId = h.Id
)
SELECT * FROM CategoryHierarchy ORDER BY Path;
```

### Subqueries

```sql
-- Correlated subquery (runs once per outer row — can be slow)
SELECT t.Description, t.Amount,
    (SELECT c.Name FROM Categories c WHERE c.Id = t.CategoryId) AS CategoryName
FROM Transactions t;
-- Better: use JOIN instead

-- EXISTS (efficient — stops scanning when first match found)
SELECT Name FROM Users u
WHERE EXISTS (
    SELECT 1 FROM Transactions t 
    WHERE t.UserId = u.Id AND t.Amount > 10000
);

-- NOT EXISTS
SELECT Name FROM Categories c
WHERE NOT EXISTS (
    SELECT 1 FROM Transactions t WHERE t.CategoryId = c.Id
);

-- IN vs EXISTS: prefer EXISTS for large datasets (better performance)
-- IN: executes inner query once, creates list
-- EXISTS: checks row by row, stops at first match
```

---

## 8. Query Optimization

```sql
-- 1. Use EXPLAIN / Execution Plan (SSMS: Ctrl+M or Ctrl+L)
-- Look for: Table Scan (bad), Index Scan (ok), Index Seek (good)

-- 2. Avoid SELECT * — specify columns
-- ❌ SELECT * FROM Transactions  (fetches all columns)
-- ✅ SELECT Id, Amount, TransactionDate FROM Transactions

-- 3. SARGable predicates (Search ARGument ABLE)
-- ❌ NOT sargable — can't use index
WHERE YEAR(TransactionDate) = 2026        -- function on column = no index
WHERE Amount * 1.12 > 1000               -- expression on column = no index
WHERE CONVERT(VARCHAR, UserId) = '123'   -- type conversion = no index

-- ✅ Sargable — can use index
WHERE TransactionDate >= '2026-01-01' AND TransactionDate < '2027-01-01'
WHERE Amount > 1000 / 1.12
WHERE UserId = CAST('123' AS UNIQUEIDENTIFIER)

-- 4. Pagination — OFFSET/FETCH (avoid ROW_NUMBER for large tables)
SELECT Id, Description, Amount
FROM Transactions
WHERE UserId = @UserId
ORDER BY TransactionDate DESC
OFFSET (@Page - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;

-- 5. Avoid cursors — use set-based operations
-- ❌ CURSOR — row by row
-- ✅ UPDATE with JOIN or CTEs

-- 6. Statistics update
UPDATE STATISTICS Transactions;
-- Or: EXEC sp_updatestats; -- all tables
```

---

## 9. Stored Procedures vs Functions

| | Stored Procedure | Scalar Function | Table-Valued Function |
|---|---|---|---|
| Returns | Nothing, single value, or resultset | Single scalar value | Table |
| Can modify data | ✅ Yes | ❌ No | ❌ No |
| Can call in SELECT | ❌ No | ✅ Yes | ✅ Yes |
| Can use transactions | ✅ Yes | ❌ No | ❌ No |
| Output parameters | ✅ Yes | ❌ | ❌ |

```sql
-- Stored Procedure
CREATE PROCEDURE sp_GetMonthlyReport
    @UserId UNIQUEIDENTIFIER,
    @Year INT,
    @Month INT
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        c.Name AS Category,
        SUM(t.Amount) AS Total,
        COUNT(*) AS TransactionCount
    FROM Transactions t
    INNER JOIN Categories c ON t.CategoryId = c.Id
    WHERE t.UserId = @UserId 
        AND YEAR(t.TransactionDate) = @Year 
        AND MONTH(t.TransactionDate) = @Month
    GROUP BY c.Name
    ORDER BY Total DESC;
END;

-- Calling it
EXEC sp_GetMonthlyReport @UserId = 'guid-here', @Year = 2026, @Month = 7;

-- Table-Valued Function (inline)
CREATE FUNCTION dbo.fn_GetUserTransactions(@UserId UNIQUEIDENTIFIER, @Top INT)
RETURNS TABLE AS RETURN
(
    SELECT TOP (@Top) 
        t.Id, t.Description, t.Amount, t.TransactionDate, c.Name AS Category
    FROM Transactions t
    LEFT JOIN Categories c ON t.CategoryId = c.Id
    WHERE t.UserId = @UserId
    ORDER BY t.TransactionDate DESC
);

-- Use in SELECT
SELECT * FROM dbo.fn_GetUserTransactions('guid-here', 10);
```

---

## 10. Common Interview Questions

**Q1: What is the difference between WHERE and HAVING?**

> `WHERE` filters rows **before** grouping — it works on individual rows. `HAVING` filters **after** `GROUP BY` — it works on aggregated results. Use `WHERE` to filter raw data; use `HAVING` to filter aggregated groups. `WHERE` is generally faster because it reduces rows before aggregation.

**Q2: What is a covering index?**

> A covering index is a non-clustered index that includes all columns needed by a query — both the search columns (key columns) and the additional columns being selected (via `INCLUDE`). This lets the query engine satisfy the entire query using just the index, without doing a key lookup back to the clustered index (heap). This dramatically reduces I/O.

**Q3: Explain the difference between TRUNCATE and DELETE.**

> `DELETE` is logged (each row deletion is logged), can have a `WHERE` clause, can be rolled back, fires DELETE triggers. `TRUNCATE` removes all rows, is minimally logged, cannot have a `WHERE` clause, cannot be rolled back (in SQL Server outside a transaction), and does NOT fire triggers. `TRUNCATE` is much faster for clearing a table.

**Q4: What is a deadlock and how do you prevent it?**

> A deadlock occurs when two transactions each hold a lock that the other needs. SQL Server automatically detects deadlocks and kills the "deadlock victim" transaction. Prevention: (1) Access resources in the same order across transactions, (2) Keep transactions short, (3) Use `NOLOCK` hint for read-heavy analytics (accepts dirty reads), (4) Use Snapshot Isolation, (5) Use proper indexing to reduce lock duration.

**Q5: What is index fragmentation and how do you fix it?**

> Fragmentation occurs when index pages are out of order due to INSERT/UPDATE/DELETE operations. Logical fragmentation > 30%: rebuild the index (`ALTER INDEX ... REBUILD`). 10–30%: reorganize (`ALTER INDEX ... REORGANIZE`). Below 10%: leave it. SQL Server Agent jobs typically run weekly/monthly index maintenance.

**Q6: What is the N+1 query problem in SQL?**

> Occurs when you fetch N parent records and then execute 1 additional query per parent to get related data — N+1 total queries instead of 1. Fix with JOINs or EF Core `Include()`. Example: fetching 100 transactions, then querying category name for each one = 101 queries instead of 1 JOIN.

**Q7: Explain the difference between UNION and UNION ALL.**

> `UNION ALL` — returns all rows from both queries, including duplicates (faster). `UNION` — returns distinct rows only, removing duplicates (slower — requires a sort/hash). Always use `UNION ALL` when you know there are no duplicates or duplicates are acceptable.
