# 07. Self join

A **self join** joins a table to itself. There is only one table, but it appears twice in the query with different aliases, and I treat it as two separate tables.

Typical tasks:
- **hierarchy** (employee → manager), when the relationship is stored in one table;
- **pairs of records** that share something (same date, same salary, same city).

## Hierarchy: employee and manager

**1. Employees hired in 2013 and their managers** (Northwind, SQLite).
```sql
SELECT e.first_name, e.last_name, e.hire_date,
       m.first_name || ' ' || m.last_name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.reports_to = m.employee_id
WHERE e.hire_date >= '2013-01-01'
  AND e.hire_date <  '2014-01-01'
ORDER BY e.hire_date;
```
| first_name | last_name | hire_date | manager_name |
|---|---|---|---|
| Margaret | Peacock | 2013-05-03 | Andrew Fuller |
| Steven | Buchanan | 2013-10-17 | Andrew Fuller |
| Michael | Suyama | 2013-10-17 | Steven Buchanan |

> **LEFT JOIN**, because the top manager has `reports_to = NULL`, and an INNER JOIN would drop them from the result.
> The year is filtered with a range rather than `LIKE '2013%'`: this way the query also works where the date is stored as a `DATE` type (for example, in PostgreSQL).

## Pairs of records without duplicates

When a table is joined to itself on a shared attribute, each record matches **itself**, and each pair appears **twice** (A–B and B–A). A single condition removes both problems: `t1.id < t2.id`.

| t1 | t2 | `t1.id < t2.id` | |
|---|---|---|---|
| A (1) | A (1) | 1 < 1 — no | ❌ matched with itself |
| A (1) | B (2) | 1 < 2 — yes | ✅ |
| B (2) | A (1) | 2 < 1 — no | ❌ mirrored pair |

**2. Pairs of customers who made transactions on the same day** (without a database, MySQL).
Table `transactions (transaction_id, customer_id, transaction_date)`.
```sql
SELECT DISTINCT t1.customer_id AS customer_1,
                t2.customer_id AS customer_2
FROM transactions t1
JOIN transactions t2 ON t1.transaction_date = t2.transaction_date
WHERE t1.customer_id < t2.customer_id;
```
> `DISTINCT` is needed here: the same pair of customers may coincide on several different days.

**3. Pairs of employees with the same salary** (without a database).
Table `employees (employee_id, name, salary)`.
```sql
SELECT e1.employee_id, e1.name,
       e2.employee_id, e2.name,
       e1.salary
FROM employees e1
JOIN employees e2 ON e1.salary = e2.salary
WHERE e1.employee_id < e2.employee_id;
```
> No `DISTINCT` needed: each employee has one row, so each pair appears exactly once.

**Formula:** self join + `=` on the shared attribute + `<` on the ID (+ `DISTINCT` if one object can have several rows).
