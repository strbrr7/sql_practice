# 06. Window functions

```
function() OVER (PARTITION BY [group] ORDER BY [order])
```

- **`PARTITION BY`** — what to calculate for (like GROUP BY, but rows are not collapsed).
- **`ORDER BY`** inside `OVER` — who comes first and who is the "neighbor". Required for rankings, running totals, LAG / LEAD.
- Window functions are calculated at the `SELECT` stage, i.e. after `WHERE`. So their result can only be filtered through a CTE or a subquery.

| Task | Function | Parentheses |
|---|---|---|
| group value next to each row | `AVG / SUM / COUNT (column) OVER (PARTITION BY …)` | column |
| numbering, top-N | `ROW_NUMBER() / RANK() / DENSE_RANK()` | empty |
| running total | `SUM(column) OVER (… ORDER BY …)` | column |
| previous / next row | `LAG(column) / LEAD(column)` | column |
| first value in a group | `FIRST_VALUE(column)` | column |

## Part 1. Payments

Data: the `payments` table from section [05](05_subqueries_cte.md#data).

**1. Each payment next to the merchant's average amount, payment count and deviation from the average.**
```sql
SELECT payment_id, merchant, amount,
       ROUND(AVG(amount) OVER (PARTITION BY merchant), 2)          AS avg_merchant,
       COUNT(*) OVER (PARTITION BY merchant)                       AS n_payments,
       ROUND(amount - AVG(amount) OVER (PARTITION BY merchant), 2) AS diff
FROM payments;
```
| payment_id | merchant | amount | avg_merchant | n_payments | diff |
|---|---|---|---|---|---|
| 9 | Ajax | 950 | 1033.33 | 3 | -83.33 |
| 10 | Ajax | 1200 | 1033.33 | 3 | 166.67 |
| 11 | Ajax | 950 | 1033.33 | 3 | -83.33 |
| 5 | Bolt | 120 | 128 | 5 | -8 |
| 6 | Bolt | 85 | 128 | 5 | -43 |
| 7 | Bolt | 300 | 128 | 5 | 172 |
| 8 | Bolt | 85 | 128 | 5 | -43 |
| 12 | Bolt | 50 | 128 | 5 | -78 |
| 1 | MEGOGO | 199 | 249 | 4 | -50 |
| 2 | MEGOGO | 199 | 249 | 4 | -50 |
| 3 | MEGOGO | 399 | 249 | 4 | 150 |
| 4 | MEGOGO | 199 | 249 | 4 | -50 |

> Unlike GROUP BY, all 12 rows are kept. The `diff` column immediately shows payments that differ a lot from what is typical for the merchant.

**2. ROW_NUMBER, RANK and DENSE_RANK: the difference on equal amounts.**
```sql
SELECT merchant, amount,
       ROW_NUMBER() OVER (PARTITION BY merchant ORDER BY amount DESC) AS rn,
       RANK()       OVER (PARTITION BY merchant ORDER BY amount DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY merchant ORDER BY amount DESC) AS drnk
FROM payments
WHERE merchant = 'Bolt';
```
| merchant | amount | rn | rnk | drnk |
|---|---|---|---|---|
| Bolt | 300 | 1 | 1 | 1 |
| Bolt | 120 | 2 | 2 | 2 |
| Bolt | 85 | 3 | 3 | 3 |
| Bolt | 85 | 4 | 3 | 3 |
| Bolt | 50 | 5 | 5 | 4 |

> RANK skips a number after a tie (it counts how many rows are above), DENSE_RANK does not (it counts how many distinct values are above). ROW_NUMBER always gives unique numbers.

**3. The two largest payments of each merchant.**
```sql
WITH ranked AS (
    SELECT merchant, amount,
           ROW_NUMBER() OVER (PARTITION BY merchant ORDER BY amount DESC) AS rn
    FROM payments
)
SELECT merchant, amount, rn
FROM ranked
WHERE rn <= 2;
```
Result: Ajax 1200, 950; Bolt 300, 120; MEGOGO 399, 199.
> With `RANK`, Ajax would get three rows: both 950 payments share second place.

**4. Running total of each merchant's payments.**
```sql
SELECT merchant, created_at, amount,
       SUM(amount) OVER (PARTITION BY merchant ORDER BY created_at, payment_id) AS running_total
FROM payments;
```
Result for MEGOGO: 199 → 398 → 797 → 996. The total starts over for each merchant.
> `payment_id` in the ordering makes same-day payments add up one by one rather than as a single block.

**5. Change in amount compared to the merchant's previous payment.**
```sql
SELECT merchant, created_at, amount,
       LAG(amount) OVER (PARTITION BY merchant ORDER BY created_at, payment_id)          AS prev_amount,
       amount - LAG(amount) OVER (PARTITION BY merchant ORDER BY created_at, payment_id) AS change
FROM payments;
```
The first payment of each merchant has NULL: there is no previous payment.

**6. Daily approval rate and its change compared to the previous day.**
```sql
WITH daily AS (
    SELECT created_at,
           ROUND(100.0 * SUM(CASE WHEN status = 'approved' THEN 1 ELSE 0 END) / COUNT(*), 1) AS approval_rate
    FROM payments
    GROUP BY created_at
)
SELECT created_at, approval_rate,
       LAG(approval_rate) OVER (ORDER BY created_at) AS prev_rate,
       ROUND(approval_rate - LAG(approval_rate) OVER (ORDER BY created_at), 1) AS change
FROM daily;
```
| created_at | approval_rate | prev_rate | change |
|---|---|---|---|
| 2026-11-01 | 75.0 | | |
| 2026-11-02 | 50.0 | 75.0 | -25.0 |
| 2026-11-03 | 100.0 | 50.0 | 50.0 |

> A sharp drop in approval rate (−25 pp on November 2) is a signal to check which acquirers and decline codes it's related to.

## Part 2. Northwind

Database: [sql-practice.com](https://www.sql-practice.com/), `northwind.db` (SQLite).

**7. Each customer's previous order.**
```sql
SELECT c.customer_id, o.order_id, o.order_date,
       LAG(o.order_id) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS previous_order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
```

**8. Each customer's first order.**
```sql
SELECT c.customer_id, o.order_id, o.order_date,
       FIRST_VALUE(o.order_id) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS first_order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
```
> `LAST_VALUE` with the same `ORDER BY` would return the current row, because by default the window ends at the current row. For the last value it's simpler to use `FIRST_VALUE` with `ORDER BY … DESC`.

**9. The three most expensive products in each category.**
```sql
WITH ranked AS (
    SELECT c.category_id, c.category_name, p.product_id, p.product_name, p.unit_price,
           RANK() OVER (PARTITION BY c.category_id ORDER BY p.unit_price DESC) AS product_rank
    FROM products p
    LEFT JOIN categories c ON c.category_id = p.category_id
)
SELECT category_id, category_name, product_id, product_name, unit_price, product_rank
FROM ranked
WHERE product_rank <= 3
ORDER BY category_id, product_rank;
```
Result (Beverages): Côte de Blaye 263.5, Ipoh Coffee 46, Chang 19.

**10. Days between a customer's current and previous order.**
```sql
WITH prev_d AS (
    SELECT c.customer_id, o.order_id, o.order_date,
           LAG(o.order_date) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS previous_order_date
    FROM customers c
    JOIN orders o ON o.customer_id = c.customer_id
)
SELECT customer_id, order_id, order_date, previous_order_date,
       JULIANDAY(order_date) - JULIANDAY(previous_order_date) AS diff_days
FROM prev_d;
```
Result for ALFKI: 39, 10, 94, 60 days between orders.

**11. Average interval between customers' orders by region.**
```sql
WITH prev_od AS (
    SELECT c.region, o.order_date,
           LAG(o.order_date) OVER (PARTITION BY c.customer_id ORDER BY o.order_date, o.order_id) AS previous_order_date
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
),
diff_days AS (
    SELECT region,
           JULIANDAY(order_date) - JULIANDAY(previous_order_date) AS diff_days
    FROM prev_od
)
SELECT region,
       ROUND(AVG(diff_days), 2) AS avg_diff_days
FROM diff_days
GROUP BY region
ORDER BY avg_diff_days;
```
| region | avg_diff_days |
|---|---|
| Northern Europe | 44.22 |
| North America | 49.21 |
| Western Europe | 53.96 |
| Scandinavia | 54.28 |
| British Isles | 58.45 |
| South America | 65.28 |
| Eastern Europe | 84.00 |
| Southern Europe | 89.07 |
| Central America | 94.39 |

**Conclusions.** Customers place repeat orders most often in Northern Europe and least often in Central America. But to compare regions correctly, we need to know the number of customers in each: if Central America has only a few customers, the average depends on the behavior of one or two people, and a single long gap between orders noticeably raises it. So I would add the number of customers (`COUNT(DISTINCT customer_id)`) and look at the median, which is more robust to outliers. Also, customers with only one order are not included in the calculation: they have no interval.

**12. The three employees with the most processed orders.**
```sql
WITH order_count_table AS (
    SELECT e.employee_id, e.first_name, e.last_name,
           COUNT(o.order_id) AS order_count
    FROM employees e
    LEFT JOIN orders o ON o.employee_id = e.employee_id
    GROUP BY e.employee_id, e.first_name, e.last_name
),
ranked AS (
    SELECT employee_id, first_name, last_name, order_count,
           RANK() OVER (ORDER BY order_count DESC) AS employee_rank
    FROM order_count_table
)
SELECT employee_id, first_name, last_name, order_count, employee_rank
FROM ranked
WHERE employee_rank <= 3
ORDER BY employee_rank;
```
> No `PARTITION BY` here: the ranking is overall, across all employees.

**13. The top three employees in each region.**
```sql
WITH order_count_table AS (
    SELECT e.employee_id, e.first_name, e.last_name, e.region,
           COUNT(o.order_id) AS order_count
    FROM employees e
    LEFT JOIN orders o ON o.employee_id = e.employee_id
    GROUP BY e.employee_id, e.first_name, e.last_name, e.region
),
ranked AS (
    SELECT employee_id, region, first_name, last_name, order_count,
           RANK() OVER (PARTITION BY region ORDER BY order_count DESC) AS employee_rank
    FROM order_count_table
)
SELECT region, employee_id, first_name, last_name, order_count, employee_rank
FROM ranked
WHERE employee_rank <= 3
ORDER BY region, employee_rank;
```
| region | employee_id | first_name | last_name | order_count | employee_rank |
|---|---|---|---|---|---|
| British Isles | 7 | Robert | King | 72 | 1 |
| British Isles | 6 | Michael | Suyama | 67 | 2 |
| British Isles | 9 | Anne | Dodsworth | 43 | 3 |
| North America | 4 | Margaret | Peacock | 156 | 1 |
| North America | 3 | Janet | Leverling | 127 | 2 |
| North America | 1 | Nancy | Davolio | 123 | 3 |
