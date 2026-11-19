# 03. Joins

- **INNER JOIN** (or just `JOIN`) keeps only rows that have a match in both tables.
- **LEFT JOIN** keeps all rows from the left table; where there is no match, NULL is filled in.
- If, after a LEFT JOIN, `WHERE` has a condition on a column from the **right** table, the LEFT JOIN effectively becomes an INNER JOIN: NULL does not pass the condition.

## Part 1. Payments and clients

### INNER JOIN

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (id INTEGER PRIMARY KEY, client_id INTEGER, amount INTEGER);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT);
INSERT INTO payments VALUES (1, 1, 200), (2, 2, 150), (3, 1, 300);
INSERT INTO clients VALUES (1, 'Uklon'), (2, 'Megogo');
```

**1. Client name and amount for each payment.**
```sql
SELECT c.name, p.amount
FROM clients c
JOIN payments p ON c.id = p.client_id;
```
Result: Uklon 200, Megogo 150, Uklon 300.

### JOIN + WHERE + GROUP BY

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (id INTEGER PRIMARY KEY, client_id INTEGER, amount INTEGER, status TEXT);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT, industry TEXT);
INSERT INTO payments VALUES
(1, 1, 200, 'success'),
(2, 2, 150, 'declined'),
(3, 1, 300, 'success'),
(4, 3, 500, 'success'),
(5, 2, 100, 'success');
INSERT INTO clients VALUES
(1, 'Uklon', 'transport'),
(2, 'Megogo', 'media'),
(3, 'Rozetka', 'ecommerce');
```

**2. Total amount of successful payments per client.**
```sql
SELECT c.name, SUM(p.amount) AS total
FROM payments p
JOIN clients c ON c.id = p.client_id
WHERE p.status = 'success'
GROUP BY c.id, c.name
ORDER BY total DESC;
```
Result: Uklon 500, Rozetka 500, Megogo 100.

### LEFT JOIN

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (id INTEGER PRIMARY KEY, client_id INTEGER, amount INTEGER);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT);
INSERT INTO clients VALUES (1, 'Uklon'), (2, 'Megogo'), (4, 'Nova Post');
INSERT INTO payments VALUES (1, 1, 200), (2, 2, 150), (3, 1, 300);
```

**3. All clients and their number of payments, even if they have none.**
```sql
SELECT c.name, COUNT(p.id) AS payments_count
FROM clients c
LEFT JOIN payments p ON c.id = p.client_id
GROUP BY c.id, c.name;
```
Result: Uklon 2, Megogo 1, Nova Post 0.
> After a LEFT JOIN I count `COUNT(p.id)`, not `COUNT(*)`: Nova Post has a row with NULL, and `COUNT(*)` would count it as 1.

**4. Clients with no payments.**
```sql
SELECT c.name
FROM clients c
LEFT JOIN payments p ON c.id = p.client_id
WHERE p.id IS NULL;
```
Result: Nova Post.

## Part 2. Mixed round

```sql
DROP TABLE IF EXISTS payments;
DROP TABLE IF EXISTS clients;
CREATE TABLE payments (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    amount INTEGER,
    method TEXT,
    status TEXT,
    day TEXT
);
CREATE TABLE clients (id INTEGER PRIMARY KEY, name TEXT, industry TEXT);
INSERT INTO payments VALUES
(1, 1, 200, 'card',       'success',  '2026-11-10'),
(2, 1, 100, 'apple_pay',  'declined', '2026-11-10'),
(3, 2, 300, 'card',       'success',  '2026-11-11'),
(4, 3,  50, 'card',       'declined', '2026-11-11'),
(5, 3, 400, 'google_pay', 'success',  '2026-11-12'),
(6, 1, 250, 'card',       'success',  '2026-11-12'),
(7, 2, 150, 'apple_pay',  'success',  '2026-11-12');
INSERT INTO clients VALUES
(1, 'Uklon',     'transport'),
(2, 'Megogo',    'media'),
(3, 'Rozetka',   'ecommerce'),
(4, 'Nova Post', 'logistics'),
(5, 'Bolt',      'transport');
```

**5. All successful card payments, from largest to smallest amount.**
```sql
SELECT *
FROM payments
WHERE status = 'success'
  AND method = 'card'
ORDER BY amount DESC;
```
Result: payments 3 (300), 6 (250), 1 (200).
> No GROUP BY or SUM here: the task asks to show rows, not to calculate.

**6. How many different clients made at least one payment.**
```sql
SELECT COUNT(DISTINCT client_id) AS clients_count
FROM payments;
```
Result: 3.

**7. Clients with a total of successful payments above 300.**
```sql
SELECT c.name, SUM(p.amount) AS total
FROM payments p
JOIN clients c ON c.id = p.client_id
WHERE p.status = 'success'
GROUP BY c.id, c.name
HAVING SUM(p.amount) > 300;
```
Result: Uklon 450, Megogo 450, Rozetka 400.

**8. Clients who have never paid.**
```sql
SELECT c.name
FROM clients c
LEFT JOIN payments p ON c.id = p.client_id
WHERE p.id IS NULL;
```
Result: Nova Post, Bolt.

**9. Number of payments per industry.**
```sql
SELECT c.industry, COUNT(p.id) AS payments_count
FROM payments p
JOIN clients c ON c.id = p.client_id
GROUP BY c.industry
ORDER BY payments_count DESC;
```
Result: transport 3, media 2, ecommerce 2.

**10. Days with more than one payment: count and average amount.**
```sql
SELECT day, COUNT(*) AS payments_count, AVG(amount) AS avg_amount
FROM payments
GROUP BY day
HAVING COUNT(*) > 1;
```
Result: 2026-11-10 — 2, 150; 2026-11-11 — 2, 175; 2026-11-12 — 3, 266.67.

## Part 3. Northwind

Database: [sql-practice.com](https://www.sql-practice.com/), `northwind.db`.

**11. All orders with the name of the employee who processed them.**
```sql
SELECT o.order_id, e.first_name, e.last_name
FROM orders o
LEFT JOIN employees e ON e.employee_id = o.employee_id
ORDER BY o.order_id;
```
> LEFT JOIN because we need **all** orders, even if one has no employee assigned.

**12. Orders from October 2016 with the customer's company name, sorted by date.**
```sql
SELECT o.order_id, o.order_date, c.company_name
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date >= '2016-10-01'
  AND o.order_date <  '2016-11-01'
ORDER BY o.order_date;
```
> `BETWEEN '2016-10-01' AND '2016-11-01'` would also include November 1, so I set the boundaries with `>=` and `<`.

**13. Products from suppliers in North America and Western Europe, sorted by supplier name.**
```sql
SELECT p.product_name, s.company_name, s.region
FROM products p
JOIN suppliers s ON s.supplier_id = p.supplier_id
WHERE s.region IN ('North America', 'Western Europe')
ORDER BY s.company_name;
```

**14. For each ordered product: how many orders it appeared in and how many units were sold.**
```sql
SELECT p.product_name,
       COUNT(DISTINCT od.order_id) AS n_orders,
       SUM(od.quantity)            AS sum_quantity
FROM order_details od
JOIN products p ON p.product_id = od.product_id
GROUP BY p.product_id, p.product_name
ORDER BY n_orders, p.product_name;
```
> `COUNT` counts how many times something occurs, `SUM` adds up numbers. This task needs both.

**15. Customers with more than 7 orders.**
```sql
SELECT c.company_name, COUNT(o.order_id) AS n_orders
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.company_name
HAVING COUNT(o.order_id) > 7
ORDER BY n_orders DESC;
```

**16. Employees working in territories whose name starts with "Santa".**
```sql
SELECT DISTINCT e.first_name, e.last_name
FROM employees e
JOIN employee_territories et ON et.employee_id = e.employee_id
JOIN territories t ON t.territory_id = et.territory_id
WHERE t.territory_description LIKE 'Santa%';
```
Result: Robert King.
> Without `DISTINCT`, Robert King appears three times because he works in three such territories.

**17. Products priced above 20 with the supplier name.**
```sql
SELECT p.product_name, p.unit_price, s.company_name
FROM products p
LEFT JOIN suppliers s ON s.supplier_id = p.supplier_id
WHERE p.unit_price > 20
ORDER BY p.unit_price DESC;
```
> Here the WHERE condition is on the **left** table, so LEFT JOIN really keeps products without a supplier.

**18. Categories with more than 5 products.**
```sql
SELECT c.category_name, COUNT(p.product_id) AS n_products
FROM categories c
JOIN products p ON p.category_id = c.category_id
GROUP BY c.category_id, c.category_name
HAVING COUNT(p.product_id) > 5
ORDER BY n_products DESC;
```

## Part 4. Task without a database

**19. Names of customers from the Kyiv region and the total balance of their accounts.**
Tables `customers (customer_id, customer_name, region)` and `accounts (account_id, customer_id, balance)`.
```sql
SELECT c.customer_name,
       COALESCE(SUM(a.balance), 0) AS total_balance
FROM customers c
LEFT JOIN accounts a ON a.customer_id = c.customer_id
WHERE c.region = 'Kyiv'
GROUP BY c.customer_id, c.customer_name;
```
> I group by `customer_id`, not just the name: two different customers can have the same name. `COALESCE` shows 0 instead of NULL for customers without accounts.
