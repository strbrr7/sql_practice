# 05. Subqueries and CTEs

**When you need a subquery:** when checking a condition requires calculating something over the whole table first (for example, an average) or taking a list from another query. `WHERE` only sees one row at a time, so `WHERE amount > AVG(amount)` does not work.

**A CTE (`WITH … AS`)** is a named subquery: you calculate an intermediate table, give it a name, and then work with it like a regular table. It's easier to read than nested subqueries and lets you build a query step by step.

## Data

A payments table with merchants and acquirers (also used in section 06).

```sql
DROP TABLE IF EXISTS payments;
CREATE TABLE payments (
    payment_id INTEGER PRIMARY KEY,
    merchant   TEXT,
    acquirer   TEXT,
    country    TEXT,
    amount     REAL,
    status     TEXT,
    created_at TEXT
);
INSERT INTO payments VALUES
(1,  'MEGOGO', 'Bank A', 'UA', 199,  'approved', '2026-11-01'),
(2,  'MEGOGO', 'Bank B', 'PL', 199,  'declined', '2026-11-01'),
(3,  'MEGOGO', 'Bank A', 'UA', 399,  'approved', '2026-11-02'),
(4,  'MEGOGO', 'Bank B', 'DE', 199,  'approved', '2026-11-03'),
(5,  'Bolt',   'Bank A', 'PL', 120,  'approved', '2026-11-01'),
(6,  'Bolt',   'Bank A', 'PL', 85,   'declined', '2026-11-02'),
(7,  'Bolt',   'Bank C', 'DE', 300,  'approved', '2026-11-02'),
(8,  'Bolt',   'Bank C', 'UA', 85,   'approved', '2026-11-03'),
(9,  'Ajax',   'Bank B', 'UA', 950,  'approved', '2026-11-01'),
(10, 'Ajax',   'Bank B', 'PL', 1200, 'declined', '2026-11-02'),
(11, 'Ajax',   'Bank C', 'DE', 950,  'approved', '2026-11-03'),
(12, 'Bolt',   'Bank A', 'UA', 50,   'approved', '2026-11-03');
```

## Subqueries

**1. Payments with an amount above the table-wide average.**
```sql
SELECT payment_id, merchant, amount
FROM payments
WHERE amount > (SELECT AVG(amount) FROM payments)
ORDER BY amount DESC;
```
Result: the average amount is 394.67; above it are payments 10 (Ajax 1200), 9 and 11 (Ajax 950), 3 (MEGOGO 399).
> The subquery runs first and returns a single number, which is then used in the condition.

**2. Merchants that had a declined payment above 500.**
```sql
SELECT DISTINCT merchant
FROM payments
WHERE merchant IN (
    SELECT merchant
    FROM payments
    WHERE status = 'declined'
      AND amount > 500
);
```
Result: Ajax.

**3. Customers who made at least one transaction above the average transaction amount** (MySQL, task without a database).
Table `transactions (transaction_id, customer_id, amount, transaction_date)`.
```sql
SELECT DISTINCT customer_id
FROM transactions
WHERE amount > (SELECT AVG(amount) FROM transactions);
```
> `DISTINCT`, because one customer can have several large transactions, and we need a list of customers.

## CTEs

**4. Approval rate per acquirer, from worst to best.**
```sql
WITH stats AS (
    SELECT acquirer,
           COUNT(*) AS total,
           SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) AS approved
    FROM payments
    GROUP BY acquirer
)
SELECT acquirer, total, approved,
       ROUND(100.0 * approved / total, 1) AS approval_rate
FROM stats
ORDER BY approval_rate;
```
| acquirer | total | approved | approval_rate |
|---|---|---|---|
| Bank B | 4 | 2 | 50.0 |
| Bank A | 5 | 4 | 80.0 |
| Bank C | 3 | 3 | 100.0 |

**5. Acquirers with an approval rate below the average across all acquirers** (two chained CTEs + a subquery).
```sql
WITH stats AS (
    SELECT acquirer,
           COUNT(*) AS total,
           SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) AS approved
    FROM payments
    GROUP BY acquirer
),
rated AS (
    SELECT acquirer,
           ROUND(100.0 * approved / total, 1) AS approval_rate
    FROM stats
)
SELECT acquirer, approval_rate
FROM rated
WHERE approval_rate < (SELECT AVG(approval_rate) FROM rated);
```
Result: Bank B (50.0).
> Each CTE takes data from the previous one, and the final query from the last one. `WITH` is written once, with commas between CTEs.

> Analysis note: Bank B has only 4 payments, so before making a decision (for example, rerouting traffic) it's worth checking a longer period to make sure this isn't noise from a small sample.
