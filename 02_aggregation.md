# 02. Aggregation: GROUP BY, HAVING, dates

**The rule behind this whole section:** every column in `SELECT` must either be in `GROUP BY` or inside an aggregate function. A condition on an individual row goes into `WHERE`, a condition on an aggregated result goes into `HAVING`.

## Part 1. Payments

```sql
DROP TABLE IF EXISTS payments;
CREATE TABLE payments (
    id INTEGER PRIMARY KEY,
    merchant TEXT,
    amount INTEGER,
    currency TEXT,
    status TEXT,
    country TEXT,
    date TEXT
);
INSERT INTO payments VALUES
(1, 'Uklon',   200, 'UAH', 'success', 'UA', '2026-11-01'),
(2, 'Megogo',  150, 'UAH', 'failed',  'UA', '2026-11-01'),
(3, 'Uklon',   300, 'PLN', 'success', 'PL', '2026-11-02'),
(4, 'Rozetka', 500, 'UAH', 'success', 'UA', '2026-11-02'),
(5, 'Megogo',  120, 'EUR', 'failed',  'DE', '2026-11-03');
```

**1. Merchant, amount and country of all failed payments.**
```sql
SELECT merchant, amount, country
FROM payments
WHERE status = 'failed';
```
Result: Megogo 150 UA; Megogo 120 DE.

**2. Successful payments from Ukraine or Poland with an amount above 250.**
```sql
SELECT *
FROM payments
WHERE status = 'success'
  AND country IN ('UA', 'PL')
  AND amount > 250;
```
Result: payments 3 (Uklon, 300) and 4 (Rozetka, 500).

**3. Payments from November 1 to November 2 inclusive, from largest to smallest amount.**
```sql
SELECT merchant, amount
FROM payments
WHERE date BETWEEN '2026-11-01' AND '2026-11-02'
ORDER BY amount DESC;
```
Result: Rozetka 500, Uklon 300, Uklon 200, Megogo 150.

**4. Total amount of successful payments per merchant.**
```sql
SELECT merchant, SUM(amount) AS total
FROM payments
WHERE status = 'success'
GROUP BY merchant;
```
Result: Uklon 500, Rozetka 500. Megogo is missing because both of its payments failed.

**5. Number of payments per country.**
```sql
SELECT country, COUNT(*) AS payments_count
FROM payments
GROUP BY country
ORDER BY payments_count DESC;
```
Result: UA 3, PL 1, DE 1.

**6. Merchants with a total payment amount above 300.**
```sql
SELECT merchant, SUM(amount) AS total
FROM payments
GROUP BY merchant
HAVING SUM(amount) > 300;
```
Result: Uklon 500, Rozetka 500.

**7. Countries with more than one successful payment.**
```sql
SELECT country, COUNT(*) AS payments_count
FROM payments
WHERE status = 'success'
GROUP BY country
HAVING COUNT(*) > 1
ORDER BY payments_count DESC;
```
Result: UA 2.

**8. Total number of failed payments.**
```sql
SELECT COUNT(*) AS failed_count
FROM payments
WHERE status = 'failed';
```
Result: 2.

**9. Number and total amount of payments per status.**
```sql
SELECT status, SUM(amount) AS total, COUNT(*) AS cnt
FROM payments
GROUP BY status
ORDER BY total DESC;
```
Result: success 1000, 3; failed 270, 2.

**10. Merchants with an average payment amount below 200.**
```sql
SELECT merchant, AVG(amount) AS avg_amount
FROM payments
GROUP BY merchant
HAVING AVG(amount) < 200;
```
Result: Megogo 135.

**11. Number of successful payments per day.**
```sql
SELECT date, COUNT(*) AS cnt
FROM payments
WHERE status = 'success'
GROUP BY date
ORDER BY date;
```
Result: 2026-11-01 — 1; 2026-11-02 — 2.

**12. Merchants with more than one payment in UAH.**
```sql
SELECT merchant, COUNT(*) AS cnt
FROM payments
WHERE currency = 'UAH'
GROUP BY merchant
HAVING COUNT(*) > 1;
```
Result: empty — each merchant has only one payment in UAH.

**13. Payment count and maximum amount for merchants whose maximum exceeds 250.**
```sql
SELECT merchant, COUNT(*) AS cnt, MAX(amount) AS max_amount
FROM payments
GROUP BY merchant
HAVING MAX(amount) > 250
ORDER BY max_amount DESC;
```
Result: Rozetka 1, 500; Uklon 2, 300.

## Part 2. Transactions: practice

```sql
DROP TABLE IF EXISTS transactions;
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY,
    client TEXT,
    amount INTEGER,
    method TEXT,
    status TEXT,
    country TEXT,
    day TEXT
);
INSERT INTO transactions VALUES
(1, 'Nova Post', 100, 'card',       'success',  'UA', '2026-11-05'),
(2, 'Nova Post', 250, 'apple_pay',  'success',  'UA', '2026-11-05'),
(3, 'Uklon',      80, 'card',       'declined', 'UA', '2026-11-05'),
(4, 'Uklon',     300, 'card',       'success',  'PL', '2026-11-06'),
(5, 'Megogo',     50, 'google_pay', 'success',  'UA', '2026-11-06'),
(6, 'Megogo',    400, 'card',       'declined', 'DE', '2026-11-06'),
(7, 'Nova Post', 150, 'card',       'declined', 'PL', '2026-11-07'),
(8, 'Uklon',     200, 'apple_pay',  'success',  'UA', '2026-11-07');
```

**14. Number of transactions per payment method.**
```sql
SELECT method, COUNT(*) AS cnt
FROM transactions
GROUP BY method
ORDER BY cnt DESC;
```
Result: card 5, apple_pay 2, google_pay 1.

**15. Total amount of successful transactions per client.**
```sql
SELECT client, SUM(amount) AS total
FROM transactions
WHERE status = 'success'
GROUP BY client
ORDER BY total DESC;
```
Result: Uklon 500, Nova Post 350, Megogo 50.

**16. Countries with more than two transactions.**
```sql
SELECT country, COUNT(*) AS total_transactions
FROM transactions
GROUP BY country
HAVING COUNT(*) > 2;
```
Result: UA 5.

**17. Number of declined transactions per day.**
```sql
SELECT day, COUNT(*) AS declined_count
FROM transactions
WHERE status = 'declined'
GROUP BY day;
```
Result: one declined transaction on each of November 5, 6 and 7.

**18. Clients with an average transaction amount above 170.**
```sql
SELECT client, AVG(amount) AS avg_amount
FROM transactions
GROUP BY client
HAVING AVG(amount) > 170
ORDER BY avg_amount;
```
Result: Uklon 193.33, Megogo 225. Nova Post (166.67) does not pass the filter.

**19. Number of transactions per client–status pair.**
```sql
SELECT client, status, COUNT(*) AS cnt
FROM transactions
GROUP BY client, status
ORDER BY client;
```
Result: Megogo — declined 1, success 1; Nova Post — declined 1, success 2; Uklon — declined 1, success 2.

**20. Number of declines per payment method.**
```sql
SELECT method, COUNT(*) AS declined_count
FROM transactions
WHERE status = 'declined'
GROUP BY method
ORDER BY declined_count DESC;
```
Result: card 3. Conclusion: all declines were card payments.

**21. Daily turnover (sum of successful payments) in Ukraine, only days with turnover above 200.**
```sql
SELECT day, SUM(amount) AS turnover
FROM transactions
WHERE status = 'success'
  AND country = 'UA'
GROUP BY day
HAVING SUM(amount) > 200;
```
Result: 2026-11-05 — 350.

**22. Number of transactions per client in each country, most active first.**
```sql
SELECT client, country, COUNT(*) AS cnt
FROM transactions
GROUP BY client, country
ORDER BY cnt DESC, client;
```
Result: Nova Post UA 2, Uklon UA 2, all other combinations 1.

**23. How many different clients paid by card.**
```sql
SELECT COUNT(DISTINCT client) AS clients_count
FROM transactions
WHERE method = 'card';
```
Result: 3.

**24. Clients whose declined transactions total more than 100.**
```sql
SELECT client, COUNT(*) AS declined_count, SUM(amount) AS declined_amount
FROM transactions
WHERE status = 'declined'
GROUP BY client
HAVING SUM(amount) > 100
ORDER BY declined_amount DESC;
```
Result: Megogo 1, 400; Nova Post 1, 150.

## Part 3. Tasks without a database (MySQL)

These tasks were solved without access to a database, as in an interview, based only on the table descriptions.

**25. Total sales amount per product.**
Table `sales (sale_id, product, amount)`.
```sql
SELECT product, SUM(amount) AS total_amount
FROM sales
GROUP BY product;
```
> I don't add `amount` on its own to SELECT: a product has many sales, and without an aggregate function it's unclear which amount to show.

**26. Customers whose orders total more than 1000.**
Table `orders (order_id, customer_id, amount)`.
```sql
SELECT customer_id, SUM(amount) AS total_amount
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

**27. Total completed transactions per customer, only customers with a total above 500.**
Table `transactions (transaction_id, customer_id, amount, status)`.
```sql
SELECT customer_id, SUM(amount) AS total_completed
FROM transactions
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 500;
```
> The status is filtered in WHERE, before grouping. Otherwise failed transactions would be included in the sum.

**28. Total transaction amount per month, only months with a total above 1000.**
Table `transactions (transaction_id, transaction_date, amount)`.
```sql
SELECT DATE_FORMAT(transaction_date, '%Y-%m') AS month,
       SUM(amount) AS total_amount
FROM transactions
GROUP BY DATE_FORMAT(transaction_date, '%Y-%m')
HAVING SUM(amount) > 1000
ORDER BY month;
```
> I group by year and month (`%Y-%m`), not just the month: otherwise November of different years would merge into one group.

**29. Total completed orders per region–customer pair, only pairs with a total above 2000.**
Table `orders (order_id, customer_id, region, amount, status)`.
```sql
SELECT region, customer_id, SUM(amount) AS total_completed
FROM orders
WHERE status = 'completed'
GROUP BY region, customer_id
HAVING SUM(amount) > 2000;
```

**30. Customers with completed transactions totalling more than 1000 in the current month.**
Table `transactions (transaction_id, customer_id, amount, transaction_date, status)`.
```sql
SELECT customer_id
FROM transactions
WHERE DATE_FORMAT(transaction_date, '%Y-%m') = DATE_FORMAT(CURDATE(), '%Y-%m')
  AND status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```
> Both sides of the comparison use the same `year-month` format. For large tables, a range filter is faster: `transaction_date >= DATE_FORMAT(CURDATE(), '%Y-%m-01') AND transaction_date < DATE_FORMAT(CURDATE(), '%Y-%m-01') + INTERVAL 1 MONTH`.
