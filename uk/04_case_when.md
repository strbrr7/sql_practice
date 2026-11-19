# 04. CASE WHEN і approval rate

`CASE WHEN` дозволяє рахувати не всі рядки, а тільки ті, що відповідають умові:

| Що порахувати | Як |
|---|---|
| скільки рядків за умовою | `SUM(CASE WHEN … THEN 1 ELSE 0 END)` |
| яка сума за умовою | `SUM(CASE WHEN … THEN amount ELSE 0 END)` |
| яка частка (approval rate) | `AVG(CASE WHEN … THEN 1.0 ELSE 0 END)` |

`1.0`, а не `1`: інакше в деяких базах ділення буде цілочисельним.

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

**1. Для кожного способу оплати: кількість платежів, кількість відхилених і approval rate.**
```sql
SELECT method,
       COUNT(*) AS total,
       SUM(CASE WHEN status = 'declined' THEN 1 ELSE 0 END) AS declined_count,
       AVG(CASE WHEN status = 'success' THEN 1.0 ELSE 0 END) AS approval_rate
FROM payments
GROUP BY method;
```
Результат: card 4, 1, 0.75; apple_pay 2, 1, 0.50; google_pay 1, 0, 1.00.

**2. Для кожного дня: кількість платежів, сума успішних і approval rate.**
```sql
SELECT day,
       COUNT(*) AS total,
       SUM(CASE WHEN status = 'success' THEN amount ELSE 0 END) AS success_amount,
       AVG(CASE WHEN status = 'success' THEN 1.0 ELSE 0 END) AS approval_rate
FROM payments
GROUP BY day
ORDER BY day;
```
Результат: 2026-11-10 — 2, 200, 0.50; 2026-11-11 — 2, 300, 0.50; 2026-11-12 — 3, 800, 1.00.

**3. Клієнти з approval rate менше 1: кількість успішних і відхилених платежів.**
```sql
SELECT client_id,
       SUM(CASE WHEN status = 'success'  THEN 1 ELSE 0 END) AS success_count,
       SUM(CASE WHEN status = 'declined' THEN 1 ELSE 0 END) AS declined_count,
       AVG(CASE WHEN status = 'success'  THEN 1.0 ELSE 0 END) AS approval_rate
FROM payments
GROUP BY client_id
HAVING AVG(CASE WHEN status = 'success' THEN 1.0 ELSE 0 END) < 1;
```
Результат: client 1 — 2, 1, 0.67; client 3 — 1, 1, 0.50.

## Категоризація

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

**4. Категорія суми: small (менше 100), medium (100–499), large (500 і більше).**
```sql
SELECT id, amount,
       CASE
           WHEN amount < 100 THEN 'small'
           WHEN amount < 500 THEN 'medium'
           ELSE 'large'
       END AS category
FROM transactions;
```
Результат: small — 2, 4, 6; medium — 1, 3, 7, 9; large — 5, 8.
> Умови CASE перевіряються по черзі, тому в другій достатньо `< 500`: якщо дійшли до неї, сума вже точно не менша за 100.
