# SQL Payments Analytics

🇬🇧 English | 🇺🇦 [Українська](uk/README.md)

My SQL solutions grouped by topic, from basic queries to JOINs, CTEs, window functions and self joins. Most tasks are built on **payment transaction data** (merchants, acquirers, statuses, approval rate); some use the **Northwind** sample database.

## Contents

| # | Topic | What's inside |
|---|---|---|
| 01 | [Basics](01_basics.md) | SELECT, WHERE, IN, BETWEEN, LIKE, IS NULL, ORDER BY, LIMIT, INSERT / UPDATE / DELETE |
| 02 | [Aggregation](02_aggregation.md) | COUNT, SUM, AVG, MAX, GROUP BY, HAVING, WHERE vs HAVING, working with dates |
| 03 | [Joins](03_joins.md) | INNER JOIN, LEFT JOIN, multi-table joins, finding records without a match |
| 04 | [CASE WHEN](04_case_when.md) | conditional counts, approval rate, categorization |
| 05 | [Subqueries and CTEs](05_subqueries_cte.md) | comparing with a table-wide average, `WITH`, chained CTEs |
| 06 | [Window functions](06_window_functions.md) | `OVER`, `PARTITION BY`, ROW_NUMBER / RANK / DENSE_RANK, LAG, FIRST_VALUE, running totals, top-N per group |
| 07 | [Self join](07_self_join.md) | employee–manager hierarchy, unique pairs without duplicates |
| 08 | [Descriptive statistics](08_statistics.md) | mean vs median, outliers, variance, percentiles |

## Data

- **Payment tables** (`payments`, `transactions`, `clients`). Each section starts with SQL that creates the table and inserts the data, so the tasks can be run on [sqliteonline.com](https://sqliteonline.com) or any SQLite environment.
- **Northwind**, a classic sample database (orders, products, customers, employees). Queries were run on [sql-practice.com](https://www.sql-practice.com/) with `northwind.db`.
- Some tasks in sections 02, 05 and 07 are written for **MySQL** (`DATE_FORMAT`, `CURDATE`); this is noted in the task description.

