# 30x SQL Performance Tips Markdown

>Note: The original SQL file was converted to Markdown to enhance readability and make the content more accessible.

This document outlines best practices for optimizing SQL performance, covering fetching data, filtering, joins, UNION, aggregations, subqueries/CTE, DDL, and indexing. Techniques include selecting only necessary columns, proper filtering methods, explicit joins, avoiding redundant logic, and efficient indexing strategies.

## Table of Contents

1. Fetching Data
2. Filtering
3. Joins
4. Union
5. Aggregations
6. Subqueries, CTE
7. DDL
8. Indexing

---

## Fetching Data

### Tip 1: Select Only What You Need

Avoid using `SELECT *` to reduce unnecessary data retrieval.

**Bad Practice:**

```sql
SELECT * FROM Sales.Customers;
```

**Good Practice:**

```sql
SELECT CustomerID, FirstName, LastName FROM Sales.Customers;
```

---

### Tip 2: Avoid Unnecessary DISTINCT & ORDER BY

Unnecessary `DISTINCT` and `ORDER BY` clauses can degrade performance.

**Bad Practice:**

```sql
SELECT DISTINCT FirstName 
FROM Sales.Customers 
ORDER BY FirstName;
```

**Good Practice:**

```sql
SELECT FirstName 
FROM Sales.Customers;
```

---

### Tip 3: For Exploration Purpose, Limit Rows!

Limit the number of rows returned when exploring data to improve performance.

**Bad Practice:**

```sql
SELECT OrderID, Sales 
FROM Sales.Orders;
```

**Good Practice:**

```sql
SELECT TOP 10 OrderID, Sales 
FROM Sales.Orders;
```

---

## Filtering

### Tip 4: Create Nonclustered Index on Frequently Used Columns in WHERE Clause

Indexes on frequently filtered columns improve query performance.

```sql
SELECT *
FROM Sales.Orders
WHERE OrderStatus = 'Delivered';
```

```sql
CREATE NONCLUSTERED INDEX Idx_Orders_OrderStatus ON Sales.Orders(OrderStatus);
```

---

### Tip 5: Avoid Applying Functions to Columns in WHERE Clauses

Functions on columns in `WHERE` clauses prevent index usage.

**Bad Practice:**

```sql
SELECT * FROM Sales.Orders 
WHERE LOWER(OrderStatus) = 'delivered';
```

**Good Practice:**

```sql
SELECT * FROM Sales.Orders 
WHERE OrderStatus = 'Delivered';
```

**Bad Practice:**

```sql
SELECT * 
FROM Sales.Customers
WHERE SUBSTRING(FirstName, 1, 1) = 'A';
```

**Good Practice:**

```sql
SELECT * 
FROM Sales.Customers
WHERE FirstName LIKE 'A%';
```

**Bad Practice:**

```sql
SELECT * 
FROM Sales.Orders 
WHERE YEAR(OrderDate) = 2025;
```

**Good Practice:**

```sql
SELECT * 
FROM Sales.Orders 
WHERE OrderDate BETWEEN '2025-01-01' AND '2025-12-31';
```

---

### Tip 6: Avoid Leading Wildcards as They Prevent Index Usage

Leading wildcards in `LIKE` clauses hinder index utilization.

**Bad Practice:**

```sql
SELECT * 
FROM Sales.Customers 
WHERE LastName LIKE '%Gold%';
```

**Good Practice:**

```sql
SELECT * 
FROM Sales.Customers 
WHERE LastName LIKE 'Gold%';
```

---

### Tip 7: Use IN Instead of Multiple OR

`IN` is more efficient and readable than multiple `OR` conditions.

**Bad Practice:**

```sql
SELECT * 
FROM Sales.Orders
WHERE CustomerID = 1 OR CustomerID = 2 OR CustomerID = 3;
```

**Good Practice:**

```sql
SELECT * 
FROM Sales.Orders
WHERE CustomerID IN (1, 2, 3);
```

---

## Joins

### Tip 8: Understand the Speed of Joins & Use INNER JOIN When Possible

`INNER JOIN` is generally faster than `LEFT` or `RIGHT JOIN`.

**Best Performance:**

```sql
SELECT c.FirstName, o.OrderID 
FROM Sales.Customers c 
INNER JOIN Sales.Orders o 
ON c.CustomerID = o.CustomerID;
```

**Slightly Slower Performance:**

```sql
SELECT c.FirstName, o.OrderID 
FROM Sales.Customers c 
RIGHT JOIN Sales.Orders o 
ON c.CustomerID = o.CustomerID;
```

```sql
SELECT c.FirstName, o.OrderID 
FROM Sales.Customers c 
LEFT JOIN Sales.Orders o 
ON c.CustomerID = o.CustomerID;
```

**Worst Performance:**

```sql
SELECT c.FirstName, o.OrderID 
FROM Sales.Customers c 
LEFT OUTER JOIN Sales.Orders o 
ON c.CustomerID = o.CustomerID;
```

---

### Tip 9: Use Explicit Join (ANSI Join) Instead of Implicit Join (non-ANSI Join)

Explicit joins are clearer and easier to optimize.

**Bad Practice:**

```sql
SELECT o.OrderID, c.FirstName
FROM Sales.Customers c, Sales.Orders o
WHERE c.CustomerID = o.CustomerID;
```

**Good Practice:**

```sql
SELECT o.OrderID, c.FirstName
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID;
```

**Note:** For simple queries, there is no measurable performance difference if both ANSI and non-ANSI queries are correctly written. For complex queries, ANSI joins are usually easier to optimize and debug.

---

### Tip 10: Index Columns Used in the ON Clause

Indexing join columns improves performance.

```sql
SELECT c.FirstName, o.OrderID
FROM Sales.Orders AS o
INNER JOIN Sales.Customers AS c
    ON c.CustomerID = o.CustomerID;
```

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID ON Sales.Orders(CustomerID);
```

---

### Tip 11: Filter Before Joining (Big Tables)

Filtering before joining reduces the dataset size for large tables.

**Best Practice for Small-Medium Tables (Filter After Join):**

```sql
SELECT c.FirstName, o.OrderID
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID
WHERE o.OrderStatus = 'Delivered';
```

**Filter During Join (ON):**

```sql
SELECT c.FirstName, o.OrderID
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID
    AND o.OrderStatus = 'Delivered';
```

**Best Practice for Big Tables (Filter Before Join):**

```sql
SELECT c.FirstName, o.OrderID
FROM Sales.Customers AS c
INNER JOIN (
    SELECT OrderID, CustomerID
    FROM Sales.Orders
    WHERE OrderStatus = 'Delivered'
) AS o
    ON c.CustomerID = o.CustomerID;
```

---

### Tip 12: Aggregate Before Joining (Big Tables)

Pre-aggregating data reduces the number of rows before joining.

**Best Practice for Small-Medium Tables:**

```sql
SELECT c.CustomerID, c.FirstName, COUNT(o.OrderID) AS OrderCount
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID, c.FirstName;
```

**Best Practice for Big Tables:**

```sql
SELECT c.CustomerID, c.FirstName, o.OrderCount
FROM Sales.Customers AS c
INNER JOIN (
    SELECT CustomerID, COUNT(OrderID) AS OrderCount
    FROM Sales.Orders
    GROUP BY CustomerID
) AS o
    ON c.CustomerID = o.CustomerID;
```

**Bad Practice (Correlated Subquery):**

```sql
SELECT 
    c.CustomerID, 
    c.FirstName,
    (SELECT COUNT(o.OrderID)
     FROM Sales.Orders AS o
     WHERE o.CustomerID = c.CustomerID) AS OrderCount
FROM Sales.Customers AS c;
```

---

### Tip 13: Use UNION Instead of OR in Joins

Using `UNION` can be more efficient than `OR` in join conditions.

**Bad Practice:**

```sql
SELECT o.OrderID, c.FirstName
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID
    OR c.CustomerID = o.SalesPersonID;
```

**Best Practice:**

```sql
SELECT o.OrderID, c.FirstName
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID
UNION
SELECT o.OrderID, c.FirstName
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.SalesPersonID;
```

---

### Tip 14: Check for Nested Loops and Use SQL HINTS

Use query hints to optimize join algorithms for large tables.

```sql
SELECT o.OrderID, c.FirstName
FROM Sales.Customers c
INNER JOIN Sales.Orders o 
ON c.CustomerID = o.CustomerID;
```

**Good Practice for Big Table & Small Table:**

```sql
SELECT o.OrderID, c.FirstName
FROM Sales.Customers AS c
INNER JOIN Sales.Orders AS o
    ON c.CustomerID = o.CustomerID
OPTION (HASH JOIN);
```

---

## Union

### Tip 15: Use UNION ALL Instead of UNION When Duplicates Are Acceptable

`UNION ALL` is faster than `UNION` as it does not remove duplicates.

**Bad Practice:**

```sql
SELECT CustomerID FROM Sales.Orders
UNION
SELECT CustomerID FROM Sales.OrdersArchive;
```

**Best Practice:**

```sql
SELECT CustomerID FROM Sales.Orders
UNION ALL
SELECT CustomerID FROM Sales.OrdersArchive;
```

---

### Tip 16: Use UNION ALL + DISTINCT Instead of UNION When Duplicates Are Not Acceptable

Combine `UNION ALL` with `DISTINCT` for better performance when duplicates must be removed.

**Bad Practice:**

```sql
SELECT CustomerID FROM Sales.Orders
UNION
SELECT CustomerID FROM Sales.OrdersArchive;
```

**Best Practice:**

```sql
SELECT DISTINCT CustomerID
FROM (
    SELECT CustomerID FROM Sales.Orders
    UNION ALL
    SELECT CustomerID FROM Sales.OrdersArchive
) AS CombinedData;
```

---

## Aggregations

### Tip 17: Use Columnstore Index for Aggregations on Large Tables

Columnstore indexes optimize aggregation queries on large datasets.

```sql
SELECT CustomerID, COUNT(OrderID) AS OrderCount
FROM Sales.Orders 
GROUP BY CustomerID;
```

```sql
CREATE CLUSTERED COLUMNSTORE INDEX Idx_Orders_Columnstore ON Sales.Orders;
```

---

### Tip 18: Pre-Aggregate Data and Store It in a New Table for Reporting

Pre-aggregating data into a summary table improves reporting performance.

```sql
SELECT MONTH(OrderDate) OrderYear, SUM(Sales) AS TotalSales
INTO Sales.SalesSummary
FROM Sales.Orders
GROUP BY MONTH(OrderDate);
```

```sql
SELECT OrderYear, TotalSales FROM Sales.SalesSummary;
```

---

## Subqueries, CTE

### Tip 19: JOIN vs EXISTS vs IN (Avoid Using IN)

`JOIN` or `EXISTS` are often more efficient than `IN` for subqueries.

**JOIN (Best Practice if Performance Equals EXISTS):**

```sql
SELECT o.OrderID, o.Sales
FROM Sales.Orders AS o
INNER JOIN Sales.Customers AS c
    ON o.CustomerID = c.CustomerID
WHERE c.Country = 'USA';
```

**EXISTS (Best Practice for Large Tables):**

```sql
SELECT o.OrderID, o.Sales
FROM Sales.Orders AS o
WHERE EXISTS (
    SELECT 1
    FROM Sales.Customers AS c
    WHERE c.CustomerID = o.CustomerID
      AND c.Country = 'USA'
);
```

**IN (Bad Practice):**

```sql
SELECT o.OrderID, o.Sales
FROM Sales.Orders AS o
WHERE o.CustomerID IN (
    SELECT CustomerID
    FROM Sales.Customers
    WHERE Country = 'USA'
);
```

---

### Tip 20: Avoid Redundant Logic in Your Query
Use window functions or `CASE` to avoid redundant subqueries for better performance and readability.

**Bad Practice:**
```sql
SELECT EmployeeID, FirstName, 'Above Average' AS Status
FROM Sales.Employees
WHERE Salary > (SELECT AVG(Salary) FROM Sales.Employees)
UNION ALL
SELECT EmployeeID, FirstName, 'Below Average' AS Status
FROM Sales.Employees
WHERE Salary < (SELECT AVG(Salary) FROM Sales.Employees);
```

**Good Practice:**
```sql
SELECT 
    EmployeeID, 
    FirstName, 
    CASE 
        WHEN Salary > AVG(Salary) OVER () THEN 'Above Average'
        WHEN Salary < AVG(Salary) OVER () THEN 'Below Average'
        ELSE 'Average'
    END AS Status
FROM Sales.Employees;
```

---

## DDL

### Tip 21: Avoid VARCHAR Data Type If Possible
Use more specific data types to optimize storage and performance.

### Tip 22: Avoid Using MAX or Overly Large Lengths
Specify appropriate lengths for `VARCHAR` to save space.

### Tip 23: Use NOT NULL If Possible
Enforce `NOT NULL` constraints to improve query optimization.

### Tip 24: Make Sure All Tables Have a CLUSTERED PRIMARY KEY
A clustered primary key improves data retrieval efficiency.

### Tip 25: Create Nonclustered Index on Frequently Used Foreign Keys
Indexes on foreign keys speed up joins and lookups.

**Bad Practice:**
```sql
CREATE TABLE CustomersInfo (
    CustomerID INT,
    FirstName VARCHAR(MAX),
    LastName TEXT,
    Country VARCHAR(255),
    TotalPurchases FLOAT, 
    Score VARCHAR(255),
    BirthDate VARCHAR(255),
    EmployeeID INT,
    CONSTRAINT FK_Bad_Customers_EmployeeID FOREIGN KEY (EmployeeID)
        REFERENCES Sales.Employees(EmployeeID)
);
```

**Good Practice:**
```sql
CREATE TABLE CustomersInfo (
    CustomerID INT PRIMARY KEY CLUSTERED,
    FirstName VARCHAR(50) NOT NULL,
    LastName VARCHAR(50) NOT NULL,
    Country VARCHAR(50) NOT NULL,
    TotalPurchases FLOAT,
    Score INT,
    BirthDate DATE,
    EmployeeID INT,
    CONSTRAINT FK_CustomersInfo_EmployeeID FOREIGN KEY (EmployeeID)
        REFERENCES Sales.Employees(EmployeeID)
);
CREATE NONCLUSTERED INDEX IX_CustomersInfo_EmployeeID
ON CustomersInfo(EmployeeID);
```

---

## Indexing

### Tip 26: Avoid Over Indexing
Too many indexes can slow down `INSERT`, `UPDATE`, and `DELETE` operations.

### Tip 27: Regularly Review and Drop Unused Indexes
Remove unused indexes to save space and improve write performance.

### Tip 28: Update Table Statistics Weekly
Ensure the query optimizer has up-to-date statistics for optimal performance.

### Tip 29: Reorganize and Rebuild Fragmented Indexes Weekly
Maintain query performance by addressing index fragmentation.

### Tip 30: Partition Large Tables and Apply Columnstore Index
For large fact tables, partitioning combined with columnstore indexes optimizes performance.