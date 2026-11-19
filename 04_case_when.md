# 04. CASE WHEN and approval rate

`CASE WHEN` lets you count not all rows, but only those that meet a condition:

| What to calculate | How |
|---|---|
| number of rows meeting a condition | `SUM(CASE WHEN … THEN 1 ELSE 0 END)` |
| amount for rows meeting a condition | `SUM(CASE WHEN … THEN amount ELSE 0 END)` |
| share (approval rate) | `AVG(CASE WHEN … THEN 1.0 ELSE 0 END)` |

`1.0` rather than `1`: otherwise some databases perform integer division.

## Approval rate

```sql
DROP TABLE IF EXISTS payments;
CREATE TABLE payments (
    id INTEGER PRIMARY KEY,
    client_id INTEGER,
    amount INTEGER,
    method TEXT,
    status TEXT,
    day TEXT
);
INSERT INTO payments VALUES
(1, 1, 200, 'card',       'success',  '2026-11-10'),
(2, 1, 100, 'apple_pay',  'declined', '2026-11-10'),
(3, 2, 300, 'card',       'success',  '2026-11-11'),
(4, 3,  50, 'card',       'declined', '2026-11-11'),
(5, 3, 400, 'google_pay', 'success',  '2026-11-12'),
(6, 1, 250, 'card',       'success',  '2026-11-12'),
(7, 2, 150, 'apple_pay',  'success',  '2026-11-12');
```

**1. For each payment method: number of payments, number of declines and approval rate.**
```sql
SELECT method,
       COUNT(*) AS total,
       SUM(CASE WHEN status = 'declined' THEN 1 ELSE 0 END) AS declined_count,
       AVG(CASE WHEN status = 'success' THEN 1.0 ELSE 0 END) AS approval_rate
FROM payments
GROUP BY method;
```
Result: card 4, 1, 0.75; apple_pay 2, 1, 0.50; google_pay 1, 0, 1.00.

**2. For each day: number of payments, amount of successful payments and approval rate.**
```sql
SELECT day,
       COUNT(*) AS total,
       SUM(CASE WHEN status = 'success' THEN amount ELSE 0 END) AS success_amount,
       AVG(CASE WHEN status = 'success' THEN 1.0 ELSE 0 END) AS approval_rate
FROM payments
GROUP BY day
ORDER BY day;
```
Result: 2026-11-10 — 2, 200, 0.50; 2026-11-11 — 2, 300, 0.50; 2026-11-12 — 3, 800, 1.00.

**3. Clients with an approval rate below 1: number of successful and declined payments.**
```sql
SELECT client_id,
       SUM(CASE WHEN status = 'success'  THEN 1 ELSE 0 END) AS success_count,
       SUM(CASE WHEN status = 'declined' THEN 1 ELSE 0 END) AS declined_count,
       AVG(CASE WHEN status = 'success'  THEN 1.0 ELSE 0 END) AS approval_rate
FROM payments
GROUP BY client_id
HAVING AVG(CASE WHEN status = 'success' THEN 1.0 ELSE 0 END) < 1;
```
Result: client 1 — 2, 1, 0.67; client 3 — 1, 1, 0.50.

## Categorization

```sql
DROP TABLE IF EXISTS transactions;
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY,
    merchant TEXT,
    amount INTEGER,
    currency TEXT,
    status TEXT,
    country TEXT,
    method TEXT
);
INSERT INTO transactions VALUES
(1, 'Uklon',    120, 'UAH', 'success',  'UA', 'card'),
(2, 'Uklon',     80, 'UAH', 'declined', 'UA', 'card'),
(3, 'Bolt',     300, 'EUR', 'success',  'PL', 'apple_pay'),
(4, 'Bolt',      50, 'EUR', 'declined', 'PL', 'card'),
(5, 'Megogo',   500, 'UAH', 'success',  'UA', 'google_pay'),
(6, 'Megogo',    40, 'UAH', 'declined', 'UA', 'card'),
(7, 'Megogo',   200, 'UAH', 'success',  'UA', 'card'),
(8, 'Rozetka', 1000, 'UAH', 'success',  'UA', 'card'),
(9, 'Bolt',     150, 'EUR', 'success',  'DE', 'card');
```

**4. Amount category: small (under 100), medium (100–499), large (500 and above).**
```sql
SELECT id, amount,
       CASE
           WHEN amount < 100 THEN 'small'
           WHEN amount < 500 THEN 'medium'
           ELSE 'large'
       END AS category
FROM transactions;
```
Result: small — 2, 4, 6; medium — 1, 3, 7, 9; large — 5, 8.
> CASE conditions are checked in order, so the second one only needs `< 500`: if we reached it, the amount is already at least 100.
